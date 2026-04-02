# Team Discovery, Pane/Worktree Backends, and Operator UI

## Scope

This document covers the operator-facing swarm backend layer: how the runtime decides between pane-based and in-process teammates, how pane backends behave, how teams are surfaced in the UI, and how the team file is used to drive teammate discovery, visibility, and cleanup metadata.

Primary files read for this slice:

- `utils/teamDiscovery.ts`
- `utils/swarm/backends/types.ts`
- `utils/swarm/backends/registry.ts`
- `utils/swarm/backends/detection.ts`
- `utils/swarm/backends/TmuxBackend.ts`
- `utils/swarm/backends/ITermBackend.ts`
- `utils/swarm/backends/InProcessBackend.ts`
- `utils/swarm/backends/PaneBackendExecutor.ts`
- `utils/swarm/backends/teammateModeSnapshot.ts`
- `utils/swarm/backends/it2Setup.ts`
- `utils/swarm/It2SetupPrompt.tsx`
- `utils/swarm/teammateLayoutManager.ts`
- `utils/swarm/spawnUtils.ts`
- `utils/swarm/teamHelpers.ts`
- `tools/shared/spawnMultiAgent.ts`
- `components/PromptInput/PromptInput.tsx`
- `components/PromptInput/PromptInputFooterLeftSide.tsx`
- `components/teams/TeamStatus.tsx`
- `components/teams/TeamsDialog.tsx`

## The Backend Choice Is A Session-Level Runtime Decision

The code does not treat teammate execution mode as a simple one-off spawn flag.

`registry.ts` and `teammateModeSnapshot.ts` show a session-scoped decision model:

- teammate mode is captured at startup from CLI override or config
- the captured snapshot is reused for the lifetime of the session
- runtime config changes are intentionally ignored unless a dedicated override-clearing path updates the snapshot

The effective modes are:

- `auto`
- `tmux`
- `in-process`

But the runtime also distinguishes:

- the configured mode
- the resolved mode after environment detection

So the code treats teammate backend selection as part of session initialization, not just ad hoc spawn logic.

## `auto` Mode Resolves Differently Depending On Environment

`isInProcessEnabled()` in `registry.ts` is the actual resolution point.

Verified behavior:

- non-interactive/headless sessions force in-process mode
- explicit `in-process` mode always chooses in-process
- explicit `tmux` mode always chooses pane-based mode
- `auto` chooses pane mode when inside tmux or iTerm2
- `auto` chooses in-process when not in a pane-capable environment

There is also a persistent runtime override:

- if pane-based spawn fails in `auto` mode and the system falls back to in-process, `markInProcessFallback()` is set
- after that, `isInProcessEnabled()` keeps returning true for the rest of the session

That means the UI and later spawns are intentionally made to reflect the actual fallback path, not the original user preference.

## Environment Detection Is Conservative By Design

`detection.ts` deliberately avoids over-detecting pane capability.

Verified choices:

- tmux detection uses the original `TMUX` value captured at module load
- it does not use `tmux display-message` as a fallback, because that would detect any tmux server on the system rather than whether this process is inside tmux
- the leader's original tmux pane ID is captured from `TMUX_PANE` at startup
- iTerm2 detection uses `TERM_PROGRAM`, `ITERM_SESSION_ID`, and terminal metadata
- tmux availability and it2 CLI availability are probed separately

This matters because later pane operations depend on knowing:

- whether the leader is actually inside tmux
- whether an external swarm socket is needed
- whether the leader's original pane/window should be targeted instead of the user's currently focused pane

## Backend Detection Has A Strict Priority Order

`detectAndGetBackend()` implements a specific precedence model:

1. if already inside tmux, use tmux
2. otherwise, if in iTerm2 and it2 works, use iTerm2 native panes
3. otherwise, if in iTerm2 and tmux is available, use tmux fallback and optionally recommend/setup iTerm2
4. otherwise, if tmux is available, use external tmux session mode
5. otherwise, fail with platform-specific tmux installation instructions

That order is important because the runtime does not simply "prefer iTerm2 on macOS". It prefers the environment the user is already in, with tmux winning if Claude itself started inside tmux.

## Backend Registration And Detection Are Intentionally Separate

`registry.ts` distinguishes:

- dynamic backend class registration
- environment detection
- executor creation

Important verified split:

- `ensureBackendsRegistered()` imports backend classes without doing expensive environment detection
- `detectAndGetBackend()` does the actual environment probe and caches the result
- `getBackendByType(...)` can instantiate a backend class directly once registration has happened

That separation is used by UI and shutdown flows that only need to kill or show panes and should not redo detection in potentially different conditions.

## Pane Backends And Teammate Executors Are Different Abstractions

`backends/types.ts` defines two related but distinct interfaces:

1. `PaneBackend` for pane operations
2. `TeammateExecutor` for high-level teammate lifecycle operations

`PaneBackend` owns low-level operations such as:

- create pane
- send command
- set border color
- set title
- enable border status
- rebalance panes
- kill pane
- hide pane
- show pane

`TeammateExecutor` owns higher-level operations such as:

- spawn teammate
- send teammate message
- graceful terminate
- force kill
- active-state check

`PaneBackendExecutor` is the adapter that wraps a detected `PaneBackend` and exposes the higher-level `TeammateExecutor` contract.

So the system separates visual terminal layout from teammate lifecycle semantics.

## Tmux Backend Supports Two Distinct Layout Modes

`TmuxBackend.ts` shows that tmux behaves differently depending on whether Claude started inside tmux.

Inside tmux:

- teammates are added to the leader's current window
- first teammate splits horizontally from the leader
- later teammates split off teammate panes
- layout is rebalanced using `main-vertical`
- the leader pane is resized to 30%

Outside tmux:

- teammates are created in an external `claude-swarm` tmux session
- a dedicated `swarm-view` window is created if needed
- the first pane in that external window is reused for the first teammate
- later panes are split and rebalanced in tiled layout

So "tmux mode" itself has two runtime submodes:

- shared-window mode with leader present
- external-session swarm-view mode without the leader pane

## Tmux Pane Operations Are Socket-Aware

The tmux backend explicitly distinguishes:

- commands run against the user's original tmux session
- commands run against the external swarm socket

This is why several methods take `useExternalSession`.

Examples:

- sending commands to panes
- enabling border status
- killing panes
- hide/show pane operations

Without this split, external swarm teammates would target the wrong tmux server.

## Tmux Hide/Show Is Real Backend Functionality

The tmux backend supports pane visibility management.

Verified implementation:

- hide uses `break-pane` into a detached hidden session
- show uses `join-pane` back into the target window/pane
- after showing, the backend reapplies `main-vertical` layout and leader sizing

This is backed by persistent team-file state via `hiddenPaneIds` in `teamHelpers.ts`, and surfaced in discovery via `teamDiscovery.ts`.

So hide/show is not just a UI tag. It is backed by real pane movement and stored state.

## iTerm2 Backend Is Native But More Constrained

`ITermBackend.ts` implements a native iTerm2 split-pane path using the `it2` CLI.

Verified characteristics:

- only available when running inside iTerm2 and the `it2` CLI can actually reach the Python API
- first teammate splits vertically from the leader session
- later teammates split from the last teammate session
- pane creation uses targeted session IDs rather than active-window assumptions
- dead targeted session IDs are pruned if a split fails and `it2 session list` confirms the target is gone

Important limitations in code:

- `supportsHideShow = false`
- pane border color and title setters are effectively skipped for performance
- pane rebalancing is a no-op because iTerm2 handles layout automatically

So iTerm2 offers native panes, but with a narrower control surface than tmux.

## iTerm2 Setup Is Its Own Guided Workflow

The spawn path can enter an iTerm2 setup flow before proceeding.

Verified behavior from `spawnMultiAgent.ts`, `It2SetupPrompt.tsx`, and `it2Setup.ts`:

- if running in iTerm2 without usable `it2`, and tmux is available, the runtime can prompt instead of failing immediately
- the prompt can install `it2` via `uv`, `pipx`, or `pip`
- installation is intentionally run from the user's home directory, not the repo, to avoid malicious project-local Python tool config
- after install, the prompt walks the user through enabling the iTerm2 Python API
- the user can instead choose to prefer tmux over iTerm2 in future

That preference is persisted in global config and changes backend detection behavior later.

## In-Process Backend Is A Peer Executor, Not A Pane Backend

`InProcessBackend.ts` does not implement `PaneBackend`; it implements `TeammateExecutor`.

Verified behavior:

- always available
- requires a `ToolUseContext` before spawn
- spawns teammates through `spawnInProcessTeammate(...)`
- starts their execution loop directly with `startInProcessTeammate(...)`
- uses mailbox delivery for messages for consistency with pane-based teammates
- uses graceful shutdown requests for termination
- uses `killInProcessTeammate(...)` for force-kill

So in-process execution is intentionally modeled as another teammate backend at the lifecycle level, but not at the visual-pane abstraction level.

## `PaneBackendExecutor` Is The High-Level Pane Adapter

`PaneBackendExecutor.ts` is the pane-based executor bridge.

Verified responsibilities:

- creates a pane through the selected pane backend
- builds the teammate CLI command with team identity flags and inherited flags/env
- sends that command into the pane
- sends the initial prompt through mailbox
- tracks spawned teammate agent ID to pane ID mappings
- registers cleanup to kill pane-backed teammates on leader exit

This is important because the executor layer is what turns a pane backend into something equivalent to `InProcessBackend` from the caller's perspective.

## Spawn Command Construction Is Deliberately Inherited

`spawnUtils.ts` and `spawnMultiAgent.ts` show that spawned teammates inherit key session settings.

Verified propagated state includes:

- permission mode, with plan mode overriding bypass inheritance
- model override
- settings path
- inline plugin directories
- teammate mode snapshot
- chrome/no-chrome CLI choices
- important environment variables for provider selection, custom endpoints, config dir, remote mode, proxy settings, and certs

This is a source-of-truth correction from code:

- teammate spawning is not "run the same binary with a team name"
- it is a carefully constructed child-session environment

## Team Discovery For The UI Is Team-File Based

`teamDiscovery.ts` reads teammate state from the team file, not from a live process registry.

Verified outputs include:

- teammate name and agent ID
- agent type, model, prompt
- status derived from `isActive`
- color
- pane ID
- cwd and optional `worktreePath`
- hidden-pane status via `hiddenPaneIds`
- backend type
- permission mode

One subtle point:

- status is derived from durable config state, not from querying the actual pane process directly

That means team discovery is intentionally low-cost and file-backed.

## The Footer Team UI Uses Live `teamContext`, Not Filesystem Discovery

`PromptInput.tsx`, `PromptInputFooterLeftSide.tsx`, and `TeamStatus.tsx` show a separate lightweight path for the active session footer.

Verified behavior:

- the footer derives team presence from `teamContext`
- it intentionally avoids filesystem I/O
- it is hidden in in-process mode, where teammate navigation uses different controls
- the footer count excludes the leader
- `PromptInput.tsx` builds a cached one-team `TeamSummary` from `teamContext` only

So the runtime has two operator-facing representations:

- a cheap live-session footer from `teamContext`
- a richer dialog/discovery view that consults the team file

## `TeamsDialog` Mixes Live Session State With Team-File Discovery

`TeamsDialog.tsx` uses:

- initial team identity from `PromptInput` and `teamContext`
- detailed teammate rows from `getTeammateStatuses(...)`, which reads the team file
- task lists from the file-backed team task system
- `setAppState(...)` updates to keep the live session in sync after kill flows

Verified controls include:

- viewing teammate output by focusing the tmux pane or iTerm2 session
- killing a teammate immediately
- sending graceful shutdown requests
- pruning idle teammates
- cycling one teammate or all teammates through permission modes

The detail view also prefers `worktreePath` over `cwd` when presenting the teammate working path.

## Operator Kill And View Paths Respect Backend Type

The UI kill and view helpers use the teammate's recorded backend type rather than assuming tmux.

Verified behavior:

- viewing output focuses an iTerm2 session when backend is `iterm2`
- otherwise it selects the tmux pane, using the external swarm socket when not inside tmux
- killing a teammate first tries the recorded backend's `killPane(...)`
- after pane kill, the UI removes the member from the team file, unassigns tasks, and updates `teamContext`

This is another important correction:

- pane identity alone is not enough
- the backend type is part of the durable routing key for operator actions

## Worktree Awareness In This Layer Is Mostly Metadata And Cleanup

Within the swarm discovery/backend slice, worktrees appear in two places:

1. as per-member metadata in the team file and UI
2. in team cleanup logic that removes recorded teammate worktrees

Verified behavior in this layer:

- `TeammateStatus` surfaces `worktreePath`
- `TeamsDialog` displays worktree path preferentially over `cwd`
- `cleanupTeamDirectories(...)` removes any member worktrees recorded in the team file before deleting the team directory
- worktree removal first tries `git worktree remove --force` and falls back to recursive directory removal

The actual creation of swarm teammate worktrees was not verified in this slice from a concrete spawn path in the files read here, so the worktree portion of this document is intentionally limited to metadata and cleanup behavior.

## Source-Of-Truth Corrections From Code

The code makes several important corrections to a simplified mental model:

- teammate backend selection is a session-scoped, cached runtime decision, not only a spawn-time switch
- `auto` mode can resolve to panes or in-process, and can permanently flip to in-process after fallback
- tmux mode itself splits into inside-tmux leader layout versus external swarm-session layout
- iTerm2 native panes are supported, but with a narrower feature set than tmux
- the footer team UI is intentionally derived from live `teamContext`, while detailed discovery is file-backed
- backend type is durable operational metadata used for later kill/view actions
- the team file stores enough information to reconstruct hidden state, backend type, mode, and worktree/cwd display without querying live panes directly

## Caveats

Two caveats are worth making explicit:

- `components/teams/TeamsDialog.tsx` in this workspace snapshot is compiled output, and the `hideTeammate(...)` / `showTeammate(...)` helpers are stubbed there, so hide/show behavior is documented from the backend implementations, team-file helpers, and UI call sites rather than a complete source implementation in that component file
- this slice verifies worktree metadata and cleanup behavior in the swarm backend/UI layer, but not the full worktree-creation subsystem itself
