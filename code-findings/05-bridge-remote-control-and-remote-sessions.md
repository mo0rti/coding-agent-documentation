# Bridge, Remote Control, And Remote Sessions

## Scope

This document covers the code-backed runtime paths for:

- Remote Control of a local CLI/REPL session from Claude web/mobile
- attaching the local REPL to a remote CCR session
- adjacent direct-connect remote entrypoints that `main.tsx` wires into the REPL

Primary source files:

- `hooks/useReplBridge.tsx`
- `bridge/initReplBridge.ts`
- `bridge/replBridge.ts`
- `bridge/remoteBridgeCore.ts`
- `bridge/replBridgeTransport.ts`
- `bridge/bridgeMessaging.ts`
- `bridge/createSession.ts`
- `bridge/replBridgeHandle.ts`
- `bridge/bridgeStatusUtil.ts`
- `main.tsx`
- `cli/print.ts`
- `remote/RemoteSessionManager.ts`
- `remote/SessionsWebSocket.ts`
- `hooks/useRemoteSession.ts`
- `cli/remoteIO.ts`
- `server/createDirectConnectSession.ts`

## The Key Architectural Fact

This codebase has multiple "remote" features, and they are not the same system.

The implementation separates three distinct models:

- `Remote Control`: the local CLI session stays authoritative, and Claude web/mobile talks to it through the bridge
- `Remote Session Attach`: the authoritative agent/session is already remote in CCR, and the local REPL acts as a viewer/controller for that remote session
- `Direct Connect` / SSH-style remote execution: the local REPL is pointed at another server or remote host instead of the normal local engine

Older high-level descriptions tend to blur these together. The code does not.

## Remote Control In The Interactive REPL

The interactive REPL owns Remote Control through `hooks/useReplBridge.tsx`.

That hook:

- watches `AppState.replBridgeEnabled`
- initializes or tears down the bridge connection
- forwards new transcript messages to the active bridge handle
- injects inbound user prompts from the bridge back into the local queue
- publishes bridge state and permission callbacks into `AppState`

Two details matter here:

- the hook uses an index-based diff over the local `messages` array, so it forwards only new eligible messages after the bridge becomes ready
- it stores the live handle globally through `setReplBridgeHandle(...)` so commands/tools outside the React tree can still interact with the bridge session

So the React hook is the owner of bridge lifecycle in interactive mode, but the actual transport/runtime implementation lives below it.

## `initReplBridge.ts`: The Bootstrap Boundary

`bridge/initReplBridge.ts` is the REPL-specific bootstrap wrapper.

It does the startup work that depends on runtime session state and policy:

- checks whether bridge mode is enabled
- requires claude.ai OAuth
- checks policy entitlement for remote control
- applies cross-process backoff for dead OAuth tokens
- proactively refreshes OAuth when needed
- derives the visible session title
- gathers git context
- chooses which bridge core to use

This file is intentionally separated from `replBridge.ts` so the lower-level bridge core can stay free of heavy REPL/session-storage imports.

## Session Title Derivation Is Shared Across Bridge Modes

Before picking a transport path, `initReplBridge.ts` derives a session title with this precedence:

- explicit initial name
- saved renamed title from session storage
- most recent meaningful user message
- generated fallback slug

It also wires an `onUserMessage(...)` callback that can update the bridge session title later:

- once after the first meaningful user prompt
- again after the third prompt using more conversation context

That logic is shared across both bridge implementations.

## Two Remote Control Bridge Implementations

The bridge has two different runtime cores.

### 1. Env-based bridge (`bridge/replBridge.ts`)

This is the older environment/work-dispatch path.

Its verified shape is:

- register a bridge environment
- create or reuse a remote session
- start a persistent work-poll loop
- when work arrives, obtain ingress credentials and attach a transport
- keep polling in the background so the session can be redispatched after transport loss

Important runtime pieces in this path:

- `environmentId` and `environmentSecret` are real runtime state, not UI metadata
- the bridge can reuse a prior environment/session in `perpetual` mode
- it stores a crash-recovery pointer so later runs can resume the same session
- it carries forward SSE sequence numbers across transport swaps to avoid replaying old inbound events
- it can reconnect in place when the environment is lost, and only falls back to creating a fresh session if that fails

### 2. Env-less bridge (`bridge/remoteBridgeCore.ts`)

This is the newer REPL-only path gated by `tengu_bridge_repl_v2`.

Its verified shape is:

- create a code session directly with `POST /v1/code/sessions`
- fetch bridge credentials from `POST /v1/code/sessions/{id}/bridge`
- build a CCR v2 transport with worker JWT plus worker epoch
- refresh bridge credentials proactively before expiry
- rebuild the transport on 401/auth recovery without creating a new session

What this path removes:

- no bridge environment registration
- no work polling
- no ack/heartbeat/deregister environment lifecycle

That is the most important conceptual difference in the new path.

## "Env-less" And "CCR v2" Are Not The Same Thing

The code is explicit about this distinction.

- `env-less` means skipping the Environments API poll/dispatch layer
- `CCR v2` means using the newer `/worker/*` transport protocol

Those are related but not identical:

- the env-less bridge always uses the v2 transport
- the env-based bridge can also use the v2 transport after work dispatch

So transport version and session-dispatch model are separate axes.

## Shared Transport And Messaging Layer

The two bridge cores share lower-level helpers.

### `bridge/replBridgeTransport.ts`

This file defines the bridge transport abstraction used by the bridge cores.

It wraps two variants behind one interface:

- v1: session-ingress/hybrid transport
- v2: SSE read stream plus CCR HTTP writer

Important verified behavior:

- v2 registration is part of transport setup
- v2 carries forward an SSE sequence high-water mark across rebuilds
- v2 can operate in outbound-only mode, where it writes events but does not open the inbound SSE stream
- v2 reports worker state, metadata, and delivery acknowledgments

### `bridge/bridgeMessaging.ts`

This is the shared protocol adaptation layer.

It:

- filters which local messages are eligible to send to the bridge
- extracts title-worthy user text
- parses inbound bridge messages
- ignores echoes of messages the local side already sent
- ignores re-delivered inbound prompts using UUID tracking
- routes server control requests such as `initialize`, `set_model`, `interrupt`, and permission-related requests

This is why both bridge cores can share the same high-level semantics even though their transport lifecycles differ.

## What The Bridge Handle Actually Represents

Both bridge cores return a `ReplBridgeHandle`.

That handle is the local API for:

- `writeMessages(...)`
- `writeSdkMessages(...)`
- `sendControlRequest(...)`
- `sendControlResponse(...)`
- `sendControlCancelRequest(...)`
- `sendResult()`
- `teardown()`

The handle is not just a socket wrapper. It is the active session bridge contract used by the REPL hook and by print/control mode.

## Remote Control In Print/SDK Mode (`cli/print.ts`)

Headless/SDK mode does not use `useReplBridge.tsx`.

Instead, `cli/print.ts` handles a `control_request` with subtype `remote_control` and dynamically calls `initReplBridge(...)` itself.

In that path:

- inbound bridge prompts are converted into queued prompt work and fed into the headless run loop
- bridge permission responses are injected back into `StructuredIO`
- local permission requests are forwarded out through the bridge handle
- bridge state changes are emitted as system messages on stdout

This is important architecturally:

- interactive REPL remote control is hook-driven
- print/SDK remote control is control-message-driven
- both reuse the same bridge core underneath

## Remote Sessions Are A Different System

`--remote`, `--teleport`, and assistant-viewer style flows do not use the bridge as the primary runtime.

Instead, `main.tsx` launches the REPL with a `remoteSessionConfig`, and the REPL side uses `hooks/useRemoteSession.ts` plus `remote/RemoteSessionManager.ts`.

That system assumes the authoritative session already exists remotely.

## `RemoteSessionManager`: Attach To A Remote CCR Session

`remote/RemoteSessionManager.ts` coordinates a different contract from the bridge:

- subscribe to an existing session over WebSocket
- send user input to the remote session via HTTP
- relay permission requests and permission responses
- expose connect/disconnect/reconnect/cancel actions

This manager does not create or own a local bridge environment. It is a remote-session client.

### `SessionsWebSocket.ts`

This is the remote-session subscription transport.

Verified behavior includes:

- connects to `/v1/sessions/ws/{sessionId}/subscribe`
- authenticates with OAuth headers
- pings periodically
- reconnects with bounded retries
- treats some close codes as permanent
- gives `4001` a short retry budget because session-not-found can be transient during compaction

### `useRemoteSession.ts`

This hook adapts remote session messages into the local REPL UI.

It:

- instantiates `RemoteSessionManager`
- converts remote SDK messages into local message objects
- filters echoed user messages using UUID tracking
- maps remote permission requests into the existing local permission UI
- tracks connection state and stuck-session timeouts
- handles `viewerOnly` behavior for assistant/session-view flows

So the local REPL remains the UI, but the agent loop itself is remote.

## `main.tsx`: Separate Remote Entry Paths

`main.tsx` wires multiple entry branches into the REPL.

### `--remote` / `--teleport`

This path creates or resumes a remote CCR session, then launches the REPL with:

- `remoteSessionConfig`
- remote-safe command filtering
- no local MCP clients or local initial tool pool

The resulting session is a local UI attached to a remote engine.

### Assistant viewer-only attach

There is also a viewer-only attach path that creates a remote session config with `viewerOnly: true`.

That path is intentionally more restrictive:

- the remote agent owns the session
- the local REPL is a viewer/controller rather than the primary actor

### `claude connect <url>`

This is not bridge mode and not a CCR remote-session attach.

`server/createDirectConnectSession.ts` shows the verified flow:

- POST to `${serverUrl}/sessions`
- validate the returned session payload
- return a `DirectConnectConfig` plus optional remote working directory

`main.tsx` then launches the REPL against that direct-connect config.

## SSH Remote Is A Distinct Entry Path Too

`main.tsx` also has a separate SSH branch that creates an SSH-backed session and then launches the REPL against it.

From the verified entrypoint code, this path:

- probes/creates a remote SSH session
- may run a local test mode instead of actual SSH
- updates cwd to the remote working directory
- points the REPL at that remote session

Important limitation of this document:

- `main.tsx` imports `./ssh/createSSHSession.js`, but the corresponding source file is not present in the current workspace snapshot I read
- so the orchestration in `main.tsx` is verified, but the deeper SSH session implementation is not documented here as source-of-truth

## `cli/remoteIO.ts`: Remote Worker I/O Is Another Specialized Path

`cli/remoteIO.ts` is the structured I/O implementation used when the CLI itself is acting as a remote worker.

It:

- creates a transport from the provided remote/session-ingress URL
- reads session ingress auth dynamically
- can initialize a CCR client for v2 worker flows
- registers delivery/state/metadata listeners
- writes keep-alive frames for bridge-topology sessions

This is adjacent to bridge behavior, but it serves the worker/remote-runner side rather than the REPL remote-control hook.

## Practical Mental Model

Use this model when reading the repo:

- `useReplBridge.tsx` is the interactive owner of Remote Control
- `initReplBridge.ts` chooses and bootstraps the actual bridge core
- `replBridge.ts` is the env-based bridge runtime
- `remoteBridgeCore.ts` is the env-less REPL bridge runtime
- `bridgeMessaging.ts` and `replBridgeTransport.ts` are the shared protocol/transport layer
- `useRemoteSession.ts` plus `RemoteSessionManager.ts` are for attaching the local REPL to an already-remote session
- `createDirectConnectSession.ts` and the SSH branch are separate remote-entry plumbing, not the bridge subsystem

## Corrections To The Existing Docs

- Remote Control and remote session attach are not one subsystem with different UIs. They are different runtime models.
- The env-less bridge is not just "bridge v2" in a generic sense. It specifically removes the Environments API poll/dispatch lifecycle.
- Print/SDK remote control does not go through the React `useReplBridge` hook, even though it reuses `initReplBridge(...)`.
- A remote REPL session created by `--remote` or `--teleport` is not the same thing as exposing a local REPL via the bridge.
- Direct connect is separate again: it creates a session on another server and points the REPL at that server rather than at the normal local engine.
