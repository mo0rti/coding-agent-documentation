# Structured I/O Control Protocol, SDK Events, and Headless Control Flow

## Scope

This document covers the non-interactive SDK transport layer that sits underneath print mode and remote-control style consumers:

- the NDJSON stdin/stdout protocol surface
- control request, response, and cancellation handling
- SDK-visible permission, elicitation, MCP, and sandbox prompts
- system-init and message-mapping rules
- session-state and SDK-only event side channels
- the split between `StructuredIO`, `RemoteIO`, transport classes, and `print.ts`

Primary files read for this slice:

- `cli/structuredIO.ts`
- `cli/remoteIO.ts`
- `cli/transports/WebSocketTransport.ts`
- `cli/print.ts`
- `utils/sessionState.ts`
- `utils/sdkEventQueue.ts`
- `utils/messages/systemInit.ts`
- `utils/messages/mappers.ts`
- `utils/queryHelpers.ts`
- `utils/controlMessageCompat.ts`
- `entrypoints/sdk/controlSchemas.ts`

## High-Level Model

The code does not treat SDK mode as "print mode with JSON output".

There are three distinct layers:

1. `StructuredIO` implements the protocol boundary: parse input lines, track pending control requests, and serialize outbound protocol messages in order.
2. `RemoteIO` swaps stdio for a remote transport while preserving the same message protocol.
3. `cli/print.ts` runs the actual headless session loop on top of that protocol layer.

So the source-of-truth model is:

- `StructuredIO` is the protocol adapter
- `RemoteIO` is a transport specialization
- `print.ts` is the runtime that consumes the adapter

## The Verified Wire Model Is NDJSON Plus A Control Subprotocol

The protocol is line-delimited JSON, not a bespoke RPC channel.

`entrypoints/sdk/controlSchemas.ts` shows the verified message families:

- regular SDK conversation messages
- `control_request`
- `control_response`
- `control_cancel_request`
- `keep_alive`
- `update_environment_variables` on stdin

The control layer is a subprotocol inside the same stream, not a separate socket.

Verified control request families include:

- `initialize`
- `interrupt`
- `can_use_tool`
- `set_permission_mode`
- `set_model`
- `set_max_thinking_tokens`
- `mcp_status`
- `get_context_usage`
- `hook_callback`
- `mcp_message`
- `rewind_files`
- `cancel_async_message`
- `seed_read_state`
- `mcp_set_servers`
- `reload_plugins`
- `mcp_reconnect`
- `mcp_toggle`
- `stop_task`
- `apply_flag_settings`
- `get_settings`
- `elicitation`

One important workspace caveat:

- runtime files import `src/entrypoints/sdk/controlTypes.js`, but the type-source file is not present in this workspace snapshot
- the verified wire shapes here come from `controlSchemas.ts` plus runtime usage sites

## `StructuredIO` Is The Core Protocol Adapter

`StructuredIO` owns the main protocol mechanics.

Verified responsibilities:

- reads an async text stream and splits it into newline-delimited frames
- ignores empty lines
- normalizes legacy `requestId` to `request_id` via `normalizeControlMessageKeys(...)`
- drops `keep_alive`
- applies `update_environment_variables` directly to `process.env`
- yields only valid SDK user/system/assistant/control messages into the headless runner
- exposes a single ordered outbound queue via `outbound`

That last point matters a lot.

The code comment is explicit that both `sendRequest(...)` and `print.ts` enqueue into the same `outbound` stream so `control_request` traffic cannot overtake already-queued stream events.

So the protocol writer is intentionally serialized even though different runtime subsystems emit messages.

## Input Replay Is Part Of The Protocol Layer

`StructuredIO` is not only a passive parser.

It also supports `prependUserMessage(...)`, which injects a synthetic user turn ahead of the next unread inbound message. The read loop re-checks that prepend queue between parsed lines, so injected turns can land correctly even mid-stream.

This is used by headless startup and hook flows to push initial user content into the session without bypassing the structured protocol.

## Request Lifecycle: Send, Resolve, Abort, And Duplicate Suppression

`sendRequest(...)` is the central control-request helper.

Verified behavior:

- creates a `control_request` with a generated or supplied `request_id`
- rejects immediately if input is already closed or the abort signal is already aborted
- enqueues the request onto `outbound`
- records the request in `pendingRequests`
- optionally wires abort to emit `control_cancel_request`
- immediately rejects the local promise on abort rather than waiting for host acknowledgment

The runtime also tracks resolved `tool_use_id` values in a bounded set.

That is not bookkeeping noise. It prevents duplicate or replayed `control_response` frames from re-triggering permission handling and producing duplicate `tool_use` or `tool_result` state after reconnects.

So the protocol layer is explicitly reconnect-aware and duplicate-aware.

## Incoming `control_response` Handling Is More Defensive Than It Looks

When `StructuredIO` receives a `control_response`, it:

- closes command lifecycle reporting if a `uuid` is present
- looks up the pending request by `request_id`
- drops already-resolved duplicate tool responses
- optionally forwards orphaned responses to `unexpectedResponseCallback`
- parses successful payloads through the registered schema
- rejects pending promises on `error` responses

This means orphaned or late responses are first-class protocol cases, not undefined behavior.

That matters because headless mode can race:

- SDK host permission decisions
- bridge-fed permission decisions
- reconnect replay
- hook-based permission resolution

## Permission Flow Is A Protocol Race, Not A Single Prompt Path

`createCanUseTool(...)` in `StructuredIO` is one of the most important verified behaviors in this subsystem.

The flow is:

1. run static permission evaluation through normal app permission logic
2. if the result is already allow or deny, return it immediately
3. otherwise start permission-request hooks in parallel
4. immediately send `can_use_tool` to the SDK host
5. whichever path resolves first wins
6. if hooks win, abort the SDK-side request
7. if SDK wins, the hook result is ignored

So SDK permission handling is intentionally symmetric with terminal-mode behavior: hooks do not block the host UI from prompting, and host prompts do not block hooks from auto-deciding.

Two other verified details matter here:

- `notifySessionStateChanged('running')` is only fired once no other permission prompts remain
- `onPermissionPrompt(...)` gets a normalized `RequiresActionDetails` object built from the tool definition, not raw protocol data alone

## Bridge And SDK Permission Paths Share The Same Control Channel

`StructuredIO` also exposes bridge-specific hooks:

- `setOnControlRequestSent(...)`
- `setOnControlRequestResolved(...)`
- `injectControlResponse(...)`

Those methods let a bridge forward permission requests to another surface and then inject the winning `control_response` back into the same pending-request machinery.

That means the bridge path is not a separate permission subsystem. It is an alternate producer and consumer on the same `can_use_tool` protocol.

## Elicitation, MCP, And Sandbox Network Access Reuse The Same Machinery

The same request/response machinery is reused for several other host interactions.

Verified cases:

- `handleElicitation(...)` sends an `elicitation` control request and returns parsed MCP user input or `cancel`
- `sendMcpMessage(...)` forwards JSON-RPC payloads through an `mcp_message` control request
- `createSandboxAskCallback()` forwards sandbox network prompts through `can_use_tool` using the synthetic tool name `SandboxNetworkAccess`

That last point is especially important.

The code does not define a separate protocol subtype for sandbox network permission. It intentionally piggybacks on the existing tool-permission protocol so SDK hosts can reuse their existing approval UI.

## `system/init` Is The Canonical Session-Capabilities Frame

`buildSystemInitMessage(...)` in `utils/messages/systemInit.ts` builds the first capability-bearing SDK message for a session.

Verified fields include:

- cwd
- session ID
- tool names
- MCP server names and statuses
- model
- permission mode
- slash commands
- API key source
- beta flags
- Claude Code version
- output style
- agents
- skills
- plugins
- fast-mode state

There is also a hidden feature-gated `messaging_socket_path` field for UDS messaging.

One compatibility detail is called out directly in code:

- the wire tool name still translates the newer Agent tool back to the legacy Task name for older SDK consumers

So `system/init` is not just metadata. It is the compatibility and capability declaration frame for downstream SDK clients.

## Internal Messages Are Remapped Before They Become SDK Messages

The SDK stream is not a raw dump of internal transcript messages.

`utils/messages/mappers.ts` and `utils/queryHelpers.ts` show several compatibility and shaping passes:

- compact boundaries become explicit SDK system messages
- local command output is rewritten into synthetic assistant messages instead of a dedicated local-command subtype
- ExitPlanMode V2 tool calls get the current plan injected into tool input for SDK consumers
- assistant and user progress messages from agents and skills are remapped into normal SDK assistant/user messages
- bash and PowerShell progress are throttled into `tool_progress` messages, mostly for remote-style consumers

So the SDK-visible stream is a curated view over internal state, not a direct transcript serialization.

## Session State Has A Separate Side Channel

`utils/sessionState.ts` defines a small but important runtime authority:

- `idle`
- `running`
- `requires_action`

`notifySessionStateChanged(...)` does more than call a listener.

Verified behavior:

- updates in-process session state
- mirrors pending action details into external session metadata
- clears pending action metadata when leaving blocked state
- clears `task_summary` on idle
- optionally emits `system/session_state_changed` into the SDK event queue when `CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS` is enabled

This means session-state tracking is not solely UI state and not solely transcript state. It is shared runtime metadata that can feed CCR, SDK consumers, and session storage.

## SDK-Only Events Are A Parallel Queue, Not Normal Transcript Messages

`utils/sdkEventQueue.ts` adds another layer that is easy to miss if you only read the main output loop.

The queue is used only for non-interactive sessions and carries SDK-only system events such as:

- `task_started`
- `task_progress`
- `task_notification`
- `session_state_changed`

When drained, each event is stamped with:

- `uuid`
- `session_id`

This is important architecturally:

- these events are not generated by the model
- they are not normal transcript messages
- they are appended into the output stream by the headless runtime as side-channel state for SDK consumers

## `print.ts` Is The Headless Orchestrator Above The Protocol Layer

`cli/print.ts` uses `StructuredIO`, but it is not part of the protocol adapter itself.

Its verified responsibilities in this slice include:

- constructing `StructuredIO` or `RemoteIO`
- attaching sandbox ask callbacks after transport setup
- feeding `handleElicitation(...)` and `sendMcpMessage(...)` into query execution
- draining `drainSdkEvents()` into the outbound stream at several turn boundaries
- emitting status updates like permission mode, auth state, and rate-limit events
- keeping `lastMessage` focused on result-bearing stream messages rather than protocol/control traffic

That last behavior matters for output semantics.

`print.ts` explicitly excludes:

- `control_request`
- `control_response`
- SDK-only system events

from final result accounting, so JSON/text outputs still reflect the actual turn result rather than protocol chatter.

## `runHeadlessStreaming(...)` Uses The Protocol Queue As The Single Output Spine

The headless runner does not write directly to stdout for every event.

Instead:

- `runHeadlessStreaming(...)` enqueues onto `structuredIO.outbound`
- result messages, status messages, task events, auth events, and rate-limit events all share that path
- `StructuredIO.write(...)` or `RemoteIO.write(...)` is then responsible for actual delivery

That architecture keeps ordering consistent across:

- model output
- host control traffic
- runtime side-channel events
- reconnect/replay-capable remote transports

So the output stream is intentionally centralized before transport delivery.

## `RemoteIO` Preserves The Protocol While Replacing The Transport

`RemoteIO` subclasses `StructuredIO`, but its role is transport-focused.

Verified behavior:

- replaces stdin with a `PassThrough` fed by remote transport data
- chooses transport by URL and auth headers
- supports dynamic auth-header refresh on reconnect
- mirrors remote data back to local stdout in bridge/debug cases
- optionally enables CCR v2 client behavior for internal-event persistence and state reporting
- can emit silent bridge keep-alive frames on an interval

So `RemoteIO` is not a second protocol. It is the same protocol running over a remote transport.

## The WebSocket Transport Is Replay-Aware, Not Fire-And-Forget

`cli/transports/WebSocketTransport.ts` makes the reconnect semantics explicit.

Verified behavior:

- buffers UUID-bearing outbound messages
- stores `lastSentId`
- replays buffered messages after reconnect
- can evict already-confirmed buffered messages based on the server's last-seen ID
- applies exponential backoff with jitter
- detects probable system sleep and resets reconnect budget
- treats some close codes as permanent unless refreshed auth headers change the situation
- sends both ping frames and protocol-level `keep_alive` data frames

So remote SDK mode is not "best effort stream-json over WebSocket". It is a stateful replaying transport designed to survive real reconnect scenarios.

## Source-Of-Truth Corrections From Code

The code makes several architecture corrections clear:

- structured output format is not the SDK protocol; the SDK protocol is a broader NDJSON stream with embedded control messages
- permission prompting is not a single modal UI call; it is a raced, abortable protocol between hooks, bridge, and SDK host
- SDK task and session events are not transcript messages; they are runtime-generated side-channel events
- `RemoteIO` does not define a different SDK protocol; it reuses `StructuredIO` semantics over reconnect-capable transports
- `print.ts` is not the protocol implementation; it is the session runner layered on top of the protocol implementation

## Caveats

One type-surface caveat remains in this workspace snapshot:

- runtime files import `entrypoints/sdk/controlTypes.js`, but that type-source file is not present here
- the message shapes in this document are verified from `controlSchemas.ts` and the runtime call sites, which are sufficient to describe the live protocol behavior
