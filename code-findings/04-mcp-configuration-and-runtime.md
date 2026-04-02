# MCP Configuration And Runtime

## Scope

This document covers how MCP servers are defined, merged, connected, exposed to the rest of the app, and kept fresh at runtime.

Primary source files:

- `services/mcp/types.ts`
- `services/mcp/config.ts`
- `services/mcp/client.ts`
- `services/mcp/useManageMCPConnections.ts`
- `services/mcp/MCPConnectionManager.tsx`
- `tools/MCPTool/MCPTool.ts`
- `tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`
- `tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`
- `commands/mcp/index.ts`
- `commands/mcp/mcp.tsx`
- `main.tsx`
- `cli/print.ts`

## The Key Architectural Fact

MCP is not just "external tools".

In the current code, the MCP subsystem has four distinct responsibilities:

- merge server definitions from multiple config sources
- establish and cache transport-specific client connections
- translate remote MCP capabilities into local tools, prompt commands, and resources
- keep app state synchronized as servers connect, fail, re-authenticate, reconnect, or change their advertised lists

That is why MCP touches startup, state, commands, tool assembly, REPL UI, and headless control-mode code.

## The Core Types (`services/mcp/types.ts`)

The MCP config and connection model is explicit and richer than the high-level docs imply.

### Supported config scopes

Verified scope values include:

- `local`
- `user`
- `project`
- `dynamic`
- `enterprise`
- `claudeai`
- `managed`

`ScopedMcpServerConfig` is the base config plus metadata such as `scope` and optional `pluginSource`.

### Supported transport/config types

Verified transport variants include:

- `stdio`
- `sse`
- `sse-ide`
- `http`
- `ws`
- `ws-ide`
- `sdk`
- `claudeai-proxy`

So MCP is not only stdio child processes. It also includes remote HTTP/SSE/WebSocket servers, IDE-specific transports, in-process SDK servers, and claude.ai connector proxying.

### Runtime connection states

`MCPServerConnection` is a tagged union with these important states:

- `connected`
- `failed`
- `needs-auth`
- `pending`
- `disabled`

The rest of the app works from these states rather than from a simple connected/disconnected boolean.

## Config Loading And Merge Rules (`services/mcp/config.ts`)

`config.ts` is the policy and precedence layer.

It does more than parse files:

- validates server definitions against transport-specific schemas
- expands environment variables
- warns on missing env vars
- applies allow/deny policy filtering
- deduplicates overlapping servers across sources
- persists enable/disable state

### Source layering

The verified merge path for Claude Code-managed config is:

- plugin-contributed MCP servers
- user-level MCP config
- project-level MCP config
- local override config

Claude.ai connectors are fetched separately and treated as the lowest-precedence source. They are deduplicated against manual servers by content/signature, not just by name.

### Enterprise behavior

If an enterprise MCP config exists, the loader switches to the enterprise-controlled path and does not merge in the broader connector set the normal way.

### Policy filtering

`filterMcpServersByPolicy()` applies managed allow/deny rules after config resolution.

Important verified behavior:

- deny rules win first
- allow rules can match by server name, stdio command, or remote URL
- `sdk` servers are exempt from this policy filter

### Disabled-state persistence

Enable/disable is not just UI state.

`isMcpServerDisabled()` and `setMcpServerEnabled()` persist toggles to settings-backed lists so later startups respect the choice. There is also special handling for default-disabled built-in MCP servers.

## `client.ts`: The MCP Runtime Engine

`services/mcp/client.ts` is the runtime implementation layer.

Its main responsibilities are:

- connect to a server with the right transport
- memoize that connection
- fetch tools, prompts, and resources from a connected client
- wrap MCP capabilities into local `Tool` and `Command` objects
- handle OAuth/session-expiry edge cases
- transform MCP results into model-safe output

## Connection Strategy

### `connectToServer(...)`

`connectToServer(...)` is memoized by server name plus serialized config.

That means:

- the connection cache is keyed by actual config, not only by server name
- a config change forces a fresh connection
- later tool/resource reads can reuse a healthy client without reconnect churn

### Transport-specific setup

Verified transport handling includes:

- `stdio`: spawn subprocess with merged env
- `sse`: `SSEClientTransport` with OAuth-aware auth provider and a no-timeout EventSource fetch
- `http`: `StreamableHTTPClientTransport` with per-request timeout wrapping and header/auth handling
- `ws` / `ws-ide`: custom WebSocket transport
- `sse-ide`: unauthenticated IDE transport path
- `claudeai-proxy`: streamable HTTP transport through the claude.ai connector proxy
- `sdk`: rejected here and handled separately in `setupSdkMcpClients(...)`

Two local MCP servers can also be run in-process instead of as subprocesses:

- Claude-in-Chrome
- Computer Use / Chicago MCP

### Timeout and auth behavior

The connection layer has explicit protection for several failure modes:

- connect timeout
- stale per-request abort signals on HTTP transports
- 401 auth failures for remote transports
- MCP session expiry on HTTP-style transports
- repeated transport errors that never surface an `onclose`

Remote auth failures are converted into `needs-auth` server states and are cached for a short TTL to avoid hammering the same server every startup.

## Fetching Capabilities From A Connected Server

The capability fetch functions are all LRU-cached by server name:

- `fetchToolsForClient(...)`
- `fetchResourcesForClient(...)`
- `fetchCommandsForClient(...)`

These do not just pass through raw SDK payloads.

### Tool wrapping

`fetchToolsForClient(...)` turns each MCP tool into a local `Tool` using `MCPTool` as a base template.

Verified wrapper behavior includes:

- MCP tool names are normally prefixed with `mcp__<server>__<tool>`
- SDK MCP servers can optionally skip that prefix
- search hints and always-load hints can come from MCP `_meta`
- read-only/destructive/open-world hints are mapped from MCP annotations
- permission checks stay in passthrough mode and suggest local allow rules
- tool calls route through `ensureConnectedClient(...)` and `callMCPToolWithUrlElicitationRetry(...)`

### Prompt-to-command wrapping

`fetchCommandsForClient(...)` exposes MCP prompts as local `Command` objects of `type: 'prompt'`.

That means MCP can add slash-command-like prompt commands, not only tools.

### Resource fetching

`fetchResourcesForClient(...)` normalizes `resources/list` output and attaches the server name to each entry so resources can be tracked per server in app state.

## `getMcpToolsCommandsAndResources(...)`: The Main Assembly Pass

This is the main "connect and hydrate MCP" function.

It:

- resolves configs when they were not passed in
- immediately marks disabled servers as `disabled` without connecting
- skips recent auth-failing remote servers by returning `needs-auth`
- splits local servers from remote servers
- connects both groups with different concurrency limits
- fetches tools, prompts, optional MCP skills, and resources in parallel per server
- emits results incrementally through the callback rather than building one giant synchronous result

Two important implementation details:

- local servers (`stdio` and `sdk`) use a lower concurrency ceiling
- remote servers use a higher concurrency ceiling because they are network-bound rather than process-spawn-bound

## Resources Are A First-Class MCP Output, Not A Side Note

The codebase adds two helper tools when MCP resource support exists:

- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

These are not generic built-ins that operate without MCP state. They are MCP-aware helper tools that sit on top of connected clients.

### `ListMcpResourcesTool`

This tool:

- optionally filters by server
- reuses the fetch cache
- calls `ensureConnectedClient(...)` before listing
- tolerates one server failing without failing the whole listing

### `ReadMcpResourceTool`

This tool:

- requires an explicit server name and URI
- checks that the target server is connected and resource-capable
- calls `resources/read`
- persists binary blob content to disk instead of placing raw base64 into prompt context

That last behavior is important because it keeps large or binary resource payloads from polluting the model context window.

## MCP Tool Result Handling Is More Defensive Than The Old Docs Suggest

`client.ts` contains a full result-normalization path:

- `transformResultContent(...)`
- `transformMCPResult(...)`
- `processMCPResult(...)`
- binary persistence helpers for blobs
- image resizing/downsampling for image payloads
- truncation/persistence behavior for oversized outputs

So the MCP boundary is not a thin RPC pass-through. It is a guarded adaptation layer between external server output and Claude Code's internal tool/message model.

## URL Elicitation And Auth Flows

MCP tool calls can fail with URL-elicitation-required errors (`-32042`).

`callMCPToolWithUrlElicitationRetry(...)` handles that by:

- detecting the specific MCP error code
- extracting URL elicitation requests
- giving registered hooks a chance to resolve them automatically
- delegating to structured control-mode I/O in print/SDK mode
- or queueing an interactive elicitation in REPL mode
- retrying the original tool call if the elicitation is accepted/completed

This is another reason the MCP client layer has to know about both interactive and headless runtime contexts.

## Interactive MCP Lifecycle (`useManageMCPConnections.ts`)

In the interactive app, the React hook `useManageMCPConnections(...)` is the state-synchronization layer.

It is responsible for:

- creating pending placeholder clients in app state
- loading and connecting resolved configs
- registering list-changed notification handlers
- registering elicitation handlers
- reconnecting remote transports with exponential backoff
- exposing reconnect and enable/disable actions to the UI

### Two-phase startup in the interactive UI

The hook intentionally separates startup into two effects:

1. initialize known servers in app state as `pending` or `disabled`
2. connect Claude Code configs first, then connect claude.ai configs after dedup

That separation keeps the UI responsive and lets claude.ai connector fetches overlap with other work.

### Batched app-state updates

The hook does not write each server update into state immediately.

It batches MCP updates into a short timed window and applies them together so many near-simultaneous connection events do not thrash the UI store.

### Runtime refresh via notifications

If a connected server advertises list-change notifications, the hook refreshes cached data on demand:

- `tools/list_changed` invalidates and refetches tools
- `prompts/list_changed` invalidates and refetches MCP prompt commands
- `resources/list_changed` invalidates and refetches resources and any resource-derived MCP skills

This is the real source of truth for MCP list freshness, not a periodic global reload.

### Reconnection behavior

When a connected remote server closes unexpectedly, the hook:

- clears caches for that server
- checks whether the server was disabled
- retries remote transports with exponential backoff
- updates the server state to `pending`, then either `connected` or `failed`

`stdio` and `sdk` clients are treated differently and do not use the same automatic reconnect path.

### Toggle/reconnect actions

The interactive `/mcp` UI ultimately uses the context exposed by `MCPConnectionManager.tsx`, which is just a thin wrapper over `useManageMCPConnections(...)`.

The actual side effects happen in the hook:

- persist enabled/disabled state
- clear caches and disconnect on disable
- reconnect and repopulate tools/commands/resources on enable

## Headless And SDK MCP Paths (`main.tsx` and `cli/print.ts`)

The non-interactive path does not use the React hook.

Instead, MCP is handled in two different ways depending on server type.

### Normal process/network MCP in headless startup

`main.tsx` uses `prefetchAllMcpResources(...)` for headless startup.

That function internally uses `getMcpToolsCommandsAndResources(...)` and returns:

- clients
- tools
- commands

Those are then merged into the headless session config passed into `cli/print.ts`.

### SDK MCP in print/control mode

`cli/print.ts` handles `sdk` MCP servers separately via `setupSdkMcpClients(...)`.

That code path:

- creates in-process control transports
- connects SDK servers directly
- stores SDK tools in app state
- merges SDK clients and tools into the active tool pool used by headless turns

### Dynamic MCP during print/control mode

`cli/print.ts` also maintains a separate mutable `dynamicMcpState` for MCP servers added at runtime through control messages such as `mcp_set_servers`.

That means headless/control mode has its own MCP lifecycle machinery instead of sharing the interactive React hook.

## `/mcp` Is Mostly A UI/Management Surface

The `/mcp` command itself is not the runtime implementation.

`commands/mcp/index.ts` just registers the command, and `commands/mcp/mcp.tsx` routes to:

- settings UI
- reconnect flow
- enable/disable flow

So the command is a thin management entry point layered over the real MCP state and connection machinery.

## Practical Mental Model

Use this mental model when reading MCP code in this repo:

- `types.ts` defines the server/config/state vocabulary
- `config.ts` decides which servers exist and whether policy allows them
- `client.ts` knows how to connect, fetch, wrap, and normalize
- `useManageMCPConnections.ts` keeps the interactive app synchronized over time
- `main.tsx` and `cli/print.ts` run separate headless/control-mode MCP paths
- MCP contributes tools, prompt commands, resources, auth states, and UI management actions

## Corrections To The Existing Docs

- MCP is not just a "tool executor to external servers" branch. It also contributes prompt commands and resources.
- The app does not have one universal MCP startup path. Interactive mode, headless mode, and SDK/control mode each integrate MCP differently.
- MCP freshness is not maintained by simple re-read-on-startup logic. The runtime uses list-changed notifications plus targeted cache invalidation.
- Resource support is substantial, including helper tools and binary persistence, not a minor add-on.
- Config precedence and dedup are content-aware and policy-aware; they are not a simple merge of JSON files.
