# Swarm Team Coordination and Leader-Teammate Protocols

## Scope

This document covers the swarm/team coordination layer above the generic runtime task system: team identity, team file state, mailbox protocol, leader-versus-teammate behavior, permission delegation, mode synchronization, and shutdown/idle coordination.

Primary files read for this slice:

- `utils/teammate.ts`
- `utils/teammateContext.ts`
- `utils/teammateMailbox.ts`
- `utils/swarm/teamHelpers.ts`
- `utils/swarm/permissionSync.ts`
- `utils/swarm/leaderPermissionBridge.ts`
- `utils/swarm/teammateInit.ts`
- `utils/swarm/teammatePromptAddendum.ts`
- `utils/swarm/inProcessRunner.ts`
- `utils/swarm/spawnInProcess.ts`
- `tools/shared/spawnMultiAgent.ts`
- `tools/TeamCreateTool/TeamCreateTool.ts`
- `tools/TeamDeleteTool/TeamDeleteTool.ts`
- `tools/SendMessageTool/SendMessageTool.ts`
- `hooks/useInboxPoller.ts`
- `hooks/useSwarmInitialization.ts`
- `components/PromptInput/PromptInput.tsx`
- `components/teams/TeamsDialog.tsx`
- `main.tsx`
- `cli/print.ts`

## The Coordination Model Is Layered, Not Just "Agents Send Messages"

The code keeps at least four distinct coordination layers:

1. Durable team membership and metadata in the team file under `~/.claude/teams/{team}/config.json`
2. Live session mirror state in `AppState.teamContext`
3. Mailbox-based structured protocol messages in per-agent inbox files
4. Local runtime control for in-process teammates via AsyncLocalStorage, task state, and leader-side permission bridges

Those layers overlap, but they are not interchangeable.

Examples from code:

- the team file is the durable source for members, pane IDs, backend type, permission mode, active/idle status, hidden panes, worktree path, and team-wide allowed paths
- `AppState.teamContext` is the live-session mirror used by the REPL, dialogs, and headless runtime
- the mailbox is the transport for both human-readable messages and machine-routed protocol messages
- in-process teammates can bypass some mailbox hops by using leader-registered queue/context setters in-process

So the swarm system is not a thin wrapper around `SendMessage`. It is a protocol stack.

## Team Identity Resolution Has A Defined Priority Order

`utils/teammate.ts` makes identity resolution explicit.

Priority order:

1. AsyncLocalStorage teammate context for in-process teammates
2. dynamic team context for pane/process teammates or runtime join flows
3. leader-provided `teamContext` where applicable

Verified helper responsibilities include:

- `getAgentId()`
- `getAgentName()`
- `getTeamName()`
- `getParentSessionId()`
- `getTeammateColor()`
- `isTeammate()`
- `isTeamLead(...)`
- `isPlanModeRequired()`

This matters because the code intentionally does not model the leader as a normal teammate process. `TeamCreateTool` creates a deterministic lead agent ID and stores it in the team file and `teamContext`, but intentionally does not set leader teammate env identity in the normal teammate path.

## Team Creation Establishes Both A Team File And A Team-Scoped Task Space

`TeamCreateTool` does more than create a name.

Verified responsibilities:

- enforces one active team per leader session
- generates a deterministic lead agent ID using `team-lead@teamName`
- writes the initial team file with the leader as the first member
- records the real leader session UUID as `leadSessionId`
- resets and creates the team task-list directory
- registers the leader team name so file-backed coordination tasks go to the team-scoped directory instead of the plain session ID
- seeds `AppState.teamContext`
- registers the team for session-end cleanup

This is an important source-of-truth point:

- a swarm team is also a coordination namespace for file-backed task assignment
- the leader's team name is part of task-routing correctness, not only UI metadata

## Team Membership Is Written Twice On Purpose

When teammates are spawned in `spawnMultiAgent.ts`, the code updates:

1. `AppState.teamContext.teammates`
2. the team file's `members` array

That duplication is intentional because the two stores serve different purposes.

`teamContext` carries:

- current session visibility
- active teammate count
- UI render data
- headless shutdown bookkeeping

The team file carries:

- durable member discovery
- backend type and pane ID
- teammate working directory and optional worktree path
- current permission mode
- active/idle state
- subscriptions and session IDs

The spawn path also differs by backend:

- pane/tmux teammates receive their initial prompt by mailbox after process creation
- in-process teammates do not get the initial prompt via mailbox, because they are started directly inside the same process

## The Team File Is More Than A Member List

`utils/swarm/teamHelpers.ts` shows that the team file is the durable coordination document for swarm runtime state.

Verified persisted fields include:

- team name and description
- creation time
- `leadAgentId`
- `leadSessionId`
- `hiddenPaneIds`
- `teamAllowedPaths`
- per-member identity, color, model, prompt, backend, pane ID, cwd, worktree path, session ID, subscriptions, active state, and current permission mode

The helper layer also owns:

- removing members by pane ID or agent ID
- removing teammates by leader-side identifier during shutdown
- syncing teammate mode into config
- atomic multi-member mode updates
- active/idle writes
- worktree cleanup
- orphan-team cleanup on leader shutdown

So the team file is effectively a lightweight swarm control plane persisted on disk.

## Teammates Get A Different System Prompt Contract

Teammates are not just ordinary Claude sessions with a team name attached.

Verified prompt wiring:

- `main.tsx` appends `TEAMMATE_SYSTEM_PROMPT_ADDENDUM` for teammate processes
- `inProcessRunner.ts` also appends the teammate addendum to the in-process system prompt stack

That addendum tells the model:

- plain assistant text is not visible to teammates
- `SendMessage` must be used for actual inter-agent communication
- the user primarily interacts with the team lead

This is a major behavioral boundary in the implementation:

- teammate output is not automatically routed to peers or the leader
- visible coordination must happen through the explicit messaging protocol

## The Mailbox Is A Protocol Transport, Not Just A Chat Inbox

`utils/teammateMailbox.ts` defines many structured message types, not just free-form messages.

Verified protocol message families include:

- idle notifications
- permission requests and permission responses
- sandbox permission requests and responses
- plan approval requests and responses
- shutdown request, shutdown approved, shutdown rejected
- mode set request

The code explicitly marks these as structured protocol messages so they can be intercepted by `useInboxPoller` and routed to the right subsystem instead of being passed through as raw LLM-visible content.

This is one of the most important corrections from the code:

- the mailbox is not only a human messaging surface
- it is also the transport for swarm control messages

## `SendMessage` Is The Public Coordination API

`tools/SendMessageTool/SendMessageTool.ts` is the model-facing entry point for most swarm communication.

Verified supported behaviors:

- plain directed teammate messages
- broadcast messages to all teammates
- shutdown request messages
- shutdown approval or rejection responses
- plan approval responses from the leader
- direct delivery to live background agents by registered name or raw agent ID
- optional cross-session `uds:` and `bridge:` addressing for plain text when that feature path is enabled

Some hard constraints in code:

- structured protocol messages cannot be broadcast
- cross-session sends only allow plain text, not structured protocol payloads
- shutdown responses must be addressed to `team-lead`
- plan approval is leader-only

So `SendMessage` is not just "append a string to a mailbox file". It is a validated public API over several routing backends.

## The Leader And Teammates Do Not Read Their Mailboxes The Same Way

The runtime has separate leader-side and teammate-side mailbox behavior.

Leader-side behavior in `useInboxPoller.ts` and `print.ts` includes:

- collecting worker permission requests into the leader permission queue
- collecting sandbox permission requests into the worker sandbox queue
- auto-approving plan approval requests in the current interactive logic
- processing shutdown approvals by removing teammates from the team file and `teamContext`
- killing panes for pane-backed teammates when shutdown approval includes pane/backend info
- unassigning tasks owned by the shutting-down teammate

Teammate-side behavior includes:

- applying mode-set requests from the leader
- applying team permission updates
- consuming permission responses and sandbox permission responses
- receiving shutdown requests as model-visible context

Headless `print.ts` mirrors a subset of interactive leader behavior:

- polling unread team-lead inbox messages while teammates are active
- processing shutdown approvals
- injecting a shutdown prompt when input closes but teammates still exist

So the headless and interactive runtimes share the same protocol ideas even though they do not share the exact same hook stack.

## In-Process Teammates Have A Distinct Control Path

`inProcessRunner.ts` shows that in-process teammates are not treated as ordinary background agents.

Verified behavior:

- each teammate runs under `runWithTeammateContext(...)` using AsyncLocalStorage isolation
- teammate history is accumulated in a local `allMessages` buffer and can auto-compact independently
- the runner does not automatically forward the teammate's raw output back to the leader
- after each prompt, the teammate becomes idle instead of terminating
- the runner then waits for the next prompt, a claimed team task, or a shutdown request

The wait loop prioritizes work in this order:

1. in-memory pending user messages injected directly into the teammate task
2. mailbox shutdown requests
3. mailbox messages from `team-lead`
4. other unread mailbox messages
5. unclaimed tasks from the team task list

That priority order is explicit in the code and gives the leader coordination precedence over peer chatter.

## Idle Notifications Are A First-Class Protocol Signal

Teammates explicitly notify the leader when they become idle.

Verified implementations:

- pane/process teammates register a `Stop` hook in `teammateInit.ts`
- in-process teammates send idle notifications on transition to idle in `inProcessRunner.ts`

Idle notifications include:

- the teammate name
- an idle reason such as `available`, `interrupted`, or `failed`
- an optional summary derived from the last peer-directed `SendMessage`

That summary extraction is deliberate. The code wants the leader to see a concise "what I just told another peer" summary rather than only raw task completion.

So idle is part of the collaboration protocol, not just a UI status.

## Permission Delegation Has Two Swarm Paths

The swarm permission system has a split runtime design.

For in-process teammates:

- `inProcessRunner.ts` first tries the leader-side `ToolUseConfirm` queue through `leaderPermissionBridge.ts`
- this gives the leader the normal rich permission UI with worker badges
- if permission updates are applied, the runner can also push updated permission context back through the bridge

Fallback behavior:

- if the bridge is unavailable, the in-process runner creates a mailbox permission request and polls its own mailbox for the response

For pane/process teammates:

- `permissionSync.ts` plus `useInboxPoller.ts` handle mailbox-based permission request and response routing

This is another important correction:

- swarm permission delegation is not only mailbox-based
- in-process teammates can share the leader's real permission UI path directly

## Permission Mode Synchronization Is Also A Team Protocol

The swarm system synchronizes permission mode at multiple levels.

Verified paths:

- a teammate changing its own mode in `PromptInput.tsx` writes local state and then calls `syncTeammateMode(...)` so the leader sees the new mode in the team file
- the leader-side `TeamsDialog.tsx` can cycle one teammate or all teammates, writing mode changes to the team file immediately and also sending `mode_set_request` mailbox messages
- teammate-side `useInboxPoller.ts` applies mode-set requests only if they came from `team-lead`
- in-process teammate task state also carries its own `permissionMode`, and the runner re-reads that state on each iteration

This means mode is not a purely local UI concern. It is part of the swarm coordination state.

## Team-Wide Allowed Paths Are Pushed Into Teammates At Initialization

`teammateInit.ts` applies `teamAllowedPaths` from the team file into the teammate's session permission context when the teammate starts or resumes.

Those entries become session allow rules such as:

- tool name
- directory path converted into a recursive allow rule
- origin metadata about who added the rule

So the team file can carry leader-approved shared edit boundaries that become real permission rules inside each teammate session.

## Shutdown Is A Negotiated Protocol, Not A Blind Kill

The shutdown path is intentionally multi-step.

Verified sequence:

1. the leader sends a structured `shutdown_request`
2. the teammate receives it as a model-visible message
3. the teammate uses `SendMessage` with a shutdown response to approve or reject
4. on approval, the leader removes the teammate from the team file and active context and unassigns its tasks
5. pane-backed teammates may also have their pane/session killed
6. in-process teammates abort their own controller directly when they approve shutdown

Related enforcement:

- `TeamDeleteTool` refuses to clean up the team while non-lead members are still active
- session shutdown cleanup in `teamHelpers.ts` still exists as a best-effort safety net for orphaned teams and panes

So graceful shutdown is the intended protocol, while hard cleanup is the fallback.

## Team Deletion Is Intentionally Separate From Teammate Shutdown

The code draws a strong boundary between:

- shutting teammates down
- deleting the team itself

`TeamDeleteTool` only succeeds after active non-lead members are gone. Then it:

- cleans up the team directory
- cleans up task directories
- removes teammate worktrees
- clears team color assignments
- clears the leader's team-name routing
- clears `teamContext` and inbox state

That design avoids collapsing "member lifecycle" and "team lifecycle" into one operation.

## The Interactive Operator UI Writes Directly To The Same Protocol

`TeamsDialog.tsx` is not a separate management plane. It writes into the same underlying swarm mechanisms.

Verified controls:

- cycle one teammate or all teammates through permission modes
- send shutdown requests
- hide or show panes where the backend supports it
- remove members and unassign tasks during kill flows

The UI updates the team file immediately for responsiveness, then sends protocol messages so the teammate runtime converges to the same state.

That "write durable state, then message the runtime" pattern appears repeatedly across the subsystem.

## Source-Of-Truth Corrections From Code

The code makes several important corrections to a simplified mental model:

- swarm coordination is not only `SendMessage`; it is a layered control protocol spanning team file state, mailbox messages, and in-process bridges
- the team file is a real control document, not just a discovery file
- teammate assistant text is intentionally not shared with peers unless a tool sends it
- leader-side permission handling for in-process teammates uses the normal `ToolUseConfirm` UI when available
- idle status is an explicit protocol event, not only a display state
- shutdown is negotiated and reflected back into both task ownership and team membership
- permission mode and team-wide allow rules are coordinated across sessions, not just stored locally

## Caveats

This slice is fully source-backed from files present in the workspace snapshot. The one limitation is scope: it focuses on the swarm coordination protocol itself, not the pane backend implementation details, worktree creation internals, or broader team discovery UI.
