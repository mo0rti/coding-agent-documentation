# Cross-Project Resume, Session Switching, and Alternate Resume Entry Paths

## Scope

This document covers the code paths that resume a session from something other than the default "pick a local session and continue it" flow:

- cross-project and same-repo-worktree resume
- session switching during resume
- direct transcript-file resume
- print-mode resume differences
- teleport resume paths
- other alternate entrypoint branches visible from `main.tsx`

Primary files read for this slice:

- `main.tsx`
- `utils/sessionRestore.ts`
- `utils/conversationRecovery.ts`
- `utils/crossProjectResume.ts`
- `utils/sessionUrl.ts`
- `cli/print.ts`
- `utils/teleport.tsx`
- `screens/ResumeConversation.tsx`

## High-Level Model

There is not one universal "resume" path in this codebase.

The runtime has several distinct entry modes that all eventually produce either:

- a reconstructed local conversation through `loadConversationForResume(...)` and `processResumedConversation(...)`
- or an alternate message bootstrap that bypasses the normal local transcript-resume path

The main branches are:

- `--continue`
- `--resume <session-id>`
- interactive picker resume
- cross-project resume from the picker
- resume from a transcript file
- print-mode `--resume`
- teleport resume
- remote session creation/resume-adjacent flows

The important thing is that these paths do not all switch sessions, hydrate state, or even source messages in the same way.

## The Session-Switching Core

The real session-switch behavior lives in `processResumedConversation(...)` in `utils/sessionRestore.ts`.

For non-fork resume:

- it determines the target session ID from `sessionIdOverride ?? result.sessionId`
- it calls `switchSession(...)`
- it can pass `dirname(transcriptPath)` to `switchSession(...)`
- it renames the asciicast recording to the resumed session ID
- it resets the session-file pointer
- it restores cost state for the resumed session

That `transcriptPath` argument is the key to cross-project correctness.

The code explicitly treats the transcript file's directory as authoritative when resuming from a transcript that does not live in the current project directory.

## Cross-Project Resume

Cross-project handling is not embedded deep in transcript loading. It is first decided at the picker/UI layer by `checkCrossProjectResume(...)`.

That function returns one of three states:

- not cross-project
- cross-project but same-repo worktree
- cross-project and different project, with a generated shell command

Verified behavior:

- if the picker is not in all-projects mode, cross-project logic does not apply
- if the selected log has no `projectPath`, cross-project logic does not apply
- same-repo worktree direct resume is only enabled for `USER_TYPE === 'ant'`
- otherwise the user gets `cd <projectPath> && claude --resume <sessionId>`

So the system distinguishes:

- visibility across projects
- direct resumability from the current process
- fallback shell handoff to the correct directory

Those are related but separate concerns.

## Same-Repo Worktree Resume

Same-repo worktrees are treated as a special case of cross-project resume, not as generic foreign-project resume.

In the interactive picker:

- `ResumeConversation` checks `checkCrossProjectResume(...)`
- if the selected log belongs to a same-repo worktree, it resumes directly
- the resumed transcript path is then passed down so `switchSession(...)` can anchor the session to the transcript's real directory

This matters because same-repo worktree resume is still local and still expected to preserve the resumed session's disk identity, cost state, recording path, and metadata.

## Interactive Picker Cross-Project Flow

The interactive startup picker in `ResumeConversation` handles cross-project cases differently depending on the result of `checkCrossProjectResume(...)`.

Verified behavior:

- same-repo worktree selection resumes directly
- different-project selection copies a shell command to the clipboard
- the UI shows a transient message telling the user to run that command
- the process exits after showing the message

So the picker is not trying to "force" every session back into the current process. It respects project-directory boundaries when the code thinks direct resume would be wrong.

## Slash Command Cross-Project Flow

The `/resume` slash command has a similar but not identical cross-project flow.

Verified behavior in `commands/resume/resume.tsx`:

- the selected picker item is upgraded to a full log if needed
- `checkCrossProjectResume(...)` is applied
- same-repo worktree sessions can resume directly
- different-project sessions produce a user-facing command string instead of resuming

So the slash command and the startup picker share the same cross-project decision primitive, even though they live in different UI surfaces.

## `--continue`

`--continue` is the simplest alternate resume path in `main.tsx`.

Verified behavior:

- it calls `loadConversationForResume(undefined, undefined)`
- that means "load the most recent resumable session"
- `loadConversationForResume(...)` skips live background/daemon sessions when that feature is enabled
- then `processResumedConversation(...)` performs the normal session-switch and restoration flow

This is important because `--continue` does not depend on picker selection or explicit session ID resolution. It is a "most recent resumable local session" entrypoint.

## `--resume <session-id>`

The normal CLI resume path in `main.tsx` eventually converges on:

- `loadConversationForResume(matchedLog ?? sessionId, undefined)`
- `processResumedConversation(result, { forkSession, sessionIdOverride, transcriptPath }, ...)`

Two details matter here:

- if the session was found by exact custom-title match, `matchedLog.fullPath` is preferred so cross-worktree resume can preserve the actual transcript location
- `sessionIdOverride` is passed explicitly so the resumed session ID stays the requested one even if the loaded log path came from another resolution route

So even the "simple UUID resume" path already supports cross-worktree correctness through `fullPath`.

## Direct Transcript-File Resume in Interactive Mode

`main.tsx` has a separate branch that tries to interpret a non-UUID `--resume <value>` as a file path.

Verified behavior:

- it resolves the provided path
- it tries `loadTranscriptFromFile(resolvedPath)`
- if that succeeds, it passes the resulting `LogOption` to `loadConversationForResume(...)`
- it then calls `processResumedConversation(...)` with `transcriptPath: result.fullPath`

If the file path does not exist, the code falls through and continues trying other resume interpretations instead of immediately failing.

This means interactive-mode file resume is a real first-class branch, not just a testing helper.

## Direct JSONL Resume in Print Mode

Print mode uses a different mechanism.

`parseSessionIdentifier(...)` in `utils/sessionUrl.ts` accepts:

- plain UUIDs
- `.jsonl` file paths
- URLs

For `.jsonl` files it intentionally:

- sets a random synthetic `sessionId`
- marks `jsonlFile`
- marks `isJsonlFile: true`

Then `cli/print.ts` calls:

- `loadConversationForResume(parsed.sessionId, parsed.jsonlFile || undefined)`

Because `sourceJsonlFile` is provided, `loadConversationForResume(...)` ignores the synthetic UUID for transcript lookup and directly chain-walks the JSONL file path.

So print-mode file resume is not implemented the same way as interactive-mode file resume:

- interactive mode loads a `LogOption` from the file first
- print mode routes the file path through the `sourceJsonlFile` argument

## URL-Based Resume in Print Mode

Print mode has one more alternate branch: URL-based resume.

`parseSessionIdentifier(...)` treats any parseable URL as:

- `isUrl: true`
- `ingressUrl: <full url>`
- `sessionId: randomUUID()`

Then `cli/print.ts` may hydrate local transcript state before loading:

- CCR v2: `hydrateFromCCRv2InternalEvents(...)`
- v1 session ingress: `hydrateRemoteSession(...)`

Only after hydration does it call `loadConversationForResume(...)`.

This is a substantial semantic difference from normal CLI resume:

- the local JSONL may be synthesized from remote session state first
- an empty hydrated transcript is treated as a valid "start empty session" case for URL/CCR paths

So print-mode URL resume is really a remote-hydration bootstrap path, not just a fancy session-ID parser.

## Print Mode Resume Restrictions

Print mode is intentionally stricter than interactive mode.

Verified behavior:

- `--resume` in print mode requires a valid session identifier shape
- the error path explicitly says session ID usage is required, though the parser also accepts `.jsonl` and URL forms
- if no conversation is found for a plain local session ID, print mode exits with an error
- for URL/CCR-v2 resume with an empty hydrated transcript, print mode falls back to startup hooks and begins as a fresh session

Print mode also supports resume-specific modifiers that are not part of the normal interactive picker flow:

- `--resume-session-at <message.uuid>`
- `--rewind-files <user-message-id>` with `--resume`

Those make print mode a more surgical replay interface than the normal REPL picker paths.

## Forked Resume

Forked resume is another important alternate mode.

Across both interactive and print flows, the code treats `--fork-session` as:

- keep the fresh current session ID instead of switching to the resumed one
- do not take ownership of the original session's worktree state
- re-seed content replacement records into the new session when needed

This is not just "resume and then change the ID." It is a different ownership model for:

- transcript persistence
- worktree state
- content replacement state
- later resume behavior

## Teleport Resume

Teleport resume is separate from ordinary local transcript resume.

There are two main branches in `main.tsx`:

- interactive teleport picker via `launchTeleportResumeWrapper(...)`
- direct `--teleport <sessionId>`

For the interactive picker branch:

- the user selects a remote/code session
- the code optionally checks out the teleported branch locally
- `processMessagesForTeleportResume(...)` deserializes the remote transcript and appends teleport notice messages
- those messages go straight into REPL bootstrap

For direct `--teleport <sessionId>`:

- the code validates repository compatibility first
- it may prompt the user to switch to a known checkout directory
- it validates local git state
- it then fetches/constructs teleported messages and boots the REPL from those messages

This is an important correction: teleport does not primarily go through `processResumedConversation(...)`. It is a parallel resume-like entrypoint that builds an initial message list by different means.

## Remote Session Creation Is Resume-Adjacent, Not Resume-Equivalent

`--remote` lives in the same broad startup branch as resume and teleport, but it is not a normal resume path.

Verified behavior:

- it creates a CCR remote session
- in non-TUI mode it prints session info and exits
- in TUI mode it switches to the remote session ID and boots the REPL with a remote session config
- it does not first load a historical local conversation through `loadConversationForResume(...)`

So `--remote` is best thought of as "create and attach to a remote session" rather than "resume a prior local conversation".

## ccshare Resume Caveat

`main.tsx` contains a separate resume branch for ccshare URLs:

- `parseCcshareId(...)`
- `loadCcshare(...)`

But the referenced `utils/ccshareResume.js` source is not present in this workspace snapshot.

So this part of the behavior is only call-site verified here:

- ccshare can be detected from `--resume <value>`
- successful ccshare loads are fed into `loadConversationForResume(...)`
- ccshare resumes are forced through `forkSession: true`
- analytics label the entrypoint as `ccshare`

The internal ccshare loading logic itself was not verifiable in this snapshot.

## Transcript Path as the Location Authority

One of the most important implementation details across these alternate paths is that `transcriptPath` is the location authority for resumed sessions.

That path influences:

- where `switchSession(...)` anchors the resumed session
- which project directory becomes associated with the resumed transcript
- whether cross-worktree and cross-project resume stays attached to the original session file instead of the current cwd

This is why many alternate resume callers pass some combination of:

- `matchedLog.fullPath`
- `result.fullPath`
- `dirname(result.fullPath)`

Without that, resuming a valid transcript from the wrong directory would silently drift the session into the wrong project root.

## Source-of-Truth Corrections from Code

The code corrects several easy assumptions:

- Cross-project resume is not automatically forbidden; same-repo worktrees can resume directly.
- Direct transcript-file resume exists in both interactive and print flows, but the two modes implement it differently.
- Teleport resume is not just "resume from another backend"; it bootstraps messages through a separate path.
- Print-mode resume is not just the interactive resume path without Ink; it has remote hydration, URL parsing, partial replay options, and stricter validation.
- `--remote` is adjacent to resume logic in startup dispatch, but it is a create/attach path, not a transcript-replay path.

## Practical Mental Model

The most accurate short model is:

- normal resume replays a local transcript and switches the session context
- cross-project/worktree resume adds directory-anchoring rules on top of that
- file and URL resume are alternate loaders feeding the same or similar restoration pipeline
- teleport and remote session flows are resume-adjacent but not equivalent to ordinary local transcript resume

That matches the current code much better than treating every startup path as just another flavor of `/resume`.
