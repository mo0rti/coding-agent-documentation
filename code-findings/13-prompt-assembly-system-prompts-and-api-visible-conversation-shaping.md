# Prompt Assembly, System Prompts, And API-Visible Conversation Shaping

## Scope

This document covers how Claude Code builds the system prompt, how session/runtime overlays modify it, how attachments and transcript messages are converted into model-visible turns, and how the API layer performs final normalization and cache-aware request shaping before send.

Primary source files:

- `constants/prompts.ts`
- `constants/system.ts`
- `constants/systemPromptSections.ts`
- `utils/systemPrompt.ts`
- `utils/systemPromptType.ts`
- `utils/messages.ts`
- `utils/api.ts`
- `utils/attachments.ts`
- `utils/toolSearch.ts`
- `utils/mcpInstructionsDelta.ts`
- `utils/fingerprint.ts`
- `services/api/claude.ts`
- `services/api/promptCacheBreakDetection.ts`
- `utils/analyzeContext.ts`

## The Key Architectural Split

The code does not have one single "prompt".

There are five separate layers:

1. the baseline system prompt assembled from `constants/prompts.ts`
2. the effective system prompt overlay logic in `utils/systemPrompt.ts`
3. attachment rendering that injects model-visible meta user messages
4. transcript normalization and repair in `utils/messages.ts`
5. final API request shaping in `services/api/claude.ts` and `utils/api.ts`

That split matters because a lot of model-visible context never lives in the raw `systemPrompt: string[]` array.

For example:

- MCP instructions may be in the system prompt or in persisted delta attachments
- queued commands and mailbox events become synthetic user messages
- local command system messages are converted into user turns
- malformed resumed transcripts are repaired immediately before send

So the real source of truth is not "the system prompt text". It is the whole request-shaping pipeline.

## The Baseline System Prompt Is Built As Static And Dynamic Sections

`getSystemPrompt(...)` in `constants/prompts.ts` is the main baseline builder.

Verified behavior:

- a simple mode exists behind `CLAUDE_CODE_SIMPLE`
- the normal path builds a long structured prompt from many section helpers
- static sections include identity, tool usage guidance, task execution guidance, tone/style, and output-efficiency instructions
- dynamic sections include session-specific guidance, memory prompt content, language/output-style sections, scratchpad guidance, tool-result-clearing guidance, and MCP instructions

The code intentionally separates these for cache behavior, not just readability.

## The Boundary Marker Is A Cache-Control Device, Not A Human-Facing Prompt Feature

`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` in `constants/prompts.ts` is load-bearing.

Verified behavior:

- when global prompt cache scope is enabled, `getSystemPrompt(...)` inserts the boundary between static and dynamic sections
- `splitSysPromptPrefix(...)` in `utils/api.ts` later uses that marker to split the system prompt into cacheable blocks
- static pre-boundary content can be given `cacheScope: 'global'`
- post-boundary dynamic content stays uncached
- if MCP tools force tool-based cache behavior, the boundary is ignored and the prompt falls back to org-scoped grouping

So this marker is part of the transport/cache architecture, not just documentation structure inside the prompt.

## System Prompt Sections Are Registry-Managed And Cached Across Turns

`constants/systemPromptSections.ts` defines the section model.

Verified behavior:

- `systemPromptSection(...)` memoizes a named section until `/clear` or `/compact`
- `DANGEROUS_uncachedSystemPromptSection(...)` always recomputes and is explicitly marked as cache-breaking
- `resolveSystemPromptSections(...)` reads and writes a shared section cache in bootstrap state
- `clearSystemPromptSections()` also clears sticky beta-header latches

The important practical consequence is that system-prompt variability is centrally managed. The code is trying to minimize accidental prompt-cache churn.

## MCP Instructions Are Special Because They Can Escape The System Prompt Entirely

`getSystemPrompt(...)` treats MCP instructions differently from most other sections.

Verified behavior:

- MCP instructions can be emitted as a `DANGEROUS_uncachedSystemPromptSection`
- when `isMcpInstructionsDeltaEnabled()` is true, that section returns `null`
- in that mode, late-arriving MCP instructions are announced via persisted `mcp_instructions_delta` attachments instead of per-turn system-prompt recomputation

This is an important correction to the common mental model:

- MCP instructions are not always part of the system prompt
- sometimes they live in the conversation history as synthetic meta user messages to preserve cache stability

## The Effective System Prompt Is A Separate Overlay Step

`buildEffectiveSystemPrompt(...)` in `utils/systemPrompt.ts` runs after the baseline prompt is built.

Verified precedence:

1. `overrideSystemPrompt` replaces everything except later append suppression rules
2. coordinator prompt wins when coordinator mode is active and there is no main-thread agent definition
3. main-thread agent prompt can replace or extend the default prompt
4. `customSystemPrompt` replaces the default prompt
5. `appendSystemPrompt` is appended unless `overrideSystemPrompt` was used

The agent path has special behavior:

- built-in agents can compute a prompt from tool-use context
- other agents use `getSystemPrompt()` from their definition
- proactive/Kairos mode appends agent instructions under `# Custom Agent Instructions` instead of replacing the default prompt

So "agent prompt", "custom prompt", and "system prompt override" are three distinct mechanisms.

## System Prompt Identity Prefix And Attribution Header Are Added Very Late

The final system prompt sent to the API is not just the output of `buildEffectiveSystemPrompt(...)`.

Inside `services/api/claude.ts`, `queryModel(...)` prepends:

- `getAttributionHeader(fingerprint)`
- `getCLISyspromptPrefix(...)`

Verified behavior:

- the CLI prefix changes for interactive vs non-interactive sessions
- Agent SDK wording differs depending on whether the caller also appended a custom system prompt
- Vertex always uses the default CLI prefix
- the attribution header can include version, entrypoint, optional attestation placeholder, and optional workload hint
- the fingerprint is computed from the first normalized user message before synthetic deferred-tool announcements are injected

So the "true" system prompt body is finalized in the API layer, not entirely in `constants/prompts.ts`.

## Attachments Are A Major Prompt Surface, Not Just Transcript Decorations

`normalizeAttachmentForAPI(...)` in `utils/messages.ts` converts attachments into `UserMessage[]`.

This is one of the most important architectural facts in the codebase.

Verified behavior:

- many attachment types are wrapped in `<system-reminder>` user messages
- some attachments are rendered as synthetic tool-use plus tool-result pairs to simulate prior reads
- others are intentionally dropped from API context and kept only for UI/runtime use

Examples of model-visible attachment content:

- file and directory context
- selected IDE lines and opened-file reminders
- relevant memories and nested memory files
- skill listings and skill discovery suggestions
- queued command/user-interruption text
- diagnostics, plan-mode reminders, auto-mode reminders, and critical reminders
- MCP resource contents
- task-status updates
- deferred-tools and MCP-instructions deltas
- teammate mailbox and team coordination context

Examples intentionally excluded from API context:

- `already_read_file`
- `command_permissions`
- several hook bookkeeping attachments
- dynamic-skill bookkeeping

So the conversation the model sees is materially richer than the visible user transcript.

## Attachment Rendering Often Encodes Policy And Behavioral Nudges

Many attachment renderers are not neutral serialization.

Verified examples:

- `queued_command` text is rewritten through `wrapCommandText(...)` to mark provenance and urgency
- `pdf_reference` explicitly instructs the model to use page-bounded reads
- running-task attachments warn not to spawn duplicates
- relevant-memory content uses stable precomputed headers to avoid prompt-cache churn
- `deferred_tools_delta` and `mcp_instructions_delta` describe additions and removals as historical conversation state rather than ephemeral system state

So attachment rendering is product behavior, not only format conversion.

## Transcript Normalization Removes UI-Only And Invalid API Shapes

`normalizeMessagesForAPI(...)` in `utils/messages.ts` is the main transcript-to-wire converter.

Verified behavior:

- attachment messages are reordered upward before normalization
- virtual messages are stripped
- progress messages are stripped
- non-local-command system messages are stripped
- local command system messages are converted into user messages so the model can refer back to them
- synthetic API error messages are removed after being used to decide targeted media stripping

This means the internal transcript and the wire transcript are intentionally different data shapes.

## Consecutive User And Assistant Messages Are Merged Aggressively

The API-shaping layer does not preserve raw message boundaries.

Verified behavior:

- consecutive user messages are merged because some providers do not support multiple adjacent user turns
- user-message text seams are joined with inserted newlines to avoid accidental concatenation
- tool-result blocks are hoisted to the front of merged user content
- assistant messages with the same `message.id` are merged even across interleaved tool-result messages

This is especially important for resumed, streamed, or interleaved conversations:

- the stored transcript may contain many small fragments
- the model-facing payload is reassembled into fewer, larger turns

## Tool-Reference And Tool-Result Handling Is A Distinct Subsystem

Several normalization passes exist specifically for ToolSearch and tool-result correctness.

Verified behavior:

- tool-reference blocks are stripped when tool search is not enabled
- when tool search is enabled, references to unavailable tools are selectively stripped
- assistant tool-use inputs are normalized per tool schema
- `caller` fields are preserved only when tool search is enabled
- tool-reference tail handling may inject or relocate turn-boundary text to avoid bad completion patterns
- system-reminder siblings can be folded into tool-result content to avoid problematic prompt shapes

This is not incidental cleanup. It is an entire conversation-shaping subsystem designed around specific API and model behaviors.

## Normalization Also Repairs Persisted-Transcript Corruption

The code expects resumed or compacted transcripts to sometimes be structurally invalid.

Verified recovery passes include:

- filtering orphaned thinking-only assistant messages
- stripping trailing thinking from the last assistant turn
- filtering whitespace-only assistant messages
- ensuring assistant messages are never empty after cleanup
- sanitizing `is_error` tool results so they contain only text blocks
- stripping signature-bearing blocks after credential changes with `stripSignatureBlocks(...)`

The code comments are explicit that many of these exist because older persisted sessions can otherwise get stuck in permanent API-error loops.

## Tool Use And Tool Result Pairing Is Repaired Right Before Send

`ensureToolResultPairing(...)` is called from `queryModel(...)` after normalization.

Verified behavior:

- missing tool results cause synthetic error `tool_result` placeholders to be inserted
- orphaned tool results are stripped
- duplicate `tool_use` IDs across assistant messages are deduped
- orphaned server-side tool uses are stripped
- empty repaired assistant messages get placeholders instead of being left invalid
- strict mode can throw instead of repairing

This means the API layer does not trust the stored transcript to already obey tool-use/tool-result invariants.

## Advisor And Tool-Search Cleanup Also Happens After Generic Normalization

`queryModel(...)` performs a second cleanup pass after `normalizeMessagesForAPI(...)`.

Verified behavior:

- model-aware tool-search disabling can still strip tool-reference blocks and `caller` fields after generic normalization
- advisor blocks are stripped unless the advisor beta header is present
- excess media items are dropped before request send

So `normalizeMessagesForAPI(...)` is not the final authority. The API layer still applies model-aware cleanup immediately before serialization.

## System Prompt Blocks And Message Cache Markers Are Built Separately

The request builder uses two different cache-marker paths:

- `buildSystemPromptBlocks(...)` for system prompt text blocks
- `addCacheBreakpoints(...)` for message history

Verified behavior:

- `buildSystemPromptBlocks(...)` delegates to `splitSysPromptPrefix(...)`
- only non-null cache scopes receive `cache_control`
- `addCacheBreakpoints(...)` places exactly one message-level cache marker per request
- `skipCacheWrite` shifts that marker backward for fire-and-forget fork-style requests
- cached-microcompact deletions are inserted as `cache_edits`
- prior `cache_edits` can be pinned and replayed at stable positions
- tool-result blocks before the last cache marker can get `cache_reference`

So prompt caching is not layered on after prompt assembly. It directly shapes how both the system prompt and the message list are serialized.

## Prompt Cache Diagnostics Treat Prompt Shape As First-Class State

`services/api/promptCacheBreakDetection.ts` confirms that prompt structure is an explicitly tracked runtime concern.

Verified behavior:

- it hashes system blocks with and without `cache_control`
- it tracks tool-schema changes separately from system-prompt changes
- it treats beta changes, model changes, fast mode, overage mode, effort, and extra body params as cache-break inputs

That subsystem reinforces an important architectural point:

- prompt assembly in this repo is designed around cache stability as much as around instruction quality

## Context Analysis Uses The Same Normalized Prompt View

`utils/analyzeContext.ts` reuses the same core builders:

- `buildEffectiveSystemPrompt(...)`
- `normalizeMessagesForAPI(...)`

This matters because token accounting is trying to estimate the same shape the live API path will see, not a simpler approximation based only on the raw transcript.

## Corrections To The Existing Mental Model

- The model-visible prompt is not just the `systemPrompt` array. It is the combination of system prompt, attachment rendering, transcript normalization, and API-side repair.
- The baseline prompt and the effective prompt are different layers. Agent/coordinator/custom/override behavior is applied after the base prompt is built.
- MCP instructions are not always in the system prompt. In delta mode, they become persisted conversation attachments instead.
- Attachments are not UI sugar. Many of them are primary prompt inputs that materially steer model behavior.
- The wire transcript is not a faithful copy of the stored transcript. User messages merge, assistant fragments merge, virtual/progress/system messages are stripped, and invalid shapes are repaired.
- Cache stability is a first-class design constraint. The dynamic boundary, section registry, delta attachments, sticky headers, and cache-break detection all exist to keep prompt bytes stable across turns.
