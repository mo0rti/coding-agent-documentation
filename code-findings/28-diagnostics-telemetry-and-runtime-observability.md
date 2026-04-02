# Diagnostics, Telemetry, and Runtime Observability

## Scope

This document covers the main observability subsystems that are directly verifiable from this workspace snapshot: queued analytics dispatch, first-party event logging, local debug and error logs, no-PII diagnostic logs, customer OpenTelemetry telemetry, session/perfetto tracing, and IDE diagnostics tracking.

Primary files read for this slice:

- `utils/sinks.ts`
- `utils/log.ts`
- `utils/errorLogSink.ts`
- `utils/debug.ts`
- `utils/diagLogs.ts`
- `services/analytics/index.ts`
- `services/analytics/config.ts`
- `services/analytics/sink.ts`
- `services/analytics/firstPartyEventLogger.ts`
- `services/analytics/firstPartyEventLoggingExporter.ts`
- `services/analytics/metadata.ts`
- `services/analytics/sinkKillswitch.ts`
- `services/analytics/growthbook.ts`
- `entrypoints/init.ts`
- `main.tsx`
- `utils/privacyLevel.ts`
- `utils/telemetry/instrumentation.ts`
- `utils/telemetry/logger.ts`
- `utils/telemetry/sessionTracing.ts`
- `utils/telemetry/betaSessionTracing.ts`
- `utils/telemetry/events.ts`
- `utils/telemetry/perfettoTracing.ts`
- `utils/telemetry/pluginTelemetry.ts`
- `utils/telemetryAttributes.ts`
- `utils/telemetry/bigqueryExporter.ts`
- `services/diagnosticTracking.ts`
- `screens/REPL.tsx`
- `tools/FileWriteTool/FileWriteTool.ts`
- `tools/FileEditTool/FileEditTool.ts`
- `utils/attachments.ts`

## The Main Observability Split

The code does not have one single "logging" subsystem.

It separates at least six distinct layers:

- queued product analytics events
- local error and debug logs
- no-PII operational diagnostics logs
- customer OpenTelemetry metrics/logs/traces
- local and beta tracing layers on top of telemetry
- IDE diagnostics tracking for edited files

That split matters because each layer has different privacy rules, initialization timing, backends, and failure behavior.

## Analytics Uses A Small Queueing Front Door

`services/analytics/index.ts` is intentionally dependency-light.

Verified behavior:

- `logEvent(...)` and `logEventAsync(...)` can be called before the analytics sink exists
- events are queued in memory until `attachAnalyticsSink(...)` runs
- queued events are drained asynchronously once the sink attaches
- metadata is intentionally restricted to numbers/booleans at the public API boundary unless specially typed elsewhere

So most of the app logs analytics through a small front door rather than importing backend implementations directly.

## Sink Attachment Is Separate From Telemetry Bootstrap

`utils/sinks.ts` shows that sink attachment is its own startup step.

Verified behavior:

- `initSinks()` initializes the error log sink first
- then initializes the analytics sink
- `main.tsx`, `setup.ts`, `entrypoints/cli.tsx`, and bridge-specific entry paths call this helper
- some special entry points such as the Chrome/computer-use MCP servers call `initializeAnalyticsSink()` directly

This is a different lifecycle from OpenTelemetry initialization, which is handled later in `entrypoints/init.ts`.

## The Analytics Sink Fans Out To Datadog And 1P Logging

`services/analytics/sink.ts` is the actual routing layer behind `logEvent(...)`.

Verified behavior:

- events may be dropped by per-event sampling from `tengu_event_sampling_config`
- the sink can route to Datadog
- the sink can route to first-party event logging
- Datadog fanout strips `_PROTO_*` fields before dispatch
- first-party logging keeps the full payload, including `_PROTO_*` values intended for privileged columns
- per-sink killswitches are checked through `tengu_frond_boric`

So analytics fanout is not symmetrical. The code deliberately keeps first-party privileged payloads separate from general-access sinks.

## Analytics Privacy Gating Is Centralized

`services/analytics/config.ts` and `utils/privacyLevel.ts` establish shared suppression rules.

Verified behavior:

- analytics is disabled in tests
- analytics is disabled for Bedrock, Vertex, and Foundry providers
- privacy level `no-telemetry` disables analytics
- privacy level `essential-traffic` also disables analytics

This means analytics opt-out is not left to each emitter. There is a shared policy layer that multiple analytics systems reuse.

## 1P Event Logging Is A Separate Pipeline From Customer OTEL

`services/analytics/firstPartyEventLogger.ts` explicitly creates its own `LoggerProvider`.

Verified behavior:

- first-party event logging does not use the global OTEL logger provider
- it builds a separate provider with a `FirstPartyEventLoggingExporter`
- GrowthBook config controls batch size, queue size, delay, endpoint options, and retry behavior
- `reinitialize1PEventLoggingIfConfigChanged()` can rebuild the provider mid-session after GrowthBook refresh
- failed exports are handled by the exporter’s disk-backed retry path rather than by the global customer telemetry pipeline

So first-party event logging is observability infrastructure, but it is not the same subsystem as customer OTEL logs.

## The 1P Exporter Is Disk-Backed And Retry-Oriented

`services/analytics/firstPartyEventLoggingExporter.ts` adds resilience beyond normal OTEL batching.

Verified behavior:

- failed events are stored under `~/.claude/telemetry`
- batches are keyed by session ID plus a process-run UUID
- previous failed batches for the same session are retried in the background
- retries use bounded attempts and backoff
- auth fallback and chunked sending are part of the exporter logic

So the first-party event path is designed to survive transient network failure in a way that local debug logs or plain Datadog fanout are not.

## Analytics Metadata Has Its Own Privacy-Shaping Layer

`services/analytics/metadata.ts` is another separate boundary, not just a helper.

Verified behavior:

- built-in tool names are safe to log directly
- MCP tool names are sanitized unless detail logging is explicitly allowed
- official/builtin MCP sources are treated differently from user-configured ones
- plugin telemetry uses a twin-column model with hashed IDs plus redacted/raw fields
- detailed tool metadata gates differ between analytics and OTEL tracing

So the code treats observability metadata shaping as its own privacy subsystem, not a casual convention.

## Local Debug Logging Is File/Stderr Observability, Not Telemetry

`utils/debug.ts` implements a local debug channel with a very different contract.

Verified behavior:

- non-ant users only write debug logs when debug mode is enabled
- ant users always log for support/share style workflows
- logs can go to stderr or a file
- default file path is under `~/.claude/debug/<session>.txt`
- a `latest` symlink is maintained
- writes are buffered in normal mode but synchronous in explicit debug mode

So debug logs are local operator artifacts, not network telemetry.

## Error Logging Has Its Own Queue And Sink

`utils/log.ts` and `utils/errorLogSink.ts` mirror the analytics queue/sink pattern, but for errors.

Verified behavior:

- errors are queued before the sink attaches
- `logError(...)` always appends to an in-memory session-local error list
- persistent error and MCP log files are only written for ant users
- error reporting is suppressed for Bedrock/Vertex/Foundry, `DISABLE_ERROR_REPORTING`, and `essential-traffic`
- sink initialization order intentionally attaches error logging before analytics

So error logging is neither just "debug output" nor "analytics". It is a separate buffered sink with its own suppression rules.

## No-PII Diagnostic Logs Are Yet Another File Channel

`utils/diagLogs.ts` is narrower than both debug logs and error logs.

Verified behavior:

- output only exists when `CLAUDE_CODE_DIAGNOSTICS_FILE` is set
- the data is written synchronously as JSONL
- the comments explicitly forbid PII such as file paths, prompts, repo names, and project names
- `withDiagnosticsTiming(...)` wraps async operations with started/completed/failed timing records

`entrypoints/init.ts` uses this channel for initialization milestones such as:

- `init_started`
- config enabling
- safe env application
- mTLS/proxy setup
- scratchpad creation
- init completion

So no-PII diagnostics logs are an operational health channel, not a developer-debug channel.

## Customer OTEL Telemetry Is Trust-Aware And Settings-Aware

`entrypoints/init.ts` and `utils/telemetry/instrumentation.ts` show that customer OTEL telemetry is initialized later than sink attachment.

Verified behavior:

- `initializeTelemetryAfterTrust()` is called after trust acceptance
- for remote-managed-settings-eligible users, telemetry waits for managed settings to load first
- env vars may be re-applied before telemetry init so remote settings can affect telemetry config
- telemetry initialization is guarded against double init

So the OTEL stack is not a fire-at-process-start subsystem. It is deliberately delayed until trust and settings state are ready.

## OTEL Telemetry Has Multiple Distinct Outputs

`utils/telemetry/instrumentation.ts` sets up different signal pipelines rather than one exporter.

Verified behavior:

- metrics, logs, and traces each have separate exporter configuration
- OTLP protocol selection is per-signal or shared through env vars
- console exporters are stripped in formatted-output mode so NDJSON/SDK output is not corrupted
- internal BigQuery metrics export is added through a separate reader path
- BigQuery metrics can be enabled even when general customer OTEL export is not

That last point is important: the runtime distinguishes customer OTEL telemetry from Anthropic/internal metrics export.

## Perfetto Tracing Is Independent Of OTEL

`utils/telemetry/perfettoTracing.ts` is initialized from `initializeTelemetry()`, but it is not an OTEL exporter.

Verified behavior:

- it is enabled via `CLAUDE_CODE_PERFETTO_TRACE`
- it writes local Chrome trace JSON files under `~/.claude/traces` by default
- it can periodically write snapshots during the session
- it tracks agent hierarchy and trace events with its own in-memory event store
- it registers its own cleanup and exit hooks

So Perfetto is a local trace artifact path, not just another OTEL backend.

## Session Tracing Layers Runtime Semantics On Top Of OTEL And Perfetto

`utils/telemetry/sessionTracing.ts` is the runtime span API the rest of the app uses.

Verified behavior:

- interaction, LLM request, tool, blocked-on-user, tool-execution, and hook spans are modeled explicitly
- span contexts are tracked with `AsyncLocalStorage`
- stale-span cleanup runs independently of exporter shutdown
- when Perfetto is enabled, the same high-level spans also drive Perfetto-side span markers

So session tracing is the semantic tracing layer, while OTEL and Perfetto are transport/output layers beneath it.

## Beta Tracing Is A Separate Detailed Trace Path

`utils/telemetry/betaSessionTracing.ts` and `instrumentation.ts` keep beta tracing distinct from standard enhanced telemetry.

Verified behavior:

- beta tracing requires `ENABLE_BETA_TRACING_DETAILED=1` plus `BETA_TRACING_ENDPOINT`
- external users only get it in non-interactive mode or via an allowlist gate
- beta tracing sets up its own OTLP log/trace exporters against `BETA_TRACING_ENDPOINT`
- system prompts, tool schemas, new context, model output, and some tool I/O are logged with hash/dedup behavior
- thinking output is ant-only

So "enhanced telemetry" and "beta tracing" are related but separate code paths.

## Content Visibility In Tracing Is Explicitly Gated

The tracing code makes high-sensitivity fields opt-in.

Verified controls:

- `OTEL_LOG_USER_PROMPTS` gates prompt content in events/spans
- `OTEL_LOG_TOOL_CONTENT` gates extra tool-content span events
- `OTEL_LOG_TOOL_DETAILS` gates detailed tool/MCP names for telemetry metadata

This is important because observability here is not all-or-nothing. Several content-bearing layers have separate disclosure gates.

## IDE Diagnostics Tracking Is A Runtime Feedback Subsystem, Not Logging

`services/diagnosticTracking.ts` is easy to misclassify as telemetry, but it is really part of the editing/runtime feedback loop.

Verified behavior:

- the tracker initializes from the connected IDE MCP client
- at query start, `REPL.tsx` either initializes or resets it
- `beforeFileEdited(...)` captures baseline diagnostics for specific files
- `getNewDiagnostics()` compares current diagnostics against that baseline
- it specially handles `_claude_fs_right:` diagnostic variants

This subsystem is consumed locally:

- `FileWriteTool.ts` and `FileEditTool.ts` capture baselines before edits
- `utils/attachments.ts` turns newly detected diagnostics into `diagnostics` attachments
- `utils/messages.ts` and `components/DiagnosticsDisplay.tsx` render summaries for the user/model flow

So IDE diagnostics tracking is observability for the live coding session, not a backend analytics channel.

## Source-Of-Truth Summary

The code keeps observability split across multiple layers with different guarantees:

- analytics events are queued and sink-routed
- first-party event logging is separate from customer OTEL
- error/debug/diagnostic files are local channels with different privacy rules
- OTEL telemetry is trust-aware and exporter-specific
- session/beta/perfetto tracing are layered, not interchangeable
- IDE diagnostics tracking feeds the interactive coding loop rather than remote telemetry

The main correction to a shallow reading is that there is no single "telemetry system" here. The runtime deliberately separates business analytics, internal event logging, local diagnostics, customer OTEL export, local trace files, and editor diagnostics so they can evolve under different privacy, performance, and reliability constraints.
