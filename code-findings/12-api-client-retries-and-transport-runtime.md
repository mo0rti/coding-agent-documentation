# API Client, Retries, And Transport Runtime

## Scope

This document covers how Claude Code assembles API requests, constructs provider-specific SDK clients, runs streaming and non-streaming request flows, applies retry and fallback behavior, normalizes errors, and logs request/runtime telemetry.

Primary source files:

- `services/api/client.ts`
- `services/api/claude.ts`
- `services/api/withRetry.ts`
- `services/api/errors.ts`
- `services/api/errorUtils.ts`
- `services/api/logging.ts`
- `services/api/bootstrap.ts`
- `services/api/promptCacheBreakDetection.ts`
- `utils/auth.ts`
- `utils/betas.ts`
- `utils/model/providers.ts`
- `utils/model/model.ts`
- `utils/model/bedrock.ts`
- `bootstrap/state.ts`

## The Key Architectural Split

The code does not have one generic "API request" path.

There are four separate layers:

1. client construction for the selected provider
2. request assembly and message normalization
3. retry, fallback, and transport recovery
4. response/error shaping plus telemetry

Those layers are intentionally separate.

For example:

- `services/api/client.ts` decides which SDK and auth mechanism to use
- `services/api/claude.ts` decides what request body to send and whether to stream
- `services/api/withRetry.ts` decides whether to retry, back off, or trigger model fallback
- `services/api/errors.ts` decides what user-visible assistant error message should be synthesized

So the runtime is not "call SDK, catch error". It is a multi-stage pipeline with different responsibilities at each stage.

## API Client Construction Is Provider-Specific And Happens Per Request Attempt

`getAnthropicClient(...)` in `services/api/client.ts` is the entry point for live client construction.

Verified behavior:

- every client gets shared default headers such as session ID, user agent, and optional remote/session metadata
- custom headers from `ANTHROPIC_CUSTOM_HEADERS` are merged in
- additional-protection headers can be added from env
- OAuth refresh checks run before client construction
- non-subscriber first-party clients may also get an `Authorization` bearer header from `ANTHROPIC_AUTH_TOKEN` or `apiKeyHelper`

But after that shared setup, the code diverges sharply by provider:

- first-party uses `@anthropic-ai/sdk`
- Bedrock uses `@anthropic-ai/bedrock-sdk`
- Vertex uses `@anthropic-ai/vertex-sdk`
- Foundry uses `@anthropic-ai/foundry-sdk`

This means the "Anthropic client" abstraction is really a provider-specific factory.

## The Fetch Layer Adds First-Party-Only Request IDs

`buildFetch(...)` wraps the underlying fetch implementation.

Verified behavior:

- it injects `x-client-request-id` only for first-party Anthropic API calls on first-party base URLs
- it does not inject that header for Bedrock, Vertex, or Foundry
- it logs the request pathname and client request ID for diagnostics

The code comments explain why this is restricted:

- first-party infrastructure can log and correlate the header
- third-party providers and strict proxies may reject unknown headers

So request-ID instrumentation is transport-aware, not universal.

## Query Execution Splits Immediately Into Streaming And Non-Streaming Entry Points

`services/api/claude.ts` exposes two top-level query surfaces:

- `queryModelWithStreaming(...)`
- `queryModelWithoutStreaming(...)`

Both delegate into the same lower-level `queryModel(...)` generator, but their consumption model differs:

- streaming yields stream events plus assistant messages
- non-streaming consumes the generator to completion and returns the final assistant message

That means non-streaming is not a separate business-logic implementation. It is a different consumption wrapper over the same main pipeline.

## The Main Query Pipeline Does Significant Pre-Request Work

Inside `queryModel(...)`, the API call is only one step in a larger preparation flow.

Verified pre-request work includes:

- checking the Opus emergency off-switch for PAYG users
- deriving the previous request ID from the current conversation chain
- resolving Bedrock inference-profile backing models for cost accounting
- computing the merged beta set
- enabling and validating the advisor server tool
- enabling or disabling tool search
- filtering deferred tools based on discovery state
- building API tool schemas
- normalizing messages for the API
- repairing tool-result pairing
- stripping advisor blocks when the beta is absent
- stripping excess media items before request send
- building the final system prompt blocks
- computing prompt-caching and cache-break-detection state

So the transport layer is tightly coupled to request shaping and conversation normalization, not only network I/O.

## Tool Search And Deferred Tool Loading Are Runtime Request Features

The tool-search path in `queryModel(...)` is worth calling out because it changes what goes on the wire.

Verified behavior:

- tool search is enabled only when the selected model and runtime conditions support it
- deferred tools are not all sent to the API immediately
- once a deferred tool has been discovered through tool references, it becomes eligible to be included in later requests
- ToolSearch-specific beta headers vary by provider
- Bedrock receives tool-search beta data through extra body params rather than the regular `betas` array

So tool search is not only a tool registry concern. It directly affects request payload composition.

## Message Normalization Is Followed By Model-Specific Cleanup

The API pipeline does not assume `normalizeMessagesForAPI(...)` is the final word.

After normalization, `queryModel(...)` may still:

- strip `tool_reference` blocks from user tool results
- strip `caller` fields from assistant tool-use blocks
- repair missing or orphaned tool-result relationships
- strip excess images/documents to stay under API limits

This second-stage cleanup exists because some model/provider capabilities are not stable across model switches or conversation history reuse.

So there is a distinction between:

- generic normalization
- model-aware post-normalization cleanup

## Prompt Caching Is Integrated Into Request Construction, Not Bolted On Later

Prompt caching in `services/api/claude.ts` affects both system blocks and message blocks.

Verified behavior:

- cache eligibility is controlled by model, query source, provider, and feature settings
- `buildSystemPromptBlocks(...)` can add cache markers to system prompt blocks
- `addCacheBreakpoints(...)` adds exactly one message-level cache marker per request
- skip-cache-write mode shifts the marker to avoid polluting the prompt cache for fire-and-forget forks
- tool-result blocks inside the cached prefix can get `cache_reference`
- cached microcompact edits can inject `cache_edits` blocks into user messages

This is not a simple boolean flag.

The request builder is carefully deciding:

- where cache markers go
- which content participates in cache reuse
- when cache-deletion edits should travel with the request

## Sticky Beta-Header Latching Exists To Protect Cache Stability

One of the most important transport details is the sticky-on beta-header behavior in `queryModel(...)`.

Verified behavior:

- AFK mode header is latched once first activated
- fast-mode header is latched once first used
- cache-editing header is latched once enabled for the session
- thinking-clear behavior is latched based on long gaps between calls

The point is not feature convenience. The point is cache-key stability.

The code comments are explicit:

- mid-session header flips can bust large prompt caches

So some runtime headers are intentionally kept stable even when the live feature state changes afterward.

## Request Parameters Are Rebuilt Per Retry Attempt

`paramsFromContext(...)` inside `queryModel(...)` is the central request-body builder.

It recomputes request parameters from the current retry context, not only from the original options.

Verified behavior:

- beta headers can be extended dynamically
- Bedrock-specific betas are pushed into extra body params
- output config can be augmented with effort and task budget parameters
- structured outputs are enabled only when model support and beta support line up
- `max_tokens` can be overridden by retry context after overflow handling
- thinking config can switch between adaptive and explicit budgeted thinking
- context-management parameters can be included when beta-gated
- fast mode stays dynamic in `speed` even though its header is latched

So retries are not simple replays. The pipeline can materially change the outgoing body on subsequent attempts.

## Streaming Is The Primary Path, But The Code Treats It As Fragile Infrastructure

The main request path uses streaming first.

Verified behavior:

- streaming requests are made with `.create(..., { stream: true }).withResponse()`
- the raw stream is used rather than the higher-level message stream wrapper to avoid expensive partial JSON parsing behavior
- the code manually accumulates content blocks, usage, stop reasons, and partial assistant messages
- streamed content is turned into normalized assistant messages block by block

But the code also assumes streaming can fail in several ways:

- no `message_start`
- no completed content blocks
- mid-stream stalls
- watchdog-triggered no-chunk timeout
- transport-layer stream creation failure

So streaming is the preferred path, but not a trusted one.

## The Streaming Watchdog Is An Active Kill Switch

`queryModel(...)` has a stream idle watchdog separate from normal SDK request timeout behavior.

Verified behavior:

- it can be enabled via env
- it warns at half the configured idle timeout
- it aborts the stream if no chunks arrive before the timeout
- it emits dedicated telemetry for idle warnings, idle timeouts, and eventual loop exit after watchdog firing

This exists because a hung streaming body can outlive the original request timeout and otherwise stall the session indefinitely.

So stream timeout handling is not delegated entirely to the SDK.

## Non-Streaming Fallback Is A First-Class Recovery Path

When streaming fails, the code can switch into `executeNonStreamingRequest(...)`.

Verified behavior:

- non-streaming fallback has its own timeout budget
- remote sessions default to a shorter fallback timeout than local sessions
- non-streaming requests still run through `withRetry(...)`
- non-streaming params are adjusted through `adjustParamsForNonStreaming(...)`
- fallback can be disabled by env flag or GrowthBook flag
- 404-on-stream-creation has its own explicit fallback path

This is more than a backup request mode.

The code is deliberately trying to recover from gateway or provider paths that:

- break on streaming
- succeed on non-streaming
- or fail differently enough that bounded fallback is useful

## Retry Policy Is Not Just Exponential Backoff

`withRetry(...)` contains a richer policy system than "retry on 5xx".

Verified behavior:

- foreground and background query sources differ for 529 overload retries
- persistent unattended mode can keep retrying 429 and 529 with long backoff and periodic keep-alive yields
- CCR sessions treat 401 and 403 as transient infrastructure failures, not always bad credentials
- 401 and revoked-token flows can force OAuth refresh
- stale connection errors can disable keep-alive and retry
- Bedrock and Vertex auth failures clear provider-specific credential caches
- retry behavior can adapt to `retry-after` and rate-limit reset headers

So retry logic is query-source-aware, provider-aware, and environment-aware.

## Fast Mode Has Its Own Retry And Cooldown Behavior

Fast mode is deeply integrated into the retry loop.

Verified behavior:

- short `retry-after` values can keep fast mode active to preserve cache benefits
- longer or unknown retry delays can trigger a fast-mode cooldown and downgrade subsequent attempts to standard speed
- specific overage-disabled headers can permanently disable fast mode for the session
- API-side "fast mode not enabled" errors can also force a downgrade

So fast mode is not only a request parameter. It has its own resilience policy and downgrade path.

## Model Fallback Is Signaled Through A Dedicated Error

The retry loop can trigger `FallbackTriggeredError` after repeated overloads.

Verified behavior:

- it mainly applies to Opus-like primary models in the configured fallback cases
- it does not itself perform the switch
- instead it throws a dedicated error so outer orchestration can change models and retry at a higher layer

This is important architecturally:

- fallback decision happens in the API retry loop
- fallback execution happens outside that loop

So model fallback is coordinated across layers, not self-contained in `withRetry.ts`.

## Error Shaping Is User-Facing Product Logic, Not Raw SDK Wrapping

`services/api/errors.ts` translates low-level transport and API failures into assistant-facing error messages.

Verified behavior:

- prompt-too-long, media-size, PDF, and request-size failures get tailored recovery guidance
- rate-limit errors can be transformed through Claude.ai limits logic
- x-api-key failures produce different messages for external API-key users versus managed login users
- revoked OAuth and org-not-allowed OAuth errors get specific login guidance
- Bedrock 404/model-access cases get `/model`-oriented fallback suggestions
- tool-use/tool-result corruption errors get rewind-style recovery guidance
- refusal stop reasons are converted into a specific assistant error message

So this module is not only formatting errors. It embeds product-specific recovery advice and policy.

## Connection Error Handling Walks The Cause Chain

`services/api/errorUtils.ts` extracts connection error details from nested causes.

Verified behavior:

- SSL/TLS codes are recognized from the underlying error chain
- SSL failures can produce dedicated human-readable messages and hints
- HTML gateway/proxy error pages are sanitized into safer user-facing strings
- deserialized JSONL API errors without a top-level `.message` can still recover nested messages

So error formatting is built to survive:

- SDK wrapping
- proxy/gateway HTML responses
- session replay / JSON round-tripping

## Logging Is A Separate Instrumentation Layer

`services/api/logging.ts` is doing more than analytics bookkeeping.

Verified behavior:

- request logging records model, provider, betas, query source, permission mode, thinking mode, effort, and previous request ID
- success logging records token usage, duration, TTFT, stop reason, cache strategy, fast mode, and gateway fingerprints
- error logging records classified error type, status, duration, request IDs, provider, and gateway fingerprints
- both success and error paths feed OTLP-style telemetry events
- request spans are closed with explicit success/error metadata

This means API logging is a first-class runtime subsystem, not an afterthought.

## Prompt Cache Break Detection Is A Parallel Diagnostic Subsystem

`services/api/promptCacheBreakDetection.ts` tracks why server-side prompt cache reads dropped between calls.

Verified behavior:

- it records a pre-call snapshot of system prompt, tool schemas, model, betas, cache strategy, effort, extra body, and related state
- after the response it compares cache-read tokens against the previous baseline
- it tries to explain breaks in terms of model changes, system prompt changes, tool changes, beta changes, cache-control changes, and TTL expiry windows
- it can write diff files for debugging
- cache deletion and compaction events reset or suppress the normal break baseline

This is not part of request correctness, but it is part of the transport-runtime architecture because it tracks the hidden server-side behavior the client is trying to optimize.

## Bootstrap Is A Separate Lightweight API Path

`services/api/bootstrap.ts` is not part of the main query loop.

Verified behavior:

- it only runs for the first-party provider
- it is skipped in essential-traffic-only mode
- it prefers OAuth with profile scope, but can fall back to API key auth
- it uses a small dedicated REST call rather than the main Anthropic messages SDK
- it persists client data and additional model options into global config only when the response changed

So bootstrap is a side-channel configuration fetch, not a normal chat request.

## Transport Runtime Also Owns Some Memory-Leak And Cleanup Defenses

The streaming path explicitly cleans up stream resources.

Verified behavior:

- `cleanupStream(...)` aborts live streams when needed
- response bodies are canceled explicitly
- cleanup is repeated in `finally` paths to survive partial generator consumption or early return
- fallback cost and usage bookkeeping is deferred carefully so early generator termination does not lose accounting

This is an architectural signal that the API layer is managing native/network resource lifecycles, not only JavaScript objects.

## Corrections To The Existing Mental Model

- The API subsystem is not one function call to an SDK. It is a staged runtime composed of client selection, request shaping, retry/fallback logic, and response/error normalization.
- Streaming is the primary path, but the code treats it as failure-prone infrastructure and has explicit watchdog, empty-stream, and 404 fallback handling.
- Retries are not generic exponential backoff. They depend on query source, provider, fast mode, CCR mode, retry headers, and even model-fallback policy.
- Prompt caching is not a single toggle. Cache markers, sticky beta headers, cache edits, and cache-break diagnostics are all part of request construction.
- Error handling is not just formatting raw messages. It is a product-layer translation system that maps low-level failures into user-recovery flows.
- The transport layer owns resource cleanup, telemetry, and cache diagnostics in addition to sending requests.
