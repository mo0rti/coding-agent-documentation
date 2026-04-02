# Background Tasks, Subagents, and Notification Flow

## Scope

This document covers the runtime task system used for background work, subagents, remote tasks, in-process teammates, and the notification flow that feeds both the model and SDK consumers.

Primary files read for this slice:

- `Task.ts`
- `tasks.ts`
- `tasks/types.ts`
- `tasks/LocalAgentTask/LocalAgentTask.tsx`
- `tasks/LocalMainSessionTask.ts`
- `tasks/LocalShellTask/LocalShellTask.tsx`
- `tasks/RemoteAgentTask/RemoteAgentTask.tsx`
- `tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`
- `tasks/DreamTask/DreamTask.ts`
- `tasks/stopTask.ts`
- `utils/task/framework.ts`
- `utils/task/sdkProgress.ts`
- `utils/swarm/spawnInProcess.ts`
- `utils/swarm/inProcessRunner.ts`
- `utils/forkedAgent.ts`
- `utils/tasks.ts`
- `cli/print.ts`

## The Important Naming Split

The code has two different "task" systems.

1. Runtime background tasks in `AppState.tasks`
2. File-backed coordination tasks in `utils/tasks.ts`

The runtime task system is for things like:

- background bash commands
- local subagents
- remote Claude.ai sessions
- in-process teammates
- feature-gated workflow or monitor tasks
- dream/memory-consolidation work

The file-backed task list in `utils/tasks.ts` is different. It stores work items for team coordination under the Claude config directory and uses statuses like:

- `pending`
- `in_progress`
- `completed`

That system is not the same thing as runtime background-task state, which uses:

- `pending`
- `running`
- `completed`
- `failed`
- `killed`

So "task" is an overloaded term in this codebase and the two systems should not be collapsed together.

## Runtime Tasks Share A Common Base Model

`Task.ts` defines the shared runtime task contract.

Verified base fields include:

- task ID
- task type
- status
- description
- optional `toolUseId`
- start and end time
- output file path
- output offset
- `notified`

Task types currently include:

- `local_bash`
- `local_agent`
- `remote_agent`
- `in_process_teammate`
- `local_workflow`
- `monitor_mcp`
- `dream`

Task IDs are type-prefixed:

- `b` for bash
- `a` for local agent
- `r` for remote agent
- `t` for in-process teammate
- `w` for local workflow
- `m` for monitor MCP
- `d` for dream

That type prefixing is not cosmetic. It is part of how the runtime distinguishes task families in state and storage.

## The Shared Task Framework Is Mostly About Registration, Eviction, And SDK Bookends

`utils/task/framework.ts` handles the generic task lifecycle.

Verified shared responsibilities:

- `registerTask(...)` adds the task to `AppState`
- re-registration preserves some UI-held fields for resumed/replaced tasks
- new registration emits SDK `task_started`
- `updateTaskState(...)` performs targeted state updates
- `evictTerminalTask(...)` drops completed/failed/killed tasks once they are terminal and already notified
- `getRunningTasks(...)` returns tasks with runtime status `running`

One subtle point from code:

- `registerTask(...)` emits the SDK `task_started` bookend only for true new registrations
- replacement registrations intentionally skip that emission to avoid double starts during resume-like flows

So the framework is the authority for task birth and general cleanup, but not for all terminal notifications.

## The Framework Does Not Generate Most Completion Notifications

`generateTaskAttachments(...)` in the framework no longer creates normal completion attachments for finished tasks.

The code comment is explicit:

- completed tasks are notified by their concrete task implementations
- generating them centrally would race with per-type callbacks and cause duplicate delivery

That means terminal notification ownership is intentionally decentralized.

The framework provides:

- task registration
- eviction logic
- polling helpers

But the concrete task type decides:

- what completion means
- whether the user/model should see XML
- whether SDK consumers need direct bookend closure

## SDK Progress And Model-Facing Notifications Are Separate Channels

The runtime uses two different outbound paths:

1. SDK event queue events like `task_started`, `task_progress`, `task_notification`
2. XML `task_notification` messages enqueued into the command queue for the model

`utils/task/sdkProgress.ts` emits SDK `task_progress` events only.

By contrast, XML task notifications are built by task implementations and pushed with `enqueuePendingNotification({ mode: 'task-notification' })`.

This split matters because the two channels serve different consumers:

- SDK consumers need structured bookends and progress
- the model needs a next-turn notification payload it can read and act on

So the notification system is intentionally dual-channel, not duplicated accidentally.

## `print.ts` Is The Bridge Between XML Notifications And SDK `task_notification`

`cli/print.ts` drains queued commands and treats `task-notification` specially.

Verified behavior:

- parses the XML fields out of the queued notification
- emits a structured SDK `system/task_notification` event when a terminal `<status>` is present
- then still falls through to `ask()` so the model sees the same notification text

This is one of the most important architectural points in the subsystem:

- XML notifications are not only for UI
- they are intentionally fed back into the main agent loop

So when a background agent finishes, the next turn can both:

- update SDK clients with a structured event
- let the model inspect the task result and decide what to do next

## Statusless Task Notifications Are Progress Pings, Not Terminal Events

`print.ts` only emits SDK `task_notification` when the XML includes a `<status>` tag.

The code comments explain why:

- some task-related XML messages are statusless progress pings
- defaulting those to "completed" would falsely close the task in SDK consumers

This is especially relevant for shell-task stall detection and other progress-style notifications.

So "task notification" in the queue is broader than "task completed".

## The Headless Runner Holds Back Final Results While Background Work Continues

`runHeadlessStreaming(...)` in `print.ts` keeps a `heldBackResult` when background agents or workflows are still running.

Verified behavior:

- result messages can be withheld while background tasks remain active
- SDK events are drained before queued notifications so progress lands first
- the runner loops, waiting for background tasks to finish and enqueue notifications
- only after background work drains does the held result get emitted

This means background tasks are part of turn completion semantics in headless mode, not just optional side work.

## In-Process Teammates Are Explicitly Excluded From The "Wait For Background Tasks" Loop

One very specific runtime rule is documented directly in `print.ts`:

- `in_process_teammate` tasks are excluded from the "wait for background agents" loop

Why:

- teammates are long-lived
- they stay `running` for their whole lifetime
- they are cleaned up by shutdown protocol, not normal task completion

The comment is explicit that including them would loop forever.

So in-process teammates are runtime tasks, but they do not participate in headless turn-finalization the way ordinary background agents do.

## Local Agent Tasks Are The Main Background Subagent Primitive

`tasks/LocalAgentTask/LocalAgentTask.tsx` defines the main background subagent task type.

Verified characteristics:

- stored as `local_agent`
- usually backed by a per-agent transcript symlink
- can start foregrounded or backgrounded
- track progress, recent activities, summary text, pending inbound messages, and retained transcript state
- can be held in the coordinator panel via `retain`

The code also separates:

- regular local agents
- main-session background tasks, which reuse the same base shape but force `agentType: 'main-session'`

And there is an important UI split:

- `isPanelAgentTask(...)` excludes `main-session`

So not every `local_agent` is the same conceptual thing even though they share a runtime type.

## Foreground-To-Background Transition Is First-Class

Local agent tasks can start foregrounded and later be backgrounded.

Verified mechanisms:

- `registerAgentForeground(...)`
- `backgroundAgentTask(...)`
- a per-task background signal promise
- optional auto-background timeout
- `unregisterAgentForeground(...)` for foreground tasks that finish without backgrounding

That means backgrounding is not a separate spawn path only. The runtime supports turning an already-running foreground agent into a background task in place.

## Local Agent Progress Is Richer Than A Simple Percent Complete

Local agent tasks track:

- token count
- tool-use count
- recent activity list
- optional synthesized summary text

`updateProgressFromMessage(...)` derives progress from assistant messages:

- input tokens are treated as cumulative
- output tokens are accumulated
- tool-use activities are classified and stored

And `updateAgentSummary(...)` can emit SDK `task_progress` summaries when the SDK option is enabled.

So progress is event-derived and usage-derived, not based on explicit task phases.

## Local Agent Terminal Flow Is Split Between State Update And Notification Emit

For local agents:

- `completeAgentTask(...)` and `failAgentTask(...)` update runtime state
- `enqueueAgentNotification(...)` is the normal model-facing notification path

That separation is deliberate.

The code comments note that notification is sent by the agent tooling path, not by the raw state transition helper.

So "task marked completed" and "task notification emitted" are related but intentionally separate operations.

## Main Session Backgrounding Reuses The Local-Agent Task Shape, But It Is Different Semantically

`tasks/LocalMainSessionTask.ts` backgrounds the main session query itself.

Verified design choices:

- uses a separate task ID prefix starting with `s`
- writes to an isolated sidechain transcript, not the main transcript
- runs the background query inside a subagent context with `subagentName: 'main-session'`
- tracks progress by appending streamed events into the sidechain transcript and task state

This is a very important correction:

- backgrounding the main session is not the same as spawning a normal Agent-tool subagent
- it reuses local-agent task state, but it is its own runtime path with special transcript isolation rules

## Local Shell Tasks Are Background Tasks Too, But Their Notifications Are Different

`tasks/LocalShellTask/LocalShellTask.tsx` uses the same runtime task registry for background shell work.

Verified behavior:

- registers shell tasks as `local_bash`
- can be foregrounded first and backgrounded later
- relies on TaskOutput/disk output rather than transcript-side task messages
- sends terminal XML notifications with a background-command summary
- emits special statusless notifications when a command appears stalled on interactive input

The stall watchdog path is especially notable:

- it looks for no output growth plus prompt-like tail text
- emits a statusless task notification with the tail content and guidance
- intentionally omits `<status>` so SDK consumers do not treat it as completion

So shell tasks are integrated into the same framework, but they use a different signal mix than agent tasks.

## Remote Agent Tasks Are Poll-Driven And Resume-Aware

`tasks/RemoteAgentTask/RemoteAgentTask.tsx` represents remote Claude.ai session work.

Verified behavior:

- stores remote session ID, remote task subtype, log, todo list, and review/ultraplan state
- persists remote-agent metadata to session sidecar storage so `--resume` can restore them
- polls remote session events
- can extract review or ultraplan results from remote logs
- archives sessions on kill

Notifications are task-type-specific:

- standard remote tasks enqueue XML notifications
- some remote-review and ultraplan paths inject richer specialized notification text
- kill paths often pre-set `notified: true` and emit `emitTaskTerminatedSdk(...)` directly because the poll loop will no longer produce a terminal XML path

So remote tasks are not just "another local task with network I/O". They have sidecar persistence and poller-specific completion rules.

## In-Process Teammates Are Runtime Tasks, But They Behave More Like Long-Lived Workers

`spawnInProcessTeammate(...)`, `InProcessTeammateTask`, and `inProcessRunner.ts` define another task family.

Verified characteristics:

- run in the same Node process using AsyncLocalStorage isolation
- have team-aware identity like `agentName@teamName`
- can stay alive across multiple prompts
- support idle and active phases
- support shutdown requests and pending user-message injection

Their terminal flow is intentionally different from background agents:

- completion and failure pre-set `notified: true`
- no XML task notification is emitted
- SDK bookends are closed directly with `emitTaskTerminatedSdk(...)`

That is because teammates are not meant to send a normal "background result for the model to process" notification on each lifecycle end. They are persistent collaborators, not one-shot background jobs.

## In-Process Teammates Also Use A Different Permission Model

`inProcessRunner.ts` shows that teammates do not simply reuse background-agent permission flow.

Verified behavior:

- ask permissions route through the leader's ToolUseConfirm queue when available
- otherwise a mailbox protocol is used
- teammates can be idle while waiting for work
- shutdown requests are turned into teammate-message content for the model

So in-process teammate execution is closer to a multi-actor collaboration runtime than to ordinary async subagents.

## Dream Tasks Are UI-Surfaced Runtime Work, Not Model-Notified Tasks

`tasks/DreamTask/DreamTask.ts` adds the dream/memory-consolidation agent to the runtime task registry.

Verified behavior:

- it is a `dream` task
- it surfaces progress and files touched in the UI
- completion and failure set `notified: true` immediately
- there is no model-facing XML notification path

The code comment makes this explicit:

- dream is UI-only in this task system

So not every runtime task is meant to wake the main model loop.

## `stopTask(...)` Only Works For Task Types In The Shared Task Registry

`tasks/stopTask.ts` looks up the task implementation through `getTaskByType(...)` in `tasks.ts`.

From the current registry, that means stop support is wired for:

- local shell
- local agent
- remote agent
- dream
- feature-gated local workflow
- feature-gated monitor MCP

Notably, `in_process_teammate` is not part of `getAllTasks()` in `tasks.ts`.

So the generic stop-task path does not currently treat in-process teammates as normal stoppable runtime tasks through that registry. They are stopped through their own swarm-specific path instead.

## Source-Of-Truth Corrections From Code

The code makes several corrections to a simplified mental model:

- background tasks are not one subsystem; they are a shared registry with type-specific completion and notification rules
- SDK task events and model-facing task notifications are related but distinct channels
- not every runtime task should notify the model when it ends
- backgrounding the main session is not equivalent to spawning a normal subagent
- in-process teammates are runtime tasks, but they are long-lived collaborators, not ordinary background jobs
- the file-backed team task list in `utils/tasks.ts` is a different system from runtime `AppState.tasks`

## Caveats

Two scope caveats are worth stating explicitly:

- `Task.ts` and `tasks.ts` show feature-gated runtime types for `local_workflow` and `monitor_mcp`, but their concrete implementation files were not present in the task file list available in this workspace snapshot, so they are documented here only from the shared registry and surrounding comments
- this document focuses on runtime task lifecycle and notification flow, not the full higher-level swarm/team protocol that decides what teammates should do
