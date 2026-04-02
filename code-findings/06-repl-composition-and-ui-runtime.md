# REPL Composition And UI Runtime

## Scope

This document covers how the interactive REPL is composed at runtime and which parts of the UI own which responsibilities.

Primary source files:

- `replLauncher.tsx`
- `components/App.tsx`
- `state/AppState.tsx`
- `state/AppStateStore.ts`
- `screens/REPL.tsx`
- `components/Messages.tsx`
- `components/PromptInput/PromptInput.tsx`
- `hooks/useMergedTools.ts`
- `hooks/useMergedCommands.ts`
- `hooks/useMergedClients.ts`
- `hooks/useQueueProcessor.ts`
- `utils/handlePromptSubmit.ts`
- `interactiveHelpers.tsx`

## The Key Architectural Fact

The interactive REPL is not a thin shell around `query.ts`.

In the current codebase:

- `query.ts` owns the turn loop
- `REPL.tsx` owns interactive session orchestration
- `Messages.tsx` owns transcript shaping and rendering policy
- `PromptInput.tsx` owns user-input behavior, footer controls, and input-adjacent modal UI
- `AppState` and context providers supply long-lived UI/runtime state

That separation is why the REPL file is large: it is the coordination layer for many subsystems, not just "screen markup".

## Launch Path

The interactive render path is intentionally small before the REPL loads.

### `interactiveHelpers.tsx`

After trust/setup screens, `renderAndRun(...)`:

- renders the React tree into Ink
- starts deferred prefetches
- waits for exit
- runs graceful shutdown

### `replLauncher.tsx`

`launchRepl(...)` dynamically imports:

- `components/App.js`
- `screens/REPL.js`

and renders:

- `<App ...><REPL ... /></App>`

So the launcher is intentionally just a code-splitting and composition boundary.

## `App.tsx`: The Minimal Top-Level Wrapper

`components/App.tsx` is deliberately small.

It provides:

- `FpsMetricsProvider`
- `StatsProvider`
- `AppStateProvider`

It does not contain business logic for the REPL itself.

That means the real interactive behavior starts in `REPL.tsx`, not in the top-level app wrapper.

## `AppStateProvider`: The Reactive UI State Boundary

`state/AppState.tsx` wraps a custom external store, not React reducer state and not Zustand.

Important verified behavior:

- store is created once with `createStore(...)`
- consumers subscribe with `useSyncExternalStore`
- components select slices through `useAppState(selector)`
- write-only access uses `useSetAppState()`
- non-React code can receive the store via `useAppStateStore()`

The provider also adds:

- mailbox context
- optional voice context
- settings change propagation into app state

So AppState is the interactive UI/runtime state hub, while bootstrap/session globals remain outside it.

## What `REPL.tsx` Actually Owns

`screens/REPL.tsx` is the runtime coordinator for the interactive session.

It owns or coordinates:

- the transcript message array
- the active loading/query state
- abort-controller lifecycle
- remote session and bridge hooks
- command queue execution
- local modal/dialog focus
- session restoration/resume behavior
- permission queues
- spinner and streaming UI state
- prompt submission flow
- layout composition for fullscreen vs transcript mode

This is why it imports both UI components and many non-UI hooks/utilities.

## The REPL Has Multiple State Layers

The screen intentionally mixes several state sources, each for a different reason.

### 1. Local React state

Examples:

- `messages`
- `inputValue`
- `toolUseConfirmQueue`
- streaming text/thinking state
- dialog-local visibility flags

These are the high-churn per-screen state values.

### 2. AppState store state

Examples:

- MCP state
- plugin state
- task state
- bridge/remote flags
- permission mode context
- teammate/expanded-view state

These are cross-component runtime values that need reactive reads from multiple parts of the UI tree.

### 3. Bootstrap/global runtime state

Examples come from `bootstrap/state.ts`, such as:

- session ID
- original cwd
- model/session metadata
- token-budget and profiling counters

These are session-global values that the UI reads but does not own.

## Message State Is Centralized In `REPL.tsx`

The message array lives in `REPL.tsx`, not in AppState.

That is a meaningful architectural choice.

Verified behavior:

- `messages` is held in local React state
- a wrapper around `setMessages(...)` keeps a ref synchronized immediately
- deferred rendering uses `useDeferredValue(messages)` when helpful
- several subsystems append or replace messages through the same central setter

This keeps transcript ordering and screen rendering under one owner instead of splitting message truth across store and local state.

## `Messages.tsx`: Transcript Preparation And Rendering Policy

`components/Messages.tsx` is not just a dumb list renderer.

It performs substantial transcript shaping before row render:

- normalization
- grouping
- reordering for UI presentation
- collapsing of repeated/related items
- brief-mode filtering
- virtualization / render-cap logic
- transcript search integration
- streaming-thought / streaming-tool rendering

Examples of shaping helpers it uses include:

- `normalizeMessages(...)`
- `applyGrouping(...)`
- collapse helpers for hooks, reads, background bash notifications, and teammate shutdowns

So transcript appearance policy lives here, not in `REPL.tsx` and not in `query.ts`.

## `Messages.tsx` Also Owns Render-Scale Policy

The component explicitly handles large-session rendering tradeoffs.

Verified behaviors include:

- virtualized rendering when fullscreen scroll infrastructure is available
- a non-virtualized safety cap for very large transcripts
- transcript-mode search/jump support
- optional hiding of older thinking blocks
- special brief-mode transcript filtering

This is an important design point: UI scalability is handled in the message renderer, not as an afterthought in the screen component.

## `PromptInput.tsx`: More Than A Text Box

`components/PromptInput/PromptInput.tsx` is the interactive command bar and footer controller.

It owns:

- text input and cursor behavior
- history search and input history
- slash command suggestions and other suggestion triggers
- footer pills and footer navigation
- prompt-mode switching
- paste/image reference handling
- prompt-local modal surfaces such as model picker, search, bridge dialog, teams dialog, and task dialogs
- submitting to the higher-level `onSubmit(...)` callback

It also reads app state for things that affect the footer and local controls, such as:

- bridge state
- task/team state
- prompt suggestion/speculation state
- teammate view state

So the prompt input area is effectively its own interactive subsystem.

## Submission Logic Lives Outside `PromptInput`

`PromptInput.tsx` does not directly execute turns.

Instead, it calls the screen-level `onSubmit(...)`, and the shared submission helper is `utils/handlePromptSubmit.ts`.

That helper is the bridge between UI input and the query loop.

Verified responsibilities in `handlePromptSubmit(...)` include:

- validation and early exits
- expansion of pasted text/image references
- special handling for exit words
- immediate local-JSX slash-command dispatch
- queue-aware execution
- eventually calling `processUserInput(...)` / `onQuery(...)`

This keeps input parsing and command-dispatch policy out of the visual input component.

## Queue Processing Is A Dedicated Layer

Queued commands are not processed ad hoc inside the render body.

`hooks/useQueueProcessor.ts` subscribes to:

- the unified command queue
- the `QueryGuard`

and only processes queued input when:

- no query is active
- there is queued work
- no blocking local JSX UI is active

That means queue draining is a reactive side-effect layer, not a special-case inside the prompt input or query loop.

## Tool, Command, And MCP Client Pools Are Merged In The REPL Layer

The REPL composes its live capability pools through dedicated hooks:

- `useMergedTools(...)`
- `useMergedCommands(...)`
- `useMergedClients(...)`

These hooks merge:

- startup-provided tools/commands/clients from `main.tsx`
- dynamically discovered MCP additions from app state

This is important because the interactive screen can outlive startup assumptions:

- MCP connections may come online later
- plugin state may refresh
- bridge/remote mode may narrow behavior

So the REPL keeps its own live merged view of capabilities instead of freezing startup props as final truth.

## The REPL Is The Junction Between UI And Query

The screen constructs the `toolUseContext` passed into query execution.

That context packages together live runtime concerns such as:

- current tool pool
- MCP clients
- app-state getters/setters
- message mutation helpers
- UI callbacks for things like dialogs or spinner/progress state

This is the practical boundary where the pure-ish query runtime gets the interactive environment it needs without directly importing REPL UI components.

## Remote And Bridge Integrations Plug Into The REPL, They Do Not Replace It

The REPL screen integrates remote variants through hooks rather than swapping out the whole screen:

- `useRemoteSession(...)`
- `useDirectConnect(...)`
- `useSSHSession(...)`
- `useReplBridge(...)`

That means the same REPL shell can host:

- the normal local query loop
- a remote CCR-backed session
- direct-connect/SSH-backed sessions
- an active bridge connection

The UI shell remains largely the same while the message/input transport changes underneath it.

## Layout Composition At The Bottom Of `REPL.tsx`

The final render confirms the ownership split.

Important structural facts:

- `MCPConnectionManager` wraps the main interactive layout so MCP lifecycle stays active during the session
- `FullscreenLayout` is the main shell in fullscreen mode
- `Messages` renders the transcript area
- `PromptInput` renders the bottom interactive surface
- many dialogs/overlays are injected at the REPL level rather than being owned by `PromptInput` or `Messages`

This is a strong signal that `REPL.tsx` is the composition root for the interactive product surface.

## Fullscreen vs Transcript Mode

`REPL.tsx` has a real mode split rather than one layout with minor toggles.

Verified distinctions include:

- fullscreen layout path with virtual scrolling and overlays
- transcript/export-style path with alternate screen handling and different rendering behavior
- special handling for message-action navigation and transcript search

So "the REPL UI" is really multiple rendering modes sharing one orchestration core.

## What Is Intentionally Not In The Query Loop

The following concerns are handled in REPL/UI code rather than in `query.ts`:

- prompt input editing behavior
- footer navigation and local dialogs
- transcript virtualization and grouping for display
- queued command draining rules
- bridge/remote session UI integration
- most modal and overlay state

That division is healthy and should be preserved in future docs: the query loop is not the UI controller.

## Practical Mental Model

Use this model when reading the interactive code:

- `launchRepl()` mounts the interactive tree
- `App.tsx` provides the outer context/state wrappers
- `AppStateProvider` exposes the reactive store
- `REPL.tsx` is the orchestration root
- `Messages.tsx` owns transcript transformation and rendering
- `PromptInput.tsx` owns the interactive input surface
- `handlePromptSubmit.ts` and `useQueueProcessor.ts` bridge UI input into execution

## Corrections To The Existing Docs

- The REPL is not just `Messages + PromptInput`. `REPL.tsx` is a substantial runtime coordinator.
- The transcript message array is local screen state, not AppState store state.
- `Messages.tsx` does significant transformation and performance management; it is not a dumb renderer.
- `PromptInput.tsx` is a control surface with footer/navigation/dialog responsibilities, not just a text field.
- Dynamic MCP, bridge, remote-session, and queued-command behavior are all integrated at the REPL layer rather than being hidden inside `query.ts`.
