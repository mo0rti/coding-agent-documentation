# Session Discovery, Progressive Log Enrichment, and Resume Picker Behavior

## Scope

This document covers how the code finds resumable sessions before full transcript replay:

- stat-only session discovery
- progressive metadata enrichment
- same-repo worktree and all-project aggregation
- interactive `/resume` picker behavior
- slash-command and CLI search/selection behavior

Primary files read for this slice:

- `utils/sessionStorage.ts`
- `screens/ResumeConversation.tsx`
- `components/LogSelector.tsx`
- `commands/resume/resume.tsx`
- `main.tsx`
- `utils/crossProjectResume.ts`
- `dialogLaunchers.tsx`

## High-Level Model

The resume UI is intentionally not built on full transcript replay for every session on disk.

Instead, the code uses a layered pipeline:

1. discover candidate session files by directory scan and `stat`
2. build lite `LogOption` objects without reading transcript bodies
3. enrich only some of those logs by reading the file head and tail
4. filter out non-resumable entries
5. lazily enrich more entries as the picker scrolls closer to the end

Full transcript replay only happens once the user actually selects a session or when a code path explicitly needs the full log.

## Stat-Only Discovery

`getSessionFilesWithMtime(...)` is the first layer.

Verified behavior:

- it scans a project directory for `*.jsonl`
- it only accepts filenames that parse as valid session UUIDs
- it batches `stat(...)` calls with `Promise.all`
- it records `path`, `mtime`, `ctime`, and `size`

`getSessionFilesLite(...)` then converts that into lite `LogOption` records with:

- `messages: []`
- `isLite: true`
- `sessionId`
- `fullPath`
- file timestamps and size
- optional `projectPath`

This is the core reason session discovery is fast: no transcript parsing is required up front.

## Current-Project, Same-Repo, and All-Projects Modes

The code has several discovery scopes, not one global session list.

`fetchLogs(...)`:

- reads only the current project directory

`loadSameRepoMessageLogsProgressive(...)`:

- aggregates session files across same-repo worktrees
- uses worktree-aware directory matching
- deduplicates by session ID
- enriches only the initial visible slice

`loadAllProjectsMessageLogsProgressive(...)`:

- scans every project directory under Claude's projects home
- deduplicates by session ID
- enriches only the initial visible slice

So "show me sessions" can mean:

- current project only
- same repo including worktrees
- all known projects

The entrypoint determines which of those scopes is used.

## Worktree-Aware Discovery

`getStatOnlyLogsForWorktrees(...)` is the key same-repo discovery helper.

Important verified behavior:

- if there is only one worktree path, it falls back to the current project directory
- on Windows, worktree/project directory matching is case-insensitive
- worktree prefixes are sorted longest-first so more specific sanitized directory names win
- matching project dirs are loaded through `getSessionFilesLite(...)`
- duplicates across worktrees are removed by session ID, keeping the newest modified entry

This is more careful than a naive "scan sibling directories" approach, especially on Windows and repos with multiple similarly named worktrees.

## Progressive Enrichment

The initial resume picker does not enrich every discovered session.

The code uses `INITIAL_ENRICH_COUNT = 50`, with the explicit comment that each enrichment reads up to head + tail windows per file, giving a much better initial picker than the older smaller default while staying cheap on disk I/O.

`enrichLogs(...)`:

- starts at a supplied `startIndex`
- enriches until it has produced `count` visible results
- advances `nextIndex` past both accepted and filtered-out candidates

That last point matters: filtered sessions still consume scan progress, so progressive loading can skip over hidden sessions and keep fetching until it has enough visible rows.

## What Enrichment Adds

`enrichLog(...)` uses `readLiteMetadata(...)` to fill in picker-facing fields without doing full transcript replay.

Verified enriched metadata includes:

- `firstPrompt`
- `gitBranch`
- `isSidechain`
- `teamName`
- `customTitle`
- AI-title fallback
- `summary`
- `tag`
- `agentSetting`
- PR metadata
- `projectPath`

If neither `firstPrompt` nor `customTitle` can be extracted, the code sets a fallback title of `(session)` so those sessions are still reachable in `/resume`.

This is an important correction to the earlier failure mode where some sessions silently vanished from the picker.

## What Gets Filtered Out

Enrichment is not just annotation; it also performs resumability filtering.

Verified hidden sessions include:

- sidechains
- team/agent sessions identified by `teamName`

So the interactive session list is not "every transcript on disk". It is a curated subset aimed at top-level resumable user sessions.

The slash-command picker adds one more filter:

- it excludes the current active session ID via `filterResumableSessions(...)`

The interactive startup picker does not need that same guard in the same way because it is launched before entering a new resumed REPL.

## Full Loading Boundary

The code is careful about when it crosses from lite discovery into full replay.

It does that at selection time:

- `ResumeConversation` calls `loadConversationForResume(...)` when a row is selected
- `/resume` slash picker calls `loadFullLog(...)` only if the chosen row is still lite
- direct UUID and title matches also upgrade to a full log only when they are about to be resumed

So the picker layer and the replay layer are intentionally separate subsystems.

## Interactive Resume Picker

`ResumeConversation` is the startup/interative picker screen mounted through `launchResumeChooser(...)`.

Verified behavior:

- initial load uses `loadSameRepoMessageLogsProgressive(worktreePaths)`
- toggling "all projects" swaps to `loadAllProjectsMessageLogsProgressive()`
- a `SessionLogResult` ref keeps both visible enriched logs and the full stat-only backlog
- `loadMoreLogs(...)` calls `enrichLogs(...)` against the existing stat-only backlog
- if a requested page yields zero visible logs because candidates were filtered out, it recursively keeps loading

This means the picker is not re-scanning the filesystem every time the user scrolls. It reuses the stat-only backlog and enriches incrementally.

## Picker-Level Filtering

The interactive picker applies additional UI-level filters beyond storage filtering.

Verified filters include:

- PR-based filtering for `--from-pr`
- tag tabs
- current-branch filter
- current-worktree-only filter
- all-projects toggle
- text search over display title, branch, tag, and PR metadata

Search is title/metadata search, not full transcript semantic search.

Two interesting code-backed details:

- `isDeepSearchEnabled` is currently hardcoded `false`
- `isAgenticSearchEnabled` is also currently hardcoded `false`

So the picker contains scaffolding for deeper search modes, but the active code path today is simpler metadata/title search.

## Grouping and Branch Presentation

`LogSelector` groups displayed entries by session ID with `groupLogsBySessionId(...)`.

That means the picker does not present every `LogOption` as a flat independent row when rename/group mode is enabled. Instead:

- the newest log for a session becomes the group header
- older logs for the same session are shown as expandable children
- the header label shows `(+N other sessions)` when forks/branches exist

This is a subtle but important UI behavior:

- the discovery pipeline can produce multiple leaf logs for one session
- the picker intentionally regroups them back under one session header for navigation

So "one row" in the UI is often "one session with one or more branch leaves", not necessarily "one transcript file and one chain".

## Auto-Pagination in the Picker

`LogSelector` drives lazy loading based on focus position.

Verified behavior:

- it computes a visible row count from terminal height
- when the focused index approaches the end of loaded rows, it asks for more
- the load request size is `visibleCount * 3`

This is not just a "Load more" button. Scrolling toward the bottom automatically triggers additional enrichment.

## Cross-Project Resume Handling

`checkCrossProjectResume(...)` decides whether a selected session can be resumed directly or requires a shell command in a different directory.

Verified rules:

- if the session is in the current directory scope, no special handling
- if all-projects mode is showing a different project and the target belongs to the same repo worktree set, direct resume is allowed
- otherwise the UI generates `cd <projectPath> && claude --resume <sessionId>`

There is also a gated distinction:

- for non-`ant` users, same-repo worktree direct resume is not used, and a command is generated instead

So cross-project visibility and cross-project resumability are separate decisions.

## CLI `--resume` Search Behavior

`main.tsx` adds another layer on top of the picker.

Verified behavior for `--resume <value>`:

- if `<value>` parses as a UUID, it is treated as a direct session target
- otherwise the code first tries exact custom-title match
- if that yields exactly one match, it resumes directly
- if not, the value becomes `initialSearchQuery` for the interactive picker

This means a non-UUID `--resume foo` is not automatically an error. It can be:

- an exact title hit
- or just a prefilled picker search term

## `--from-pr` Behavior

`main.tsx` also routes `--from-pr` into the same picker surface.

Verified behavior:

- `--from-pr` with no value means show only sessions linked to any PR
- `--from-pr <number-or-url>` constrains the picker to that PR number

The filtering itself happens in `ResumeConversation`, not deep in storage.

So PR-linked selection is a picker concern layered over normal session discovery.

## Slash Command `/resume`

The slash command has different behavior from the startup picker.

With no argument:

- it loads same-repo logs eagerly through `loadSameRepoMessageLogs(...)`
- filters out sidechains and the current session
- presents a picker

With an argument:

- it first checks for UUID match
- if a UUID is found in enriched logs, it resumes that
- if not, it tries `getLastSessionLog(...)` directly as a fallback for sessions dropped by enrichment
- then it tries exact custom-title match
- otherwise it returns a not-found or multiple-matches error

That direct `getLastSessionLog(...)` fallback is important. It prevents the slash command from failing just because a session was not visible in the enriched picker list.

## User-Facing Picker Features

`LogSelector` adds several user-facing behaviors on top of session discovery:

- inline rename via `saveCustomTitle(...)`
- session preview
- search mode with prefilled query support
- tag tabs
- project/worktree/branch toggles
- grouped tree navigation for sessions with multiple branch leaves

The picker is therefore not just a list renderer. It is a session-navigation UI with its own state model over the discovered logs.

## Source-of-Truth Corrections from Code

The code corrects several easy assumptions:

- `/resume` is not backed by full transcript parsing for every session; it starts with `stat` plus head/tail reads.
- same-repo worktree discovery is a first-class path, not an afterthought.
- all-projects mode is not just a broader current-project scan; it has separate aggregation and dedup behavior.
- the picker is not a flat list of conversations; it regroups multiple leaves by session ID.
- `--resume <text>` is not only "session ID or error"; it can become a prefilled picker search.
- the slash command is more defensive than the startup picker because it has a direct file-based fallback for sessions filtered out during enrichment.

## Practical Mental Model

The most accurate short model is:

- discovery is filesystem-first and cheap
- enrichment is partial, progressive, and picker-oriented
- full replay is deferred until selection
- same-repo worktrees are treated as part of the normal resume surface
- the picker is a grouped session navigator layered on top of lite logs, not a raw transcript browser

That matches the current code much better than "the app loads all chats and shows them in a menu."
