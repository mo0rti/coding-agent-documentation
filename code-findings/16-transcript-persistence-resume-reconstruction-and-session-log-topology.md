# Transcript Persistence, Resume Reconstruction, and Session Log Topology

## Scope

This document covers the on-disk session model verified from the code:

- how transcript files are named and laid out
- what gets appended to session logs and when
- how the loader rebuilds a conversation from append-only JSONL
- how `/resume`, `--resume`, and `--continue` reconstruct more than just messages
- how lite log discovery differs from full transcript replay

Primary files read for this slice:

- `utils/sessionStorage.ts`
- `utils/sessionStoragePortable.ts`
- `utils/sessionRestore.ts`
- `utils/conversationRecovery.ts`
- `types/logs.ts`
- `utils/messages.ts`

## High-Level Model

The source of truth for session history is an append-only JSONL transcript, not a normalized database.

That JSONL stores both:

- transcript-chain entries: user, assistant, attachment, and system messages with `parentUuid`
- session-scoped metadata entries: title, tag, agent setting, mode, worktree state, PR link, summaries, file history snapshots, attribution snapshots, content replacements, and context-collapse persistence entries

The loader does not simply read the file from top to bottom and return every line. It replays the file into several maps and arrays, then rebuilds one or more conversation chains by walking `parentUuid` links from leaf nodes.

## Session File Topology

The main session transcript lives under the per-project Claude data directory as:

- `<projectDir>/<sessionId>.jsonl`

Subagent transcripts do not share that same flat file. They are stored under the session directory:

- `<projectDir>/<sessionId>/subagents/.../agent-<agentId>.jsonl`

There is also sidecar metadata for subagents:

- `agent-<agentId>.meta.json`

So the runtime has at least three distinct persistence shapes:

- main foreground session transcript
- per-agent sidechain transcripts
- sidecar metadata files for subagent launch details

The code also supports hydrating the local JSONL files from remote/session-ingress sources and from CCR v2 internal events. Even in those cases, resume still reconstructs state from local JSONL after hydration.

## Write Path

`Project` in `utils/sessionStorage.ts` is the write coordinator.

Important verified behaviors:

- Writes are queued per file and drained in batches.
- The session file is created lazily on the first user or assistant message.
- Metadata-only startup state is cached in memory first so the app does not create empty or metadata-only session files unnecessarily.
- Once the file exists, metadata is re-appended near EOF so tail-based readers can still discover it later.

`insertMessageChain(...)` stamps each persisted message with runtime context including:

- `sessionId`
- `cwd`
- `userType`
- `entrypoint`
- `version`
- `gitBranch`
- `slug`

Compact boundaries are persisted as transcript messages, but they deliberately break the main parent chain:

- `parentUuid` is set to `null`
- `logicalParentUuid` preserves the pre-break logical parent

That means the file keeps a physical append history, while the reconstructed conversation chain can intentionally restart at a compaction boundary.

## What Gets Persisted

The `Entry` union in `types/logs.ts` shows that the transcript file is not just "messages".

Verified persisted entry categories include:

- transcript messages
- `summary`
- `custom-title`
- `ai-title`
- `last-prompt`
- `task-summary`
- `tag`
- `agent-name`
- `agent-color`
- `agent-setting`
- `pr-link`
- `mode`
- `worktree-state`
- `file-history-snapshot`
- `attribution-snapshot`
- `content-replacement`
- `queue-operation`
- `speculation-accept`
- `marble-origami-commit`
- `marble-origami-snapshot`

One important consequence: resume is a replay of transcript plus attached session-state records, not just "load previous messages".

## Full Transcript Replay

`loadTranscriptFile(...)` is the core replay routine.

Its output is richer than a message array. It returns:

- a `Map<uuid, TranscriptMessage>`
- summary/title/tag/agent/mode/worktree/PR maps
- file history and attribution snapshot maps
- content replacement collections
- ordered context-collapse commits
- last-wins context-collapse snapshot
- computed leaf UUIDs

The function contains several recovery layers that matter for accuracy:

- compact-boundary-aware loading for large transcripts
- pre-boundary metadata recovery for session-scoped entries that would otherwise disappear after skipping old history
- legacy progress-bridge repair so older transcripts still chain correctly
- preserved-segment relinking after compaction
- snip-removal replay with parent relinking

This is why "just read the JSONL" is not equivalent to what the app restores.

## Large-File Loading Strategy

For larger transcripts, the loader does not naively read and parse the whole file into live state.

Verified optimizations:

- attribution snapshots are skipped at fd-read time during the large-file load path
- compact boundaries can truncate the in-memory accumulator during the forward scan
- pre-boundary metadata is recovered separately through a cheaper metadata scan
- for some large sessions, the loader pre-filters to the live chain before full parse

`utils/sessionStoragePortable.ts` provides the lower-level chunked read helpers used for this behavior.

This is an important correction to any simplified mental model that says resume "loads the whole file and picks the last session".

## Branches, Leaves, and Conversation Reconstruction

The transcript is append-only, so it can permanently contain:

- old branches from rewind/regenerate flows
- sidechains
- compacted-away history
- orphaned or partially superseded metadata

The runtime handles this by reconstructing from leaves instead of trusting file order alone.

`buildConversationChain(...)`:

- starts from a chosen leaf message
- walks `parentUuid` backward to root
- detects cycles defensively
- reverses the walk into root-to-leaf order
- then runs a post-pass to recover orphaned parallel assistant/tool-result branches

That post-pass matters because the persisted topology can be a DAG-like shape during parallel tool-use streaming, while a simple parent walk is only a linked-list traversal.

## Leaf Selection Semantics

There is not one universal "last conversation" loader.

Different callers use different leaf rules:

- `getLastSessionLog(...)` finds the most recent non-sidechain message and rebuilds that main chain
- `loadAllLogsFromSessionFile(...)` keeps all leaves and returns one `LogOption` per leaf
- `loadMessagesFromJsonlPath(...)` finds the newest non-sidechain leaf for a direct JSONL-path resume

So one session file can represent multiple branch tips, and some views intentionally expose that while normal resume paths collapse to the newest main-thread chain.

## Lite Logs Versus Full Logs

The `/resume` picker and other session listings do not eagerly load full transcripts for every session.

The code has a two-stage model:

1. stat-only session discovery creates lite `LogOption` objects
2. `readLiteMetadata(...)` reads only the head and tail windows of each file to extract enough display metadata

Lite extraction is intentionally approximate-but-fast:

- head: first prompt, sidechain flag, original cwd, team name, agent setting
- tail: custom title, AI title fallback, summary, tag, git branch, PR metadata

The design depends on the write path re-appending important metadata near EOF. Without that, old titles/tags would fall out of the tail window and disappear from the resume UI.

Full replay only happens when needed, via `loadFullLog(...)` or direct resume loading.

## Resume Loading Pipeline

`loadConversationForResume(...)` in `utils/conversationRecovery.ts` is the centralized resume loader.

It supports several sources:

- no explicit source: load most recent resumable conversation
- session ID
- already-loaded `LogOption`
- explicit `.jsonl` path for cross-directory resume

Verified pipeline:

1. pick or resolve the source log
2. fully load it if it is still lite
3. copy auxiliary plan/file-history state used by resume
4. run `checkResumeConsistency(...)` for telemetry on replay drift
5. restore invoked skills from transcript attachments
6. deserialize and normalize messages
7. detect interrupted turns and synthesize continuation/sentinel messages when needed
8. run `SessionStart` resume hooks
9. return messages plus file-history, attribution, content-replacement, context-collapse, and session metadata

So the resume result is a reconstructed session package, not only a transcript array.

## Message Deserialization and Interruption Repair

Before resumed messages are handed back to the runtime, `deserializeMessagesWithInterruptDetection(...)` repairs several classes of persisted state:

- legacy attachment type migration
- stripping invalid persisted permission modes
- filtering unresolved tool uses
- filtering orphaned thinking-only assistant messages
- filtering whitespace-only assistant messages
- interruption detection
- synthetic continuation prompt insertion for interrupted turns
- synthetic assistant sentinel insertion when the last relevant message is a user message

That last step is particularly important: the stored transcript may not itself be API-valid for immediate reuse, so resume normalizes it into a safe model-facing form.

## Resume State Restoration Beyond Messages

`processResumedConversation(...)` and `restoreSessionStateFromLog(...)` restore several runtime layers after the transcript is loaded.

Verified restored state includes:

- file history state
- attribution state
- context-collapse commit log and staged snapshot
- todo state for non-v2 todo mode
- agent context and model override
- session mode
- worktree state
- custom title, tag, PR link, and related metadata caches

For non-fork resume, the runtime also:

- switches the active session ID
- points `sessionFile` at the resumed transcript
- re-appends metadata to the resumed file
- restores worktree cwd if the persisted worktree still exists

For forked resume, content replacement records are re-seeded into the new session so cache-stable tool-result behavior is preserved.

## Worktree and Cross-Project Resume

The transcript stores persisted worktree state as a last-wins `worktree-state` entry.

On resume:

- if a fresh `--worktree` session already exists, that wins
- otherwise the app attempts to `chdir` back into the stored worktree path
- if the path no longer exists, the runtime clears the cached worktree state instead of re-persisting stale information

Cross-directory resume is also first-class:

- `loadConversationForResume(...)` accepts a direct `.jsonl` path
- `processResumedConversation(...)` can switch the active session using the resumed transcript's directory, not only the current project directory

So session identity and session file location are related but not strictly assumed to match the current cwd.

## Metadata Recovery Rules

The code treats metadata as session-scoped and mostly last-wins.

Important verified rules:

- custom title wins over AI title
- AI titles are persisted, but not re-appended the same way user titles are
- tag, mode, worktree state, PR link, and agent-setting are recovered from maps keyed by session ID
- context-collapse commits are ordered and replayed in order
- context-collapse snapshot is last-wins
- content replacements are keyed differently for main thread versus subagent sidechains

This means replay behavior varies by entry type; there is no single generic merge strategy.

## Source-of-Truth Corrections from Code

The code corrects several easy assumptions:

- A session file is not one linear conversation. It can contain multiple branches and leaves.
- Resume is not "load the last N lines". It is replay, repair, chain selection, normalization, and state restoration.
- The resume picker is not backed by full transcript parsing. It is mostly head/tail metadata enrichment over stat-only session discovery.
- Compaction does not simply delete old file content. The append-only log remains, and the loader reconstructs the live view with boundaries, relinks, and metadata scans.
- Sidechain/subagent persistence is not just a flag on main-session messages. It has separate per-agent transcript files and sidecar metadata.

## Practical Mental Model

The most accurate short model is:

- session persistence is an append-only event log in JSONL
- conversation display is a reconstructed branch view over that log
- resume is a replay-and-repair process that restores both transcript and runtime state
- fast session discovery uses lite metadata reads, while true recovery uses full replay

That model matches the code much better than "Claude stores a chat transcript and loads it back later."
