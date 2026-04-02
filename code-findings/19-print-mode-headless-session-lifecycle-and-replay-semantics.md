# Print Mode, Headless Session Lifecycle, and Replay Semantics

## Scope

This document covers the non-interactive `-p` / `--print` runtime:

- headless startup and store initialization
- how initial messages are loaded in print mode
- how `--continue`, `--resume`, and `--teleport` are handled in headless mode
- structured I/O and output-format constraints
- interrupted-turn replay behavior
- print-mode-only replay controls like `--resume-session-at` and `--rewind-files`

Primary files read for this slice:

- `cli/print.ts`
- `main.tsx`
- `utils/conversationRecovery.ts`
- `utils/sessionRestore.ts`
- `utils/sessionUrl.ts`

## High-Level Model

Print mode is not just "interactive mode without Ink".

The code reuses some shared resume and query utilities, but the overall shape is different:

- `main.tsx` builds a headless store instead of mounting the REPL
- `cli/print.ts` owns startup validation, initial-message loading, and the headless run loop
- structured I/O becomes the transport boundary for input, output, permission prompts, and control requests

So the correct mental model is:

- interactive mode is REPL-first
- print mode is transport-first

## Entry Into Headless Mode

`main.tsx` routes non-interactive sessions into `runHeadless(...)`.

Before doing that, it prepares several print-specific pieces:

- a headless `AppState` store via `createStore(...)`
- print-compatible command filtering
- deferred or immediate MCP connection updates into the headless store
- session-start hook promise handling tailored for non-interactive startup
- print-mode settings and permission context initialization

This means headless mode still has an application state model, but it is not React-driven.

## Headless Initial State

The headless initial state is explicitly assembled in `main.tsx`, not inherited from the interactive REPL tree.

Verified state shaping includes:

- MCP clients, commands, and tools
- tool permission context
- effort value
- fast mode when enabled
- advisor model when enabled
- `kairosEnabled` for headless task/agent behavior

One subtle but important point from the code comments:

- headless mode had to explicitly set `kairosEnabled`, otherwise some scheduled/background task behavior would run synchronously and block user input

So headless mode is not just a consumer of generic defaults; it has its own correctness-sensitive state assembly.

## SessionStart Hooks in Headless Mode

Headless startup hooks are handled differently from interactive startup.

In `main.tsx`:

- the startup hook promise is skipped entirely for `continue`, `resume`, `teleport`, and setup-trigger paths
- otherwise a `sessionStartHooksPromise` is created early and passed into `runHeadless(...)`

Then in `cli/print.ts`:

- `loadInitialMessages(...)` awaits that promise when no specialized resume/continue path consumed it
- if no promise was supplied, it falls back to `processSessionStartHooks('startup')`

This split is important:

- some headless startup paths get normal startup hooks
- resume/continue/teleport paths either run different hooks later or intentionally skip that startup path

## `loadInitialMessages(...)` Is the Headless Replay Boundary

The real headless replay/bootstrap boundary is `loadInitialMessages(...)` in `cli/print.ts`.

It decides how the session starts before the run loop begins.

Its major branches are:

- `--continue`
- `--teleport`
- `--resume`
- normal startup with startup hooks

That function returns:

- initial messages
- optional interrupted-turn state
- optional resumed agent setting

So in print mode, initial transcript replay is resolved before the main run loop starts consuming user input.

## `--continue` in Print Mode

The print-mode `--continue` branch uses shared conversation loading, but its restoration path is lighter than the interactive REPL startup path.

Verified behavior:

- calls `loadConversationForResume(undefined, undefined)`
- matches coordinator mode and may refresh agent definitions
- reuses the resumed session ID unless `--fork-session`
- switches session with `result.fullPath ? dirname(result.fullPath) : null`
- resets the session file pointer only if persistence is enabled
- restores session state through `restoreSessionStateFromLog(...)`
- restores metadata caches with `restoreSessionMetadata(...)`
- writes the current mode entry when coordinator mode is active

This is shared-resume logic adapted for a headless process, not a direct call into `processResumedConversation(...)`.

## `--resume` in Print Mode

Print mode has a stricter and more transport-aware resume branch than interactive mode.

It first parses the resume identifier through `parseSessionIdentifier(...)`, which accepts:

- UUIDs
- `.jsonl` paths
- URLs

Then it may hydrate local transcript state before replay:

- CCR v2: `hydrateFromCCRv2InternalEvents(...)`
- v1 ingress URL: `hydrateRemoteSession(...)`

Only after that does it call:

- `loadConversationForResume(parsed.sessionId, parsed.jsonlFile || undefined)`

This is a key design difference from interactive mode:

- print mode can treat resume as a remote-hydration bootstrap
- interactive mode mostly treats resume as a local session-selection problem

## Empty-Session Resume Behavior

Print mode has a special case for empty hydrated sessions.

The code explicitly notes that CCR-v2 hydration can write an empty transcript file, causing `loadConversationForResume(...)` to return `{ messages: [] }` rather than `null`.

The runtime then treats:

- URL-based resume
- CCR-v2 resume

as valid "start fresh" cases and falls back to startup hooks instead of failing.

For ordinary local session-ID resume, the same empty result is treated as an error.

That distinction is very specific to the headless/remote-aware runtime.

## Teleport in Print Mode

Print-mode teleport is handled as a separate branch inside `loadInitialMessages(...)`.

Verified behavior:

- enforces remote-session policy
- requires a concrete teleport session ID
- validates git state
- calls `teleportResumeCodeSession(...)`
- checks out the teleported branch when applicable
- converts the teleported log into model-ready messages with `processMessagesForTeleportResume(...)`

So teleport in print mode is not a variant of normal local transcript replay. It is a specialized message bootstrap path.

## Print-Mode Replay Controls

Print mode exposes replay controls that do not exist as normal picker features in interactive mode.

Verified controls include:

- `--resume-session-at <message.uuid>`
- `--rewind-files <user-message-id>`

`--resume-session-at`:

- requires `--resume`
- truncates the resumed message list to a specific assistant message UUID and everything before it

`--rewind-files`:

- requires `--resume`
- cannot be used with a prompt
- requires the target UUID to belong to a user message
- restores filesystem state and exits immediately

These make print mode a more scriptable replay surface than the normal REPL.

## Input Requirements in Print Mode

Unlike the interactive REPL, headless mode validates that it has enough input material to proceed.

After loading initial messages, `cli/print.ts` checks whether it has:

- a prompt
- a valid resume session ID or `.jsonl` path
- or an SDK URL

If none of those is present, print mode errors out.

This is another example of transport-first semantics:

- the process refuses to stay alive without an input source or a valid replay source

## Structured I/O Is the Headless Transport Layer

`getStructuredIO(...)` builds the transport abstraction for headless mode.

Verified behavior:

- string input is normalized into a structured user-message stream
- empty string becomes an empty stream
- `sdkUrl` selects `RemoteIO`
- otherwise plain `StructuredIO` is used

That transport is then used for:

- emitting output messages
- permission prompts
- control requests
- MCP-related messaging
- elicitation flows

So headless execution is centered around a structured message channel, not terminal rendering.

## Output-Format Semantics

Headless mode has explicit output-format-specific behavior.

Verified examples:

- `stream-json` installs a stdout guard so stray text does not corrupt NDJSON consumers
- `stream-json` requires `--verbose`
- hook events are emitted into the structured output stream only in `stream-json` + verbose mode
- final message accumulation is optimized differently depending on output format

This means output format is not just formatting at the end. It changes runtime behavior while the session is running.

## Permission and Status Emission

Because there is no REPL UI, headless mode pushes runtime state changes into structured output.

Verified mechanisms:

- permission prompts are surfaced through structured I/O callbacks
- `notifySessionStateChanged(...)` is used for running/idle/requires-action state
- permission-mode changes emit SDK-facing status messages
- auth status and rate-limit updates can also be pushed into the output stream

So headless mode externalizes internal session state that interactive mode would mostly render locally.

## Interrupted-Turn Replay

Headless mode has explicit auto-resume logic for interrupted turns.

Inside `runHeadlessStreaming(...)`:

- if `turnInterruptionState` exists
- and `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` is set

then the runtime:

- removes the interrupted message and its sentinel from the mutable message list
- re-enqueues the interrupted prompt content

The code comment makes the intent explicit:

- the model should see that prompt exactly once after restart

This is one of the clearest examples of headless replay semantics differing from passive transcript display.

## Headless Run Loop Semantics

`runHeadlessStreaming(...)` is the main headless execution loop.

Important verified responsibilities include:

- signal handling and graceful shutdown
- outbound queue management
- permission-mode status emission
- prompt suggestion and auth-status push channels
- message-state mutation over a mutable transcript array
- read-file cache seeding from resumed messages
- optional interrupted-turn auto-resume

This is not simply "call ask() once and print the answer". It is a session runner with its own lifecycle and streaming state machinery.

## Headless State Restoration Versus Interactive Restoration

Print mode reuses `restoreSessionStateFromLog(...)`, but it does not reuse the full interactive restoration flow.

What it restores directly:

- file history state
- attribution state
- context-collapse state
- todo state when applicable
- metadata caches via `restoreSessionMetadata(...)`

What it does not do in the same way:

- it does not go through REPL mounting
- it does not call `processResumedConversation(...)`
- it does not rely on REPL-local composition to finish restoration

So the shared state-restore helpers are reused, but the orchestration layer is distinct.

## Headless MCP and Plugin Timing

The headless path also differs in when it loads and applies auxiliary runtime integrations.

Verified examples from `main.tsx` and `cli/print.ts`:

- print-mode MCP connections push incremental client/tool/command updates into the headless store
- headless users can await plugin synchronization because the CLI may exit immediately after one turn
- deferred prefetches start immediately in headless mode because there is no user typing delay to hide behind

This reinforces that headless mode optimizes for short-lived orchestrated runs, not for persistent terminal interaction.

## Source-of-Truth Corrections from Code

The code corrects several easy assumptions:

- print mode is not just the REPL with different output formatting
- headless resume is not one path; `continue`, `resume`, `teleport`, and startup each have their own bootstrap logic
- output format changes runtime behavior, not just final rendering
- print mode can treat remote/URL resume as an empty-session bootstrap instead of an error
- interrupted-turn replay in headless mode can actively re-enqueue prompts on restart

## Practical Mental Model

The most accurate short model is:

- `main.tsx` builds a headless runtime shell
- `loadInitialMessages(...)` decides how the session is bootstrapped or replayed
- `runHeadlessStreaming(...)` runs a transport-oriented session loop over structured I/O
- resume in print mode is shared with interactive mode at the data-loading layer, but not at the orchestration layer

That matches the code much better than thinking of `--print` as "the same app, just without UI."
