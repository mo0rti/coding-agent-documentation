# History Snip And Context-Collapse Runtime

## Scope

This document covers the two non-summary context-management systems that sit around the main query loop: history snip and context collapse. It focuses on how they are integrated, what contracts the rest of the runtime assumes about them, how they affect model-visible history, and how their state is persisted and restored.

Primary source files:

- `query.ts`
- `commands/context/context-noninteractive.ts`
- `services/compact/autoCompact.ts`
- `services/compact/microCompact.ts`
- `services/compact/postCompactCleanup.ts`
- `utils/messages.ts`
- `utils/attachments.ts`
- `utils/sessionStorage.ts`
- `utils/sessionRestore.ts`
- `utils/conversationRecovery.ts`
- `types/logs.ts`
- `utils/analyzeContext.ts`

## Scope Caveat For This Workspace Snapshot

The core implementation modules for both subsystems are referenced from the code but are not present in this workspace snapshot:

- `services/compact/snipCompact.js`
- `services/compact/snipProjection.js`
- `services/contextCollapse/index.js`
- `services/contextCollapse/operations.js`
- `services/contextCollapse/persist.js`

So this document only treats the surrounding runtime contract as source-verified:

- where these subsystems are called
- what the rest of the code expects them to return
- how their effects are persisted, projected, and restored

Their internal algorithms are not documented here as source-verified.

## The Key Architectural Split

History snip and context collapse are not the same thing.

From the verified call sites:

- history snip is a direct message-list reduction step that runs before microcompact and before autocompact threshold checks
- context collapse is a projected-view system that summarizes spans while keeping the full REPL history in memory and replaying a commit log into model-facing views

That difference matters because they change different layers of state.

Snip changes the message array flowing through the query loop.

Context collapse, by contrast, appears to keep a richer internal history and project a collapsed view on demand.

## Query-Time Ordering Makes Their Roles Distinct

`query.ts` places these systems in a very specific order:

1. per-message tool-result budgeting
2. history snip
3. microcompact
4. context-collapse projection
5. autocompact
6. API request
7. overflow recovery, where context collapse may drain staged spans and reactive compact may run

This ordering is load-bearing.

Verified consequences:

- snip can reduce the apparent token load before autocompact is considered
- context collapse gets a chance to reduce the visible history before autocompact fires
- overflow recovery can try a context-collapse drain before escalating to reactive compact

So these are first-class runtime strategies in the main loop, not passive display features.

## History Snip Runs Before Microcompact And Feeds Its Savings Forward

`query.ts` imports `snipCompact.js` under the `HISTORY_SNIP` feature gate and calls:

- `snipCompactIfNeeded(messagesForQuery)`

Verified contract from the call site:

- it returns a new `messages` array
- it returns `tokensFreed`
- it may return a `boundaryMessage`

The query loop then:

- replaces `messagesForQuery` with the snipped messages
- threads `tokensFreed` into autocompact logic
- yields the boundary message if one exists

So history snip is not merely descriptive bookkeeping. It is an upstream context-size reduction step.

## Snip Savings Must Be Carried Separately Because Usage Data Stays Stale

The comments in `query.ts` and `autoCompact.ts` make this explicit:

- after snip, the surviving assistant usage still reflects the pre-snip context
- `tokenCountWithEstimation(...)` alone cannot see those savings yet
- `snipTokensFreed` is therefore subtracted during autocompact and blocking-limit checks

That is an important architectural fact:

- history snip changes effective context pressure before the usage accounting catches up

## History Snip Is Also A User-Facing “Context Efficiency” Guidance System

`utils/attachments.ts` shows a second contract with the snip runtime.

Verified behavior:

- `getContextEfficiencyAttachment(messages)` is only active when `HISTORY_SNIP` is enabled
- it lazily imports `isSnipRuntimeEnabled()` and `shouldNudgeForSnips(messages)`
- if both conditions pass, it emits a `context_efficiency` attachment

Then `utils/messages.ts` turns that attachment into a model-visible reminder using `SNIP_NUDGE_TEXT`.

So snip is not just automatic message removal. It also has a pacing/nudge layer that tells the model when to use the snip tooling.

## Snip Also Changes The Message Tagging Contract

`utils/messages.ts` adds `[id:...]` tags to user messages only when the snip runtime is enabled.

Verified behavior:

- tag injection is gated behind `feature('HISTORY_SNIP')`
- it further checks `isSnipRuntimeEnabled()`
- tags are appended only after all message merging is complete

The comments explain why:

- this gives the snip tool stable message references

So history snip is tied directly to API-visible message tagging, not just transcript pruning.

## Model-Facing History Is Snip-Projected By Default

`getMessagesAfterCompactBoundary(...)` in `utils/messages.ts` applies two projections:

1. slice from the latest compact boundary onward
2. filter snipped messages via `projectSnippedView(...)` by default

Verified behavior:

- the default model-facing path excludes snipped history
- callers can opt out with `includeSnipped: true`
- the REPL fullscreen compact handler is named as one example that keeps snipped messages for UI scrollback

This is another important correction:

- the REPL can retain more scrollback than the model actually sees

## Snip Persistence Is Boundary-Driven

The strongest verified snip persistence logic is in `utils/sessionStorage.ts`.

`applySnipRemovals(...)` shows that snip boundaries persist:

- a list of `removedUuids` inside `snipMetadata`

Verified restore behavior:

- removed messages are deleted from the loaded transcript map
- survivors whose `parentUuid` points into the removed gap are relinked backward through deleted-parent chains
- if there are no `removedUuids`, older boundaries are skipped and the old pre-snip history is loaded

So resume correctness depends on persisted snip-boundary metadata, not only on live in-memory filtering.

## Snip Persistence Solves A Resume-Chain Problem, Not Just A Display Problem

The comments in `applySnipRemovals(...)` are explicit about the failure mode:

- without replaying snip removals on load, `buildConversationChain(...)` reconstructs unsnipped history
- that can make resumed sessions far larger than their apparent displayed history
- deleting removed messages is not enough; parent links must also be repaired

This means history snip is part of transcript topology, not just presentation.

## Context Collapse Is A Projected-View System, Not A Simple Array Rewrite

The query-loop comments in `query.ts` give the clearest verified description.

When context collapse is enabled:

- `applyCollapsesIfNeeded(...)` returns a `messages` projection used for the next request
- nothing is yielded to the transcript at that moment
- summary messages live in the collapse store, not directly in the REPL array
- persistence comes from replaying a commit log into projected views on later turns

So the mental model is:

- full REPL history stays richer
- the model gets a projected collapsed view

That is fundamentally different from history snip.

## Context Collapse Can Suppress Proactive Auto-Compact Entirely

`services/compact/autoCompact.ts` treats context collapse as a competing owner of context pressure.

Verified behavior:

- if context collapse is enabled, `shouldAutoCompact(...)` returns `false`
- the comments explain that collapse owns the headroom strategy and autocompact would race it
- this suppression is intentionally narrower than globally disabling compact, so reactive recovery and manual `/compact` can still exist

So context collapse is not a small optimization layered on top of autocompact. It can replace proactive autocompact as the main strategy.

## Context Collapse Also Changes Overflow Recovery

The overflow path in `query.ts` shows a second major contract.

Verified behavior:

- recoverable prompt-too-long errors may be withheld instead of surfaced immediately
- `contextCollapse.isWithheldPromptTooLong(...)` can participate in that withholding decision
- on a real overflow, `contextCollapse.recoverFromOverflow(messagesForQuery, querySource)` gets the first chance to drain staged collapses
- if it commits anything, the query loop retries with the drained message set before trying reactive compact

So context collapse is not only a pre-request projection system. It also has a recovery role after overflow.

## Context Collapse Has Persisted Commit Entries And Persisted Snapshot State

`types/logs.ts` and `utils/sessionStorage.ts` expose the persistence model.

Verified persisted entry types:

- `marble-origami-commit`
- `marble-origami-snapshot`

Verified commit entry content:

- collapse ID
- summary UUID
- full summary placeholder content
- plain summary text
- first and last archived UUIDs

Verified snapshot content:

- staged spans with start/end UUIDs, summary, risk, and staged time
- armed state
- last-spawn token count

This is much richer than a single “collapse happened” marker. The runtime persists both committed collapses and live staged state.

## Context Collapse State Is Restored Before The Next Query Runs

Both interactive and CLI resume paths call into `persist.restoreFromEntries(...)` under `CONTEXT_COLLAPSE`.

Verified restore call sites:

- `restoreSessionStateFromLog(...)` in `utils/sessionRestore.ts`
- the CLI resume path later in `utils/sessionRestore.ts`

The code comments explain why this must happen early:

- `projectView()` needs the restored commit log and snapshot before the first query

So context collapse is part of session reconstruction, not a transient in-memory optimization.

## Compact Boundaries Invalidate Earlier Context-Collapse Commits

`loadTranscriptFile(...)` in `utils/sessionStorage.ts` explicitly resets stored collapse state when it encounters a compact boundary.

Verified behavior:

- on a compact boundary, accumulated `contextCollapseCommits` are cleared
- the last snapshot is cleared too

The code comment explains the reason:

- prior collapse commits reference messages that are no longer in the post-boundary chain

So context collapse and full compact are structurally related:

- a full compact cuts off the earlier collapse-log lineage

## `/context` Mirrors Collapse Projection Rather Than Raw History

`commands/context/context-noninteractive.ts` makes the user-facing context analyzer aware of context collapse.

Verified behavior:

- it starts from `getMessagesAfterCompactBoundary(messages)`
- when `CONTEXT_COLLAPSE` is enabled, it applies `projectView(...)`
- only then does it run `microcompactMessages(...)`

This is important because `/context` is trying to report what the model will actually see, not the raw full REPL transcript.

## `/context` Also Shows Collapse Runtime Status

The same command reads status from `services/contextCollapse/index.js`.

Verified call-site expectations:

- `isContextCollapseEnabled()`
- `getStats()`

The stats contract includes at least:

- `collapsedSpans`
- `collapsedMessages`
- `stagedSpans`
- `health.totalSpawns`
- `health.totalErrors`
- `health.lastError`
- `health.emptySpawnWarningEmitted`
- `health.totalEmptySpawns`

So context collapse is intended to be an observable runtime subsystem with health and staging metrics, not a silent background behavior.

## Post-Compact Cleanup Must Reset Context-Collapse State Carefully

`runPostCompactCleanup(...)` in `services/compact/postCompactCleanup.ts` treats context collapse specially.

Verified behavior:

- it calls `resetContextCollapse()` only for main-thread compacts
- it avoids doing that for subagent compacts because module-level collapse state is shared in-process

That is another architectural signal:

- context collapse appears to maintain shared module-level state, not just pure function inputs/outputs

## The UI Can Keep More History Than The Model For Both Systems

Taken together, the verified call sites show a consistent pattern:

- snip is projected out of model-facing views by default but may stay in UI scrollback
- context collapse projects summaries into model-facing views while the REPL retains richer full history plus commit-log state

So both systems separate:

- what the user can still inspect locally
- what the model sees on the next turn

But they achieve that separation differently.

## Corrections To The Existing Mental Model

- History snip is not just a tool command or transcript cosmetic. It is an upstream query-time context reduction step with persisted boundary metadata and resume-time chain repair.
- Context collapse is not the same as compact. It is a projected-view system with committed spans, staged spans, health state, and overflow-drain behavior.
- Snip physically removes middle-history segments from the model-facing chain. Context collapse appears to summarize spans while preserving a richer underlying history and replaying a commit log into projected views.
- Both systems are integrated into the model-facing view path. `/context`, resume reconstruction, autocompact suppression, and overflow recovery all know about them.
- In this snapshot, the core implementation files for both systems are missing, but their runtime contracts with the rest of the app are strong enough to document from code.
