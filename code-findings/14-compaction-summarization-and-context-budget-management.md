# Compaction, Summarization, And Context-Budget Management

## Scope

This document covers how Claude Code measures context pressure, applies pre-request context shrinking, decides when to compact, runs full and partial compaction flows, preserves critical state across boundaries, and threads token-target budgeting through those transitions.

Primary source files:

- `query.ts`
- `commands/compact/compact.ts`
- `services/compact/autoCompact.ts`
- `services/compact/compact.ts`
- `services/compact/microCompact.ts`
- `services/compact/sessionMemoryCompact.ts`
- `services/compact/prompt.ts`
- `services/compact/postCompactCleanup.ts`
- `services/compact/grouping.ts`
- `services/compact/timeBasedMCConfig.ts`
- `services/compact/apiMicrocompact.ts`
- `utils/toolResultStorage.ts`
- `utils/tokens.ts`
- `utils/analyzeContext.ts`
- `utils/messages.ts`
- `utils/sessionStorage.ts`
- `utils/tokenBudget.ts`

## Scope Caveat For This Workspace Snapshot

Two compaction-related modules are referenced from code but are not present in this workspace snapshot:

- `services/compact/reactiveCompact.js`
- `services/compact/cachedMicrocompact.js`

So this document only treats their orchestration and contracts as verified:

- what `query.ts`, `commands/compact/compact.ts`, and `microCompact.ts` assume about them
- where they are called
- what state or results they are expected to return

Their internal implementation is not documented here as source-verified.

## The Key Architectural Split

The code does not have one generic "compaction system".

There are at least seven separate context-management layers:

1. per-message tool-result budget enforcement
2. history snip
3. microcompact
4. context collapse projection
5. proactive auto-compact
6. reactive compact/retry recovery
7. token-target budgeting for long-running tasks

Those layers are deliberately ordered and are not interchangeable.

The important correction to the common mental model is:

- "compaction" in this codebase means several cooperating strategies, not only `/compact`

## Query-Time Context Shrinking Happens In A Fixed Order

The main query loop in `query.ts` applies context management in a specific sequence before the next API request:

1. `applyToolResultBudget(...)`
2. history snip
3. `microcompactMessages(...)`
4. context-collapse projection
5. `autoCompactIfNeeded(...)`
6. model request
7. reactive overflow recovery if the request still fails

That ordering is load-bearing.

Verified examples:

- per-message tool-result replacement runs before microcompact because cached microcompact only reasons about `tool_use_id`, not content
- snip runs before autocompact, and the freed-token estimate is threaded into threshold checks because API usage alone cannot see the savings yet
- context collapse runs before autocompact so it can keep granular context and avoid collapsing into a single summary unnecessarily

So the runtime is trying cheaper, more local reductions before paying the cost of full conversation summarization.

## Token Counting Uses A Hybrid Of Real Usage And Rough Estimation

`utils/tokens.ts` contains the canonical context-size helpers.

Verified behavior:

- `tokenCountWithEstimation(...)` uses the last real assistant usage record plus rough estimates for messages added since
- when parallel tool calls create multiple assistant records with the same `message.id`, it walks back to the first sibling so interleaved tool results are included
- `tokenCountFromLastAPIResponse(...)` measures the last full context size from API usage
- `finalContextTokensFromLastResponse(...)` is used for task-budget carryover and intentionally excludes cache tokens

This matters because compaction thresholds are not based on cumulative spend. They are based on estimated next-request context size.

## The Effective Context Window Is Smaller Than The Model Window

`services/compact/autoCompact.ts` defines the effective context model.

Verified behavior:

- `getEffectiveContextWindowSize(model)` subtracts a reserved summary-output budget from the model context window
- the reserved amount is the smaller of model max output tokens and a 20k summary cap
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` can cap the usable context lower for testing or forced behavior

So all thresholding is based on a reduced "safe working window", not the raw provider-advertised context length.

## Auto-Compact Thresholds Are Buffered, Not Exact

The main constants in `autoCompact.ts` are:

- `AUTOCOMPACT_BUFFER_TOKENS = 13_000`
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`
- `ERROR_THRESHOLD_BUFFER_TOKENS = 20_000`
- `MANUAL_COMPACT_BUFFER_TOKENS = 3_000`

Verified behavior:

- `getAutoCompactThreshold(model)` returns effective window minus the autocompact buffer
- `calculateTokenWarningState(...)` computes warning/error/autocompact/blocking conditions separately
- when auto-compact is off, a manual compact reserve is still held back so the user can recover with `/compact`
- blocking-limit testing can be overridden by env

So the CLI is not waiting until the model window is actually full.

## Auto-Compact Can Be Disabled Or Suppressed By Several Higher-Level Modes

`shouldAutoCompact(...)` has more gates than just "usage above threshold".

Verified behavior:

- `DISABLE_COMPACT` disables all compact behavior here
- `DISABLE_AUTO_COMPACT` disables only automatic compaction
- user config `autoCompactEnabled` also gates it
- `session_memory` and `compact` query sources are recursion guards
- main-thread context-collapse mode suppresses proactive autocompact
- reactive-only mode suppresses proactive autocompact
- `marble_origami` is excluded to avoid corrupting shared context-collapse state

So auto-compact is one strategy among several, and the runtime deliberately stands down when another context-management mode owns the problem.

## Auto-Compact Has A Circuit Breaker

`autoCompactIfNeeded(...)` keeps track of consecutive failures.

Verified behavior:

- after three consecutive autocompact failures, the session stops retrying automatic compaction
- successful compact resets the failure count
- failures are still logged, but the system avoids infinite "doomed compact attempt every turn" loops

That is an important resilience detail for irrecoverably oversize sessions.

## Session-Memory Compact Is Tried Before Full Summary Compact

When auto-compact triggers, the first compact attempt is `trySessionMemoryCompaction(...)`.

Verified behavior:

- it only runs when both session-memory and session-memory-compact feature gates are active, unless env overrides force it
- it waits for any in-progress memory extraction to finish
- it refuses to run if session memory is missing or still just the template
- it can be used both in normal sessions and resumed sessions where no explicit summarized-message boundary is known

If it succeeds:

- it returns a normal `CompactionResult`
- `lastSummarizedMessageId` is reset
- post-compact cleanup runs
- prompt-cache-break detection is notified

So session-memory compact is not a side feature. It is the preferred low-loss autocompact path when available.

## Session-Memory Compact Preserves A Tail Instead Of Replacing Everything

`services/compact/sessionMemoryCompact.ts` makes the session-memory strategy explicit.

Verified behavior:

- it calculates a `messagesToKeep` start index based on minimum token and minimum text-message thresholds
- it expands backward until minimums are met, but stops at the last compact boundary floor
- it adjusts the start index to preserve tool-use/tool-result pairs and split assistant-message fragments with the same `message.id`
- it builds a compact boundary plus one summary message from session memory
- it preserves a recent tail of live messages after the summary

This is a major architectural difference from legacy full compact:

- session-memory compact is suffix-preserving
- full compact replaces nearly the whole conversational working set with a summary and re-injected attachments

## Full Compact Is A Summarization Workflow With Hooks, Retry Logic, And State Rehydration

`compactConversation(...)` in `services/compact/compact.ts` is much more than "ask the model for a summary".

Verified stages:

1. validate there are messages to compact
2. compute pre-compact token count
3. run PreCompact hooks
4. build the compaction prompt
5. request a summary through either cache-sharing fork or streaming fallback
6. retry if the compact request itself hits prompt-too-long
7. clear read-file and nested-memory caches
8. rebuild critical attachments and hook state for the post-compact turn
9. create a compact boundary and summary message
10. run SessionStart hooks for the new compacted session state
11. log compaction metrics and notify cache diagnostics
12. re-append session metadata to keep it inside the tail-read window
13. run PostCompact hooks

So compaction is an orchestration workflow, not a single API request.

## The Summarizer Is Explicitly Forbidden From Using Tools

`services/compact/prompt.ts` defines the summarization prompt.

Verified behavior:

- the prompt begins with a strong `TEXT ONLY` / no-tools preamble
- the summary must be plain text containing `<analysis>` and `<summary>` sections
- after the response, `formatCompactSummary(...)` strips the analysis scratchpad and reformats the summary section

The reason is architectural:

- compaction may run through a fork that inherits the main thread's tool schema for cache-key matching
- without the explicit no-tools instructions, tool-use would waste the summarizer's one turn and force fallback behavior

## Full Compact Can Reuse The Main Conversation’s Prompt Cache

`streamCompactSummary(...)` in `compact.ts` first tries a cache-sharing fork path.

Verified behavior:

- it can call `runForkedAgent(...)` with `skipCacheWrite: true`
- it avoids setting `maxOutputTokensOverride` in that path to keep the cache key identical to the parent conversation
- if the cache-sharing path fails or returns unusable output, it falls back to a regular streaming request

This means compaction is optimized as a transport/cache problem, not just a summarization problem.

## Compaction Retries Prompt-Too-Long By Dropping Old API-Round Groups

If the compaction request itself overflows, `truncateHeadForPTLRetry(...)` is used.

Verified behavior:

- messages are grouped by API round, not by human turn, using `groupMessagesByApiRound(...)`
- the oldest groups are dropped until the token gap is covered, or a fallback percentage is used when the gap is unknown
- a synthetic user marker may be prepended if truncation would otherwise leave an assistant-first payload

This is intentionally lossy, but it prevents the user from being permanently stuck when even the compact request is too large.

## Full Compact Rebuilds Essential Working Context After Summarizing

After a successful summary, `compactConversation(...)` rebuilds several attachment layers.

Verified post-compact rehydration includes:

- recently read files, under file-count and token budgets
- async/background agent state
- plan file reference
- plan-mode reminder when relevant
- invoked skill contents
- deferred-tools delta
- agent-listing delta
- MCP-instructions delta
- SessionStart hook messages

This is one of the most important architectural details:

- compaction is not just shrinking history
- it is also reconstructing a minimal working environment so the next turn can continue immediately

## Post-Compact File Restoration Is Budgeted And Deduplicated

`createPostCompactFileAttachments(...)` restores recently accessed files carefully.

Verified behavior:

- recently read files are selected from `readFileState`
- plan files and Claude memory files are excluded
- files already visible in preserved messages are skipped
- each file is re-read through `generateFileAttachment(...)`
- both a max-files cap and a token budget cap are enforced

So post-compact file restoration is selective and fresh, not a blind replay of old attachment messages.

## Compact Boundaries Carry More Than “A Summary Happened”

`createCompactBoundaryMessage(...)` and `annotateBoundaryWithPreservedSegment(...)` make compact boundaries structural.

Verified boundary metadata can include:

- trigger type
- pre-compact token count
- custom user context
- message count summarized
- discovered deferred-tool names carried across the boundary
- preserved-segment relink metadata

That metadata is then used by transcript loading and tool-search reconstruction.

So compact boundaries are part of transcript topology, not just informational markers.

## Transcript Loading Reconstructs Preserved Segments After Compaction

`utils/sessionStorage.ts` contains important compaction-specific resume logic.

Verified behavior:

- `applyPreservedSegmentRelinks(...)` patches parent links for preserved tails after compaction
- it validates the preserved chain before mutating
- it zeros stale assistant usage values on preserved messages so resumed sessions do not immediately re-trigger autocompact from pre-compact usage
- it prunes pre-boundary content that should no longer participate in resume reconstruction

This is a key correction to a simplistic model:

- compaction does not just append a summary message to the transcript
- it changes how transcript chains must be interpreted on load

## `getMessagesAfterCompactBoundary(...)` Is The Standard “Live Context” Slice

The code repeatedly uses `getMessagesAfterCompactBoundary(...)` as the live working-history view.

Verified behavior:

- if a boundary exists, only the boundary and following messages are returned
- snipped messages are also filtered by default
- the boundary itself is kept in the internal slice even though later API normalization removes it

That means many model-facing paths do not look at the full stored transcript. They work from the post-boundary view.

## Microcompact Is Separate From Full Compact And Can Be Purely Pre-Request

`microcompactMessages(...)` in `services/compact/microCompact.ts` is not a small version of summary compaction.

Verified behavior:

- it clears compact-warning suppression state at the start of each attempt
- it can run a time-based content-clearing path before the request
- it can run a cached microcompact path for supported main-thread scenarios
- otherwise it may no-op entirely

The important distinction is:

- microcompact tries to keep the conversation structurally intact
- full compact rewrites conversation structure around a summary boundary

## Time-Based Microcompact Clears Old Tool Results After Long Idle Gaps

`evaluateTimeBasedTrigger(...)` and `maybeTimeBasedMicrocompact(...)` implement the idle-gap strategy.

Verified behavior:

- it only runs for explicit main-thread query sources
- it compares the time since the last assistant message against a configurable minute threshold
- if triggered, it content-clears all but the most recent N compactable tool results
- it records estimated tokens saved
- it resets cached microcompact state and notifies cache-break detection because prompt bytes changed

The design assumption is explicit in code comments:

- after a long idle gap the server prompt cache is cold anyway, so clearing old tool results before the next request reduces what must be rewritten

## Cached Microcompact Uses API Cache Editing Instead Of Mutating Local Messages

`microCompact.ts` documents the cached microcompact contract even though the backing `cachedMicrocompact.js` source is not present here.

Verified call-site behavior:

- it is main-thread only
- it requires the feature gate and a supported model
- it registers tool results by `tool_use_id`
- it can queue pending `cache_edits` for the next API call
- it does not directly rewrite local message content
- a microcompact boundary message is emitted only after the next API response, using actual `cache_deleted_input_tokens`

So cached microcompact is an API-assisted cache-management path, not a local transcript rewrite.

## API-Assisted Context Management Exists Separately From Client-Side Compact

`services/compact/apiMicrocompact.ts` builds `context_management` edits for the API.

Verified behavior:

- it can request clearing tool results, tool uses, or thinking blocks through API-native context-management edits
- these strategies are gated separately from the client-side compact flows
- some edits are ant-only

This is another reminder that "context management" in this repo spans both client-side transcript transforms and server-side edit directives.

## Per-Message Tool-Result Budget Enforcement Is Yet Another Layer

`utils/toolResultStorage.ts` applies a separate aggregate budget to tool-result content.

Verified behavior:

- it groups tool results the way `normalizeMessagesForAPI(...)` will group adjacent user messages
- when a grouped user turn exceeds the per-message budget, large fresh tool results are persisted and replaced with previews
- prior replacement decisions are frozen by `tool_use_id` so the same bytes are re-applied on later turns
- new replacement records can be written to transcript storage for resume correctness

This is not full compaction. It is an earlier content-replacement layer that prevents oversized tool-result payloads from accumulating in the first place.

## Manual `/compact` Is Not Just The Same As Auto-Compact

`commands/compact/compact.ts` shows the manual command path.

Verified behavior:

- it first slices to the post-boundary live context
- if there are no custom instructions, it tries session-memory compact first
- if reactive-only mode is enabled, it routes through the reactive path instead of legacy full compact
- otherwise it runs microcompact first, then full compact

So the manual command is an orchestrator over multiple compact strategies, not merely "force the auto-compact code now".

## Partial Compact Exists In Two Different Structural Modes

`partialCompactConversation(...)` in `compact.ts` has two directions:

- `'from'`: summarize the tail after a pivot and keep the prefix
- `'up_to'`: summarize the prefix before a pivot and keep the suffix

Verified behavior:

- the two directions have different cache implications
- `'up_to'` strips stale compact boundaries from the kept suffix to avoid the older boundary winning later scans
- summary metadata is attached differently depending on whether messages are kept
- preserved-segment anchoring differs by direction

So partial compact is not one mode. It has distinct prefix-preserving and suffix-preserving variants.

## Reactive Compact Is A Recovery Path, Not The Same As Proactive Auto-Compact

`query.ts` and `commands/compact/compact.ts` show how reactive compact is integrated.

Verified call-site behavior:

- prompt-too-long and some media-size errors can be withheld instead of being surfaced immediately
- context collapse gets a first chance to recover from overflow
- if that does not solve the overflow, reactive compact may run and then retry the turn with compacted messages
- a guard prevents repeated reactive compact spirals

But the actual `reactiveCompact.js` source is absent from this snapshot, so only that orchestration contract is verified here.

## Post-Compact Cleanup Centralizes Shared-State Reset

`runPostCompactCleanup(...)` in `services/compact/postCompactCleanup.ts` is a dedicated cleanup phase.

Verified behavior:

- microcompact state is reset
- system-prompt sections are cleared
- classifier approvals and speculative checks are cleared
- session message cache is cleared
- beta tracing state is cleared
- some resets are limited to main-thread compacts so subagents do not clobber shared module state

It intentionally does not reset invoked skills, because compact paths may still need to preserve skill content in later attachments.

## The UI Context Meter Is A Projection, Not The Same As Live Threshold Logic

`utils/analyzeContext.ts` builds the `/context` breakdown and related visualizations.

Verified behavior:

- it separates system prompt, tools, MCP tools, custom agents, memory files, skills, and messages into categories
- deferred tools are shown but excluded from actual usage
- reserved autocompact/manual buffers may be shown as explicit categories
- under reactive-only or context-collapse modes, that reserved buffer is intentionally omitted so the visualization does not lie

So the context display is an analysis view layered on top of the live runtime rules, not the runtime rule itself.

## Token-Target Budgeting Is Separate From Context Pressure Management

`utils/tokenBudget.ts`, `query.ts`, and `services/api/claude.ts` define a different mechanism: task budgets.

Verified behavior:

- user text can be parsed for token targets like `+500k` or `spend 2M tokens`
- API requests can include `output_config.task_budget`
- after a proactive or reactive compaction, `query.ts` subtracts the pre-compact final context window from the remaining task budget
- continuation messages tell the model to keep working rather than summarize when it stops at a budget threshold

This is important because "budget" here does not mean "stay under context limit". It means "keep working until this much token output/context budget has been productively used".

## Corrections To The Existing Mental Model

- Compaction is not one feature. The code combines replacement, snip, microcompact, context collapse, full compact, session-memory compact, and reactive retry recovery.
- Auto-compact is not the default owner of context pressure in every mode. It is deliberately suppressed when reactive compact or context collapse owns overflow handling.
- Session-memory compact is not a small variant of full compact. It is a distinct tail-preserving strategy built around precomputed session memory.
- Full compact is not just summary generation. It runs hooks, cache-sharing logic, PTL retry logic, post-compact rehydration, boundary creation, transcript metadata maintenance, and cleanup.
- Compact boundaries are structural transcript markers with preserved-segment metadata, not only UI messages.
- Context management also includes API-native context edits and per-message tool-result budgeting, both of which operate before or alongside full conversation summarization.
