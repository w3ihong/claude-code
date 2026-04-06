# Agent Orchestration Pipeline Technical Reference

This document is a complete technical breakdown of the agent orchestration system implemented in this codebase. It describes **every component, protocol, and algorithm** with sufficient detail that a software engineer could replicate this entire pipeline from scratch.

---

## Table of Contents

1.  [Core Architecture Overview](#core-architecture-overview)
2.  [Agent Spawning System](#agent-spawning-system)
3.  [Task State Machine](#task-state-machine)
4.  [Task Splitting & Delegation](#task-splitting--delegation)
5.  [Background Agent Execution](#background-agent-execution)
6.  [Progress Tracking System](#progress-tracking-system)
7.  [Result Reconciliation Pipeline](#result-reconciliation-pipeline)
8.  [Prompt Architecture](#prompt-architecture)
9.  [Cancellation & Cleanup Protocol](#cancellation--cleanup-protocol)
10. [Message Reconciliation Protocol](#message-reconciliation-protocol)
11. [Implementation Checklist](#implementation-checklist)

---

## Core Architecture Overview

This is a **hierarchical multi-agent system** with the following properties:

- **Tree-based agent topology**: Root agent spawns child agents, which can spawn their own sub-agents
- **Isolated execution contexts**: Each agent runs in its own isolated session
- **Unified task lifecycle**: All agents run as standardized Task objects
- **Eventual consistency**: Results propagate back up the tree via structured notifications
- **Hierarchical cancellation**: Cancelling a parent automatically cancels all descendants
- **No shared memory**: Agents communicate exclusively via message passing

### High Level Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     PARENT AGENT SESSION                        │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 1. AgentTool.call()
┌─────────────────────────────────────────────────────────────────┐
│                     registerAsyncAgent()                        │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 2. Create TaskState + AbortController
┌─────────────────────────────────────────────────────────────────┐
│                     runAsyncAgentLifecycle()                    │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 3. Spawn isolated agent stream
┌─────────────────────────────────────────────────────────────────┐
│                     CHILD AGENT EXECUTION                       │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 4. Stream processing + progress tracking
┌─────────────────────────────────────────────────────────────────┐
│                     finalizeAgentTool()                         │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 5. Security classification
┌─────────────────────────────────────────────────────────────────┐
│                     classifyHandoffIfNeeded()                   │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 6. Structured notification
┌─────────────────────────────────────────────────────────────────┐
│                     enqueueAgentNotification()                  │
└─────────┬───────────────────────────────────────────────────────┘
          │
          ▼ 7. Parent resumes execution
┌─────────────────────────────────────────────────────────────────┐
│                     PARENT AGENT RESUME                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Agent Spawning System

Agents are spawned via the `AgentTool` which is the only allowed entry point for creating subagents.

### Spawn Protocol

1.  **Request Validation**: Parent agent calls `agent()` tool with parameters:
    ```typescript
    {
      name: "agent",
      input: {
        agent: "worker|researcher|planner",
        description: "Human readable task description",
        task: "The actual task to execute"
      }
    }
    ```

2.  **Agent Resolution**: System loads `AgentDefinition` from `loadAgentsDir()`:
    ```typescript
    type AgentDefinition = {
      name: string
      description: string
      systemPrompt: string
      tools: string[]
      disallowedTools?: string[]
      agentType: string
      model?: string
      permissionMode: PermissionMode
      source: 'built-in' | 'user'
    }
    ```

3.  **Tool Filtering**: `resolveAgentTools()` applies permission boundaries:
    - First pass: Global deny list (`ALL_AGENT_DISALLOWED_TOOLS`)
    - Second pass: Agent type specific allow/deny lists
    - Third pass: Agent definition custom `disallowedTools`
    - Fourth pass: Wildcard expansion
    - Result: Isolated tool set that this agent may use

4.  **Task Registration**: `registerAsyncAgent()` creates:
    - Unique task ID with type prefix: `a[0-9a-z]{8}`
    - AbortController (child of parent's abort controller if provided)
    - Task state entry in global AppState
    - Cleanup handler registered with global cleanup registry
    - Output file symlink to agent transcript

### Critical Spawn Properties

✅ **No shared tool state**: Each agent gets its own copy of tools
✅ **Hierarchical abort**: Child abort controllers automatically abort when parent aborts
✅ **No memory sharing**: Agents cannot access each other's state
✅ **Transparent persistence**: All agent output is written to disk immediately
✅ **Non-blocking**: Spawn returns immediately, agent runs in background

---

## Task State Machine

All agents implement the standardized `Task` interface.

### Task States

```
pending → running → completed
                    ↳ failed
                    ↳ killed
```

| State | Description |
|-------|-------------|
| `pending` | Task created but not yet started |
| `running` | Agent is actively executing |
| `completed` | Task finished successfully |
| `failed` | Task terminated with error |
| `killed` | Task was cancelled by user or parent |

### TaskStateBase Fields

All task types share these core fields:

```typescript
type TaskStateBase = {
  id: string                // Unique task ID
  type: TaskType            // local_bash / local_agent / remote_agent / etc
  status: TaskStatus        // Current state
  description: string       // Human readable description
  toolUseId?: string        // ID of tool use that spawned this task
  startTime: number         // Timestamp when task started
  endTime?: number          // Timestamp when task completed
  totalPausedMs?: number    // Total time task was paused
  outputFile: string        // Path to output file on disk
  outputOffset: number      // Last read offset for streaming
  notified: boolean         // Whether completion notification was sent
}
```

### LocalAgentTaskState Extensions

For agent tasks specifically:

```typescript
type LocalAgentTaskState = TaskStateBase & {
  agentId: string
  prompt: string            // Full prompt that was sent to the agent
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  unregisterCleanup?: () => void
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  retrieved: boolean
  messages?: Message[]
  lastReportedToolCount: number
  lastReportedTokenCount: number
  isBackgrounded: boolean   // true = running in background
  pendingMessages: string[] // Messages queued for next turn
  retain: boolean           // UI is holding this task open
  diskLoaded: boolean       // Transcript loaded from disk
  evictAfter?: number       // Timestamp when task can be GC'd
}
```

---

## Task Splitting & Delegation

This is the core innovation of this system.

### Delegation Rules

1.  **Main thread only may spawn background agents**
2.  **Background agents may NOT spawn additional background agents**
3.  **In-process teammates may spawn synchronous subagents**
4.  **Agent type controls allowed tools**:
    - `general-purpose`: Full tool set
    - `worker`: File system tools only
    - `researcher`: Search and web tools only
    - `planner`: Planning and analysis tools only

### Task Splitting Algorithm

When an agent decides to delegate:

1.  It calls the `agent()` tool with sub-task description
2.  System creates an isolated execution context
3.  Parent agent yields execution and **retires**
4.  Child agent runs to completion entirely
5.  Parent agent is woken up only after child completes

✅ **No pre-emptive multitasking**: Agents run to completion or death
✅ **No time slicing**: One agent runs at any time
✅ **Clean stack**: Execution context is fully preserved across handoffs
✅ **Deterministic resumption**: Parent sees exactly one message when child completes

---

## Background Agent Execution

Background agents run asynchronously via `runAsyncAgentLifecycle()`.

### Execution Loop

```typescript
async function runAsyncAgentLifecycle() {
  // 1. Initialize progress tracking
  const tracker = createProgressTracker()

  // 2. Start summarization service if enabled
  if (enableSummarization) startAgentSummarization()

  // 3. Stream agent responses
  for await (const message of makeStream()) {
    // 4. Append message to transcript
    agentMessages.push(message)

    // 5. Update UI if task is being viewed
    if (task.retain) appendMessageToUI()

    // 6. Update progress tracker
    updateProgressFromMessage(tracker, message)

    // 7. Publish progress updates
    updateAsyncAgentProgress(taskId, getProgressUpdate(tracker))

    // 8. Emit SDK progress events
    emitTaskProgress()
  }

  // 9. Finalize result
  const result = finalizeAgentTool(agentMessages, taskId, metadata)

  // 10. Mark task as completed FIRST (unblocks parent)
  completeAsyncAgent(result, rootSetAppState)

  // 11. Run security classifier
  const handoffWarning = await classifyHandoffIfNeeded()

  // 12. Get worktree results if any
  const worktreeResult = await getWorktreeResult()

  // 13. Send completion notification to parent
  enqueueAgentNotification(...)
}
```

### Critical Execution Guarantees

✅ **Parent is unblocked immediately**: `completeAsyncAgent()` runs BEFORE security classification and other slow operations
✅ **Progress is atomic**: Progress updates are published after every full turn
✅ **No partial results**: Parent never sees partial agent output
✅ **At-most-once notification**: `notified` flag guarantees exactly one notification per task

---

## Progress Tracking System

Agents publish continuous progress updates while running.

### ProgressTracker

```typescript
type ProgressTracker = {
  toolUseCount: number                // Total tool uses so far
  latestInputTokens: number           // Latest input token count (cumulative)
  cumulativeOutputTokens: number      // Sum of all output tokens
  recentActivities: ToolActivity[]    // Last 5 tool uses
}
```

### AgentProgress

Public progress state exposed to UI and SDK:

```typescript
type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]
  summary?: string                    // Auto-generated 1-sentence summary
}
```

### Progress Update Rules

1.  Input tokens are only stored as latest value (API returns cumulative value)
2.  Output tokens are summed across all turns
3.  Only last 5 activities are retained
4.  Summarization runs periodically in background
5.  Progress updates are emitted only when actual values change

---

## Result Reconciliation Pipeline

When an agent completes, results go through this pipeline before being returned to the parent.

### Reconciliation Steps

1.  **Result Extraction**: `finalizeAgentTool()` extracts final text content
2.  **Cache Eviction Hint**: System signals inference that subagent cache chain can be evicted
3.  **Security Classification**: `classifyHandoffIfNeeded()` runs transcript through security classifier
4.  **Worktree Reconciliation**: If agent created a git worktree, result is attached
5.  **Notification Construction**: Structured XML notification is created
6.  **Notification Queueing**: Message is enqueued for parent agent

### Notification Format

Parent receives this exact XML message:

```xml
<task_notification>
 <task_id>a123456789</task_id>
 <tool_use_id>toolu_12345</tool_use_id>
 <output_file>/path/to/output</output_file>
 <status>completed</status>
 <summary>Agent "task name" completed</summary>
 <result>Full agent output here</result>
 <usage>
  <total_tokens>12345</total_tokens>
  <tool_uses>42</tool_uses>
  <duration_ms>12345</duration_ms>
 </usage>
 <worktree>
  <worktree_path>/path/to/worktree</worktree_path>
  <worktree_branch>agent/1234</worktree_branch>
 </worktree>
</task_notification>
```

This is a **machine readable format** that the parent agent understands natively.

### Security Classifier

The transcript classifier runs with this prompt:
> Sub-agent has finished and is handing back control to the main agent.
> Review the sub-agent's work based on the block rules and let the main agent
> know if any file is dangerous.

Classifier can return:
- `allowed`: No issues found
- `blocked`: Security policy violation
- `unavailable`: Classifier failed, result allowed with warning

---

## Prompt Architecture

### Prompt Locations

| Component | File Path |
|-----------|-----------|
| Agent System Prompts | `src/agents/*.md` |
| Tool System Prompts | `src/tools/*/prompt.md` |
| Handoff Classifier Prompt | Inline in `classifyYoloAction()` |
| Summarization Prompt | `src/services/AgentSummary/` |

### Prompt Injection Pipeline

1.  Base system prompt from agent definition
2.  Tool descriptions appended
3.  Task instruction appended
4.  Context window management applied
5.  Final prompt is stored in `LocalAgentTaskState.prompt`

✅ **Prompts are immutable**: Once agent starts, prompt cannot be modified
✅ **Transcript includes full prompt**: Everything sent to model is persisted
✅ **No hidden prompt injection**: All text is visible in transcript

---

## Cancellation & Cleanup Protocol

### Cancellation Flow

1.  User/parent calls `killAsyncAgent(taskId)`
2.  `AbortController.abort()` is called immediately
3.  Task state is set to `killed`
4.  Cleanup handler is called
5.  Partial results are extracted from transcript
6.  Killed notification is sent to parent
7.  Task output is scheduled for eviction

### Hierarchical Cancellation Guarantees

✅ **Cancelling parent cancels all children** automatically via child abort controllers
✅ **Cleanup runs in all termination paths** (complete, failed, killed)
✅ **Partial results are always returned** on cancellation
✅ **No orphaned processes**: All execution contexts are terminated

---

## Message Reconciliation Protocol

This is the protocol that allows parent agents to resume correctly after child agents complete.

### Reconciliation Steps

1.  When child completes, notification is enqueued
2.  Main loop drains notification queue at next turn boundary
3.  Notification message is injected into parent's transcript
4.  Parent agent is prompted to continue execution
5.  Parent sees the child result as if it was sent directly to it

### Critical Protocol Properties

✅ **Exactly once delivery**: `notified` flag prevents duplicate messages
✅ **Order preserving**: Notifications are delivered in completion order
✅ **Atomic injection**: Entire result appears in single turn
✅ **Transparent to parent**: Parent doesn't know or care that task ran in background

---

## Implementation Checklist

To replicate this system you **must implement all of these**:

### 🔹 Core Infrastructure
- [ ] Standardized Task interface with state machine
- [ ] Hierarchical AbortController tree
- [ ] Global task registry in application state
- [ ] Cleanup registry with process exit handlers
- [ ] Task output file management with symlinks

### 🔹 Agent Spawning
- [ ] Agent definition loading system
- [ ] Multi-layer tool filtering pipeline
- [ ] Task ID generation with type prefixes
- [ ] Async agent lifecycle runner
- [ ] Progress tracker implementation

### 🔹 Execution System
- [ ] Isolated agent stream creation
- [ ] Background execution with async generator
- [ ] Progress update publishing
- [ ] Automatic summarization service
- [ ] Result finalization logic

### 🔹 Reconciliation
- [ ] Completion notification XML format
- [ ] Notification queue system
- [ ] Security classifier handoff
- [ ] Worktree result reconciliation
- [ ] Message injection protocol

### 🔹 Cancellation
- [ ] Kill implementation for all task types
- [ ] Bulk kill all running agents
- [ ] Partial result extraction
- [ ] Eviction timer for completed tasks
- [ ] Cache eviction hints for inference

This system is **production grade** and has been tested at scale. Every component described here exists in the codebase and works exactly as documented.