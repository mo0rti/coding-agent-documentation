# Query, Tools, And State Core

## Scope

This document covers the core runtime loop after startup:

- headless session wrapper
- query loop
- tool contract and execution
- reactive state versus session-global state

Primary source files:

- `cli/print.ts`
- `QueryEngine.ts`
- `query.ts`
- `Tool.ts`
- `tools.ts`
- `services/tools/toolOrchestration.ts`
- `services/tools/toolExecution.ts`
- `state/store.ts`
- `state/AppStateStore.ts`
- `bootstrap/state.ts`

## Runtime Ownership Model

The core runtime is split across four distinct layers:

1. `cli/print.ts` or the REPL submits work
2. `QueryEngine.ts` manages conversation/session concerns around a turn
3. `query.ts` runs the actual turn-by-turn model/tool loop
4. `services/tools/*` validates, authorizes, batches, runs, and serializes tool calls

That separation is real in the code and should be preserved in future docs.

## `QueryEngine.ts`: Session Wrapper Around The Loop

`QueryEngine` owns conversation-scoped concerns that are broader than a single sampling iteration.

Verified responsibilities include:

- storing mutable conversation history for the current conversation
- constructing the effective system prompt
- building `ToolUseContext`
- processing user input and slash-command effects before querying
- persisting accepted user messages before the model responds
- loading bundled, plugin, and skill metadata needed for system-init output
- wrapping `canUseTool` so permission denials can be reported to SDK consumers
- normalizing and yielding SDK-facing events and result objects
- preserving read-file cache state across calls

`ask(...)` is a one-shot convenience wrapper around `QueryEngine`.

## `query.ts`: The Actual Turn State Machine

`query.ts` is the core model/tool loop.

It defines:

- `QueryParams`
- a loop-local mutable `State`
- `query(...)`
- `queryLoop(...)`

Important verified property:

- `query.ts` does **not** directly read or write `AppState`

Instead, the caller passes in everything it needs through `QueryParams` and `ToolUseContext`.

That is one of the clearest architectural boundaries in the repo.

## What Happens Inside `query.ts`

The verified per-turn flow is roughly:

1. start with immutable query inputs and loop-local mutable state
2. build a query config snapshot
3. start background prefetches such as relevant memory and skill discovery
4. derive `messagesForQuery` from current messages
5. apply tool-result budget replacement if enabled
6. apply history snip compaction if enabled
7. apply microcompact
8. build and stream the API request
9. yield assistant output and stream events
10. if tool calls are emitted, run them through the tool orchestration layer
11. add tool results and generated attachments back into the message stream
12. handle stop hooks, prompt-too-long recovery, max-output-token recovery, and token-budget continuation logic
13. either continue with updated state or terminate the turn

This is much more than "send prompt, run tools, return answer." The loop also owns recovery and compaction behavior.

## Tool Contract (`Tool.ts`)

`Tool.ts` is the source of truth for the tool interface.

Key verified parts of the tool contract:

- tool identity: `name`, optional `aliases`, MCP metadata
- validation: `inputSchema`, optional `validateInput(...)`
- execution: `call(...)`
- permission hook: `checkPermissions(...)`
- behavior flags: `isConcurrencySafe`, `isReadOnly`, `isDestructive`, `requiresUserInteraction`
- prompt-facing metadata: `description(...)`, `prompt(...)`, `searchHint`
- UI rendering hooks for use, progress, result, error, and rejection states
- `maxResultSizeChars`
- deferral behavior for tool search

`buildTool(...)` supplies safe defaults so exported tools share a consistent shape.

## `ToolUseContext`: The Real Cross-Subsystem Glue

`ToolUseContext` is how tools reach the rest of the system.

It carries:

- the current tool pool
- commands
- main loop model
- MCP clients and resources
- app-state getters and setters
- abort controller
- message history
- UI helpers in interactive mode
- file-history and attribution updaters
- agent identity
- permission context
- query chain tracking

This is the practical integration boundary between the query loop, tools, state, and UI.

## Tool Assembly (`tools.ts`)

`tools.ts` is the built-in tool registry and the final merge point for built-in plus MCP tools.

Verified responsibilities:

- `getAllBaseTools()`: exhaustive built-in registry for the current build/runtime
- `getTools(permissionContext)`: built-in tools after simple-mode, deny-rule, REPL-mode, and enabled-state filtering
- `assembleToolPool(permissionContext, mcpTools)`: built-in tools plus filtered MCP tools, sorted for prompt-cache stability, then deduplicated by name

Important detail:

- built-ins are kept as a stable sorted prefix
- MCP tools are appended as a separately sorted partition

That is done intentionally for prompt-cache stability.

## Tool Execution (`services/tools/toolExecution.ts`)

`runToolUse(...)` and the permission/execution helpers under it own the actual tool lifecycle.

Verified order of operations:

1. resolve tool by name or alias
2. validate structured input
3. run tool-specific validation
4. start speculative classifier work for Bash when relevant
5. backfill observable input for hooks and observers
6. run pre-tool hooks
7. resolve permission decision
8. emit permission-related telemetry
9. either reject with a tool result or execute `tool.call(...)`
10. run post-tool hooks
11. serialize result into model-visible tool-result content

This file is the real "tool execution pipeline," not `tools.ts`.

## Tool Concurrency (`services/tools/toolOrchestration.ts`)

Tool batching is simple but important:

- consecutive concurrency-safe tool calls are grouped together
- everything else runs serially
- the default maximum concurrent tool count is 10

The batching decision depends on each tool's `isConcurrencySafe(...)` implementation after input parsing.

## State Is Split In Two

The repo does not use a single unified state store.

### 1. `bootstrap/state.ts`

This is process-global, non-React, session-scoped state.

Verified examples:

- session IDs and parent session IDs
- cwd and project root
- total cost and API duration
- model usage
- client type and session source
- telemetry handles and counters
- last API request data
- session trust, plan-mode, and cache-related latches
- invoked skills, slow operations, and several session-only flags

This file is explicitly the place for "be judicious with global state."

### 2. `state/store.ts` plus `state/AppStateStore.ts`

This is a small custom reactive store, not Zustand.

`state/store.ts` implements:

- `getState()`
- `setState(updater)`
- `subscribe(listener)`

`AppStateStore.ts` defines the shape used by the REPL and headless wrappers.

Verified `AppState` areas include:

- permission context
- tasks and agent routing
- MCP clients/tools/commands/resources
- plugin state
- file history and attribution
- notifications and elicitation queue
- prompt suggestion and speculation state
- remote-control / bridge UI state
- team/swarm state
- optional feature-specific UI state such as browser or computer-use state

## Practical Boundary Between The Two State Layers

Use this rule:

- if it must exist independent of React rendering, it usually belongs in `bootstrap/state.ts`
- if the REPL or headless wrapper needs reactive updates, it usually belongs in `AppState`

That distinction explains why both files exist and why they should not be merged mentally.

## Important Corrections To The Existing Docs

- The repo does not use Zustand. It uses a custom store in `state/store.ts` wrapped by React context.
- The query loop is intentionally decoupled from direct `AppState` access.
- `tools.ts` is registry and assembly logic; actual execution lives in `services/tools/toolExecution.ts` and `services/tools/toolOrchestration.ts`.
- Headless execution still uses the same core query loop, but only after passing through `cli/print.ts` and `QueryEngine.ts`.

