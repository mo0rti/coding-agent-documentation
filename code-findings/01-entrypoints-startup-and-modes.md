# Entrypoints, Startup, And Runtime Modes

## Scope

This document covers the top-level execution flow from process startup to either REPL launch or headless execution.

Primary source files:

- `entrypoints/cli.tsx`
- `main.tsx`
- `entrypoints/init.ts`
- `setup.ts`
- `interactiveHelpers.tsx`
- `replLauncher.tsx`
- `cli/print.ts`

## The Real Top-Level Shape

The code does not have a single straight-line startup path. It has a layered dispatch model:

1. `entrypoints/cli.tsx` handles very early fast paths before loading the full app.
2. `main.tsx` rewrites argv for special cases, decides session mode, and builds the main command handler.
3. `entrypoints/init.ts` performs one-time process initialization in the `preAction` hook.
4. The default action in `main.tsx` performs startup work, builds the initial session state, and launches either:
   - the interactive REPL, or
   - the headless runner in `cli/print.ts`.

## Layer 1: Bootstrap Dispatcher (`entrypoints/cli.tsx`)

`entrypoints/cli.tsx` is intentionally minimal. It exists to avoid loading the whole app for a set of special cases.

Verified fast paths include:

- `--version`
- `--dump-system-prompt`
- Chrome and computer-use MCP helper modes
- `--daemon-worker`
- `remote-control` / `rc` / legacy bridge aliases
- `daemon`
- background-session commands and `--bg`
- template job commands

Important detail: bridge mode is not only a flag inside `main.tsx`. It can be dispatched directly from `entrypoints/cli.tsx` after auth and policy checks.

## Layer 2: Main Entrypoint (`main.tsx`)

`main.tsx` owns the general CLI runtime.

Before it hands control to Commander, it performs several early transforms:

- sets the Windows `NoDefaultCurrentDirectoryInExePath` protection
- installs warning and SIGINT handling
- rewrites `cc://` and `cc+unix://` URLs
- handles deep-link URI launch flows
- strips and stashes `assistant` mode arguments
- strips and stashes `ssh` mode arguments
- determines whether the session is interactive
- sets client type metadata
- eagerly loads settings flags before `init()`

This makes `main.tsx` the real mode-normalization layer.

## Layer 3: One-Time Initialization (`entrypoints/init.ts`)

`init()` is memoized and runs from the Commander `preAction` hook.

Its verified responsibilities are:

- enable and validate config loading
- apply only safe environment variables before trust is established
- apply CA certificate settings early
- set up graceful shutdown
- start first-party event logging asynchronously
- warm OAuth account info and IDE detection
- start remote managed settings and policy loading promises
- record first start time
- configure mTLS and global HTTP agents
- preconnect to the Anthropic API
- initialize CCR upstream proxy support when running remotely
- set Windows shell defaults
- register cleanup handlers
- ensure scratchpad storage when enabled

Important boundary: `init()` is process setup, not session setup.

## Layer 4: Session Setup (`setup.ts`)

`setup()` runs in the default action path after CLI options are validated.

Its verified responsibilities are session-scoped:

- switch session ID when explicitly provided
- start UDS messaging when enabled
- capture teammate snapshot state
- restore interrupted terminal/iTerm backups in interactive mode
- set and normalize cwd
- snapshot hook configuration and start the file-change watcher
- create worktrees and optional tmux sessions
- update `projectRoot` when the session itself starts in a worktree
- initialize session memory and context-collapse background setup
- start plugin hook prefetch and hot reload
- register attribution, session file access, and team-memory background hooks
- attach analytics/error sinks
- prefetch API key helper data
- prefetch release-note activity for the interactive logo path
- enforce safety restrictions around bypass-permissions mode

`setup()` is the first point where the session's working directory and worktree can materially change.

## Interactive Versus Headless

The existing architecture notes simplify this too much. The actual split is:

### Interactive path

`main.tsx`:

1. runs `setup()`
2. loads commands and agent definitions
3. shows setup/trust/onboarding flows through `showSetupScreens()`
4. resolves MCP configs and starts MCP prefetch in the background
5. builds `initialState`
6. launches `<App><REPL /></App>` via `launchRepl()`

`interactiveHelpers.tsx` contributes two important boundaries:

- `showSetupScreens()` is the trust/onboarding/dialog gate
- `renderAndRun()` starts deferred prefetches, waits for UI exit, then shuts down

### Headless path

The headless path does **not** jump directly from `main.tsx` to `query.ts`.

Instead:

1. `main.tsx` calls `runHeadless(...)` from `cli/print.ts`
2. `cli/print.ts` manages structured IO, session resume, dynamic MCP updates, control messages, and output streaming
3. `cli/print.ts` calls `ask(...)` from `QueryEngine.ts`
4. `ask(...)` wraps a `QueryEngine`
5. `QueryEngine` eventually drives `query(...)` in `query.ts`

That wrapper stack matters because headless mode owns more than just one model request. It also owns transcript persistence, control-protocol output, permission reporting, MCP mutation, and replay behavior.

## Interactive REPL Launch Boundary

`replLauncher.tsx` is intentionally thin. It dynamically imports:

- `components/App.js`
- `screens/REPL.js`

and then renders:

```tsx
<App {...appProps}>
  <REPL {...replProps} />
</App>
```

That keeps the heavy UI tree off the critical path until startup is complete.

## MCP Startup Behavior

The current code explicitly avoids blocking first interactive render on MCP connection.

What actually happens:

- MCP config resolution starts early in `main.tsx`
- trust and onboarding happen before MCP connection work is allowed to matter
- `prefetchAllMcpResources(...)` is started after trust
- interactive mode does not await MCP completion before launching the REPL
- connected MCP clients and tools are merged into app state as they arrive

This is an important architecture fact: interactive startup is optimized for first render and first turn time, not for "all MCP servers connected before UI appears."

## Startup-Time Parallelism

The code intentionally overlaps work in a few places:

- `setup()` runs in parallel with command loading and agent-definition loading when worktree mode is not changing cwd
- user settings download in headless mode overlaps with later tool and MCP work
- MCP config loading overlaps with setup, trust dialog, and command loading
- session start hooks are kicked off in parallel with MCP connection work

This is one of the clearest design themes in the top-level runtime: startup favors staged concurrency over a monolithic initialization block.

## Important Corrections To The Existing Architecture Notes

- Print mode is not a direct `main.tsx -> query.ts` path; it goes through `cli/print.ts` and `QueryEngine.ts`.
- The startup model is layered across `entrypoints/cli.tsx`, `main.tsx`, `entrypoints/init.ts`, and `setup.ts`; each owns a different boundary.
- MCP connection work is intentionally non-blocking for the interactive REPL path.
- `setup()` is not just cwd/worktree setup. It also installs background session infrastructure, hook watchers, plugin hook prefetch, and safety gates.

