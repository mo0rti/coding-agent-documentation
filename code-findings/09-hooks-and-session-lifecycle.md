# Hooks And Session Lifecycle

## Scope

This document covers how hook configuration is sourced and merged, how hook execution differs between interactive and outside-REPL paths, and how session persistence, resume, clear, compact, and shutdown are tied together through transcript storage.

Primary source files:

- `setup.ts`
- `utils/sessionStart.ts`
- `utils/hooks.ts`
- `utils/hooks/sessionHooks.ts`
- `utils/hooks/hooksConfigSnapshot.ts`
- `utils/hooks/fileChangedWatcher.ts`
- `utils/hooks/registerFrontmatterHooks.ts`
- `utils/hooks/registerSkillHooks.ts`
- `utils/plugins/loadPluginHooks.ts`
- `types/hooks.ts`
- `bootstrap/state.ts`
- `utils/sessionStorage.ts`
- `utils/sessionRestore.ts`
- `utils/conversationRecovery.ts`
- `QueryEngine.ts`
- `utils/processUserInput/processUserInput.ts`
- `commands/clear/conversation.ts`
- `commands/compact/compact.ts`
- `utils/gracefulShutdown.ts`
- `cli/print.ts`
- `entrypoints/init.ts`

## The Key Architectural Split

The code does not implement "session lifecycle" as only startup and shutdown.

Two systems run in parallel:

1. the hook system emits configurable behavior at lifecycle and tool boundaries
2. the transcript/session system persists enough state to reconstruct a conversation later

Those systems meet at a few specific seams:

- `SessionStart`, `SessionEnd`, `PreCompact`, `PostCompact`, `CwdChanged`, and `FileChanged` hooks are true lifecycle events
- hook output can inject additional context, an initial user message, permission decisions, or dynamic watch paths
- transcript persistence is what makes `--resume`, `/resume`, `/continue`, and post-crash recovery work

So the runtime model is not "hooks around a stateless chat loop". The session has its own persisted identity and hook-driven lifecycle.

## Hooks Are Assembled From Multiple Sources

`utils/hooks.ts` does not read hooks from one place.

`getHooksConfig(...)` merges hooks for a given event from:

- the startup snapshot from `hooksConfigSnapshot.ts`
- registered hooks in bootstrap state
- session-scoped hooks in `AppState.sessionHooks`
- session-scoped function hooks in `AppState.sessionHooks`

That means the effective hook set is assembled at execution time, not loaded once from a single settings file.

### Startup snapshot

`setup.ts` calls `captureHooksConfigSnapshot()` after `setCwd(...)`.

That snapshot:

- is taken once at startup
- is filtered by managed-hook policy
- is the base hook source for normal settings-defined hooks

This is why setup comments explicitly say hook config must be captured after cwd is finalized.

### Registered hooks

`bootstrap/state.ts` stores `registeredHooks`, which are a shared registry for:

- SDK callback hooks
- plugin native hooks
- internal callback hooks such as session file access analytics

`registerHookCallbacks(...)` merges into that store rather than replacing it.

### Session hooks

`utils/hooks/sessionHooks.ts` stores per-session hooks in `AppState.sessionHooks`.

These are:

- temporary
- in-memory only
- scoped by session ID or agent ID

This is where frontmatter and skill hooks land, and where runtime-added function hooks live.

## Policy Filters Apply Before Hook Execution

`utils/hooks/hooksConfigSnapshot.ts` makes the managed-policy model explicit.

Important verified behavior:

- `policySettings.disableAllHooks === true` disables all hooks
- `policySettings.allowManagedHooksOnly === true` limits execution to managed hooks
- non-managed `disableAllHooks` does not disable managed hooks; it effectively becomes managed-only mode
- strict plugin-only customization blocks ordinary settings hooks, but not all plugin-provided or policy-provided hook paths

The execution layer mirrors this:

- `getHooksConfig(...)` skips plugin hooks when managed-only mode is active
- it skips session hooks entirely when managed-only mode is active, so frontmatter hooks cannot bypass policy

So policy filtering is not only a settings concern. It is also enforced at hook assembly time.

## Plugin Hooks, SDK Hooks, And Frontmatter Hooks Use Different Registration Paths

The code keeps the sources separate even though they end up in one merged hook set.

### Plugin hooks

`utils/plugins/loadPluginHooks.ts` loads enabled plugins, converts their hook config to `PluginHookMatcher[]`, then clears and re-registers plugin hooks atomically.

That clear-then-register swap is load-bearing:

- the code comments document a previous bug where plugin hooks, especially `Stop`, disappeared after plugin cache clears
- the current implementation keeps old plugin hooks active until the new set is ready

### SDK callback hooks

`cli/print.ts` registers hook callbacks from SDK initialization requests by building `HookCallbackMatcher[]` with `structuredIO.createHookCallback(...)` and passing them into `registerHookCallbacks(...)`.

So SDK hooks are not special-cased later. They join the same merged registry as other registered hooks.

### Frontmatter and skill hooks

`registerFrontmatterHooks.ts` and `registerSkillHooks.ts` register hooks into session state.

Important verified behavior:

- frontmatter hooks become session-scoped hooks
- agent frontmatter `Stop` hooks are converted to `SubagentStop`
- skill hooks can be one-shot via `once: true`, implemented by removing them after a successful execution callback

So frontmatter hooks are intentionally ephemeral and session-bound, not persisted into settings.

## Hooks Have A Broad Event Model

The SDK-facing hook schemas in `entrypoints/agentSdkTypes.ts` and `types/hooks.ts` show the supported lifecycle model is wider than the older docs suggest.

Verified event families include:

- tool lifecycle: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`
- prompt/permission lifecycle: `UserPromptSubmit`, `PermissionRequest`, `PermissionDenied`, `Notification`
- session lifecycle: `SessionStart`, `SessionEnd`, `Setup`, `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`
- compaction lifecycle: `PreCompact`, `PostCompact`
- environment/lifecycle changes: `CwdChanged`, `FileChanged`, `ConfigChange`, `InstructionsLoaded`
- task and teammate events: `TeammateIdle`, `TaskCreated`, `TaskCompleted`
- MCP elicitation events: `Elicitation`, `ElicitationResult`

The hook system is therefore a general lifecycle/event bus, not only a shell-command wrapper around tool execution.

## Hook Matching Is Event-Specific

`getMatchingHooks(...)` derives the matcher query differently for each event.

Examples verified from code:

- tool events match on `tool_name`
- `SessionStart` matches on `source` such as `startup`, `resume`, `clear`, or `compact`
- `Setup` matches on `trigger`
- `PreCompact` and `PostCompact` match on compact trigger
- `Notification` matches on `notification_type`
- `SessionEnd` matches on exit reason
- `FileChanged` matches on the basename of the changed path

So the `matcher` field is not one universal syntax bound to tools only. Its meaning depends on the event.

## Hook Types Are Heterogeneous

`types/hooks.ts` and `utils/hooks.ts` show that hooks are not all shell commands.

Supported hook forms include:

- command hooks
- prompt hooks
- agent hooks
- HTTP hooks
- callback hooks
- function hooks

The execution pipeline treats them differently:

- callback hooks can fast-path entirely in memory
- function hooks are session-only TypeScript callbacks
- prompt hooks need a request-prompt bridge
- agent hooks run a nested query loop
- HTTP hooks return JSON and are validated like structured command-hook output

So "hook" is a common interface over several execution strategies.

## Hook Output Is Structured, Not Just Stdout

`processHookJSONOutput(...)` is one of the most important pieces of the subsystem.

Structured hook output can:

- stop continuation
- provide a stop reason
- make permission decisions
- supply additional context
- rewrite tool input
- rewrite MCP tool output
- provide an initial user message for `SessionStart`
- provide dynamic `watchPaths`
- answer permission-request hooks
- answer elicitation hooks

That is why hooks are architecturally important: they can change runtime behavior, not only print diagnostics.

## Interactive And Outside-REPL Hook Execution Are Different Pipelines

The code has two distinct execution paths.

### `executeHooks(...)`

This is the REPL/query-loop path.

Verified behavior:

- enforces the centralized trust gate
- assembles hooks through `getMatchingHooks(...)`
- emits progress messages
- supports command, prompt, agent, HTTP, callback, and function hooks
- tracks hook timing and telemetry
- yields structured results back into the conversation/runtime

This is the path used for things like `PreToolUse`, `PostToolUse`, and `UserPromptSubmit`.

### `executeHooksOutsideREPL(...)`

This is the non-conversational lifecycle path.

Verified behavior:

- also enforces the centralized trust gate
- does not yield model-visible messages
- returns `HookOutsideReplResult[]`
- is used for notification, compaction, cwd/file change, and session-end style flows

It also has deliberate limitations:

- prompt hooks outside REPL are not supported
- agent hooks outside REPL are not supported
- function hooks should not reach this path

So the system has one configuration model, but two different runtime execution surfaces.

## Trust Gating Is Centralized And Applies To All Hooks

`shouldSkipHookDueToTrust()` is explicit about why it exists.

Verified behavior:

- in interactive mode, all hooks require accepted workspace trust
- in non-interactive mode, trust is treated as implicit and hooks can run

The code comments call out historical vulnerabilities around hooks firing before trust.

That means hook trust is not a best-effort convention. It is a central execution guard.

## Internal Callback Hooks Are A Distinct Fast Path

`executeHooks(...)` has a fast path when every matched hook is an internal callback.

Examples include internal analytics hooks such as `sessionFileAccessHooks.ts`.

In that case the code:

- skips the heavier span/progress/abort/result pipeline
- runs the callbacks directly
- still records timing/telemetry

This matters because not every hook in the system is user-authored automation. Some are internal instrumentation hooks living in the same framework.

## SessionStart And Setup Hooks Are Orchestrated By `sessionStart.ts`

`processSessionStartHooks(...)` and `processSetupHooks(...)` are the lifecycle wrappers around the lower-level hook runner.

Important verified behavior:

- both short-circuit in bare mode
- both try to ensure plugin hooks are loaded first unless managed policy blocks them
- plugin load failure does not abort lifecycle hook execution
- `SessionStart` can aggregate `additionalContexts`
- `SessionStart` can emit `initialUserMessage`
- `SessionStart` can emit `watchPaths`, which are forwarded to the file-change watcher

This wrapper is also where the `pendingInitialUserMessage` side channel lives for print/headless flow.

## Startup Fires Hooks Differently In Interactive And Print Modes

`main.tsx` and `cli/print.ts` deliberately do not fire startup hooks in exactly the same place.

### Interactive startup

`main.tsx` starts `processSessionStartHooks('startup', ...)` early so it can overlap with MCP connection work.

But it guards this out for:

- non-interactive sessions
- `--continue` / `--resume`
- setup-only flows like `--init` and `--maintenance`

The code comment explicitly says this avoids firing startup hooks twice on resume.

### Print mode startup

`main.tsx` also kicks a `sessionStartHooksPromise` for print mode, and `cli/print.ts` joins it later.

If print mode is resuming a missing or empty session in CCR/URL paths, `cli/print.ts` falls back to `processSessionStartHooks('startup')` so a new empty session still gets startup hooks.

So startup hook timing is optimized separately for interactive and headless execution.

## Resume Fires `SessionStart:resume`, Not `startup`

`utils/conversationRecovery.ts` is the code-backed source of truth for resume hook behavior.

Verified behavior:

- after loading and deserializing the conversation, it runs `processSessionStartHooks('resume', { sessionId })`
- the resulting hook messages are appended to the resumed conversation

This is important because `main.tsx` explicitly avoids firing startup hooks for resume paths.

So resume is its own lifecycle event, not a special case of startup.

## `/clear` Runs SessionEnd Then SessionStart

`commands/clear/conversation.ts` shows that clear is a full lifecycle transition, not only a local state reset.

Verified order:

1. run `executeSessionEndHooks('clear', ...)` with the bounded session-end timeout
2. clear messages and session-local caches
3. regenerate the session ID
4. reset session file pointer and metadata
5. preserve selected background task state where needed
6. re-persist current mode/worktree state
7. run `processSessionStartHooks('clear')`

So `/clear` creates a new session lifecycle boundary, not just an empty transcript in the same session.

## Compaction Has Its Own Hook Lifecycle

`commands/compact/compact.ts` and `utils/hooks.ts` show compaction is explicitly hooked.

Verified behavior:

- `executePreCompactHooks(...)` runs with trigger `manual` or `auto`
- pre-compact hooks can return extra instructions
- those instructions are merged with user-supplied compact instructions
- `executePostCompactHooks(...)` runs after compaction and can return user-display messages

Reactive compact and traditional compact both preserve this lifecycle idea, even though the orchestration differs.

So compaction is not purely an internal token-management concern. It is a first-class lifecycle event in the hook system.

## Dynamic File Watching Is Hook-Driven

`utils/hooks/fileChangedWatcher.ts` is the owner of `CwdChanged` and `FileChanged` lifecycle behavior.

Important verified behavior:

- setup initializes the watcher after capturing the hook snapshot
- static watched paths come from `FileChanged` matcher names
- dynamic watch paths can be added by hook output
- `SessionStart` can seed `watchPaths`
- `CwdChanged` can recompute watch paths against the new cwd
- changed files trigger `executeFileChangedHooks(...)`

This means lifecycle hooks can expand the runtime watch surface after startup, not only react to a fixed set of files.

## Session Persistence Is Transcript-First

`QueryEngine.ts` and `utils/sessionStorage.ts` make the persistence model clear.

The conversation is not reconstructed from UI state. It is reconstructed from the transcript JSONL plus a few side-entry types.

`recordTranscript(...)`:

- filters messages through `cleanMessagesForLogging(...)`
- de-duplicates already-recorded messages by UUID
- preserves chain parentage with `startingParentUuid`
- inserts only new messages into the project/session store

This transcript-first design is why so much of resume logic works without relying on React state.

## The Transcript Is Written Earlier Than The Old Mental Model Suggests

`QueryEngine.submitMessage(...)` eagerly persists accepted user messages before the main query loop starts.

The code comment explains why:

- if the process dies before the API responds, resume must still find the accepted user prompt

This corrects a common assumption that persistence only happens after assistant output is produced. The code deliberately writes earlier to keep the session resumable.

## What Gets Persisted Is Filtered And Sometimes Transformed

`sessionStorage.ts` does not dump raw UI messages directly.

Important verified behavior:

- progress messages are not loggable
- most attachments are filtered for non-ant users
- `hook_additional_context` can be persisted for non-ant users only when a specific env flag is enabled
- for external users, REPL wrapper tool calls are stripped from persisted transcripts so resume shows native tool history instead of the REPL wrapper

So the transcript is a curated persisted view, not a byte-for-byte mirror of every in-memory message.

## Resume Reconstructs More Than Messages

`utils/sessionRestore.ts` shows that resume restores several runtime stores from transcript/log data.

Verified restored state includes:

- file history snapshots
- attribution snapshots
- context-collapse commit log and snapshot
- Todo state for non-v2 todo mode
- agent setting and model override
- standalone agent name/color context
- session metadata such as title, tag, PR linkage, and mode
- worktree session state
- content replacements used for tool-result restoration

So resume is not "load messages and continue". It rebuilds multiple pieces of runtime state around those messages.

## Resume And Forked Resume Behave Differently

`processResumedConversation(...)` treats normal resume and forked resume differently.

Verified behavior:

- normal resume switches to the old session ID
- normal resume adopts the existing session file and restores worktree state
- forked resume keeps the fresh session ID
- forked resume seeds content replacements into the new session so restored tool-result behavior still works
- forked resume strips inherited worktree ownership from session metadata

That means `--fork-session` is not only "same resume but new id". It changes how transcript ownership and related metadata are handled.

## Shutdown Runs SessionEnd Hooks Under A Tight Budget

`utils/gracefulShutdown.ts` and `utils/hooks.ts` make shutdown priorities explicit.

Verified behavior:

- terminal cleanup and resume hint happen before async cleanup work
- general cleanup functions run before session-end hooks
- `SessionEnd` hooks run with a bounded timeout budget
- the default session-end hook timeout is much lower than the normal tool-hook timeout
- shutdown ignores session-end hook errors and timeouts
- `executeSessionEndHooks(...)` clears session hooks after execution when `setAppState` is available

This is a useful correction to the simplistic mental model of "hooks run during shutdown like normal hooks". The shutdown path is deliberately tighter and failure-tolerant.

## SessionEnd Hooks Also Fire On `/clear`

This is worth calling out separately because it is easy to miss:

- graceful shutdown runs `SessionEnd`
- `/clear` also runs `SessionEnd('clear')`

So "session end" in the code means lifecycle end of the current session context, not only process exit.

## The Code Distinguishes Session Persistence From Session UI

The code paths here repeatedly reinforce a larger architecture point:

- REPL/UI state is not the source of truth for resume
- transcript persistence is
- hooks can influence the session lifecycle, but they do so through explicit typed events and persisted messages

This is why `QueryEngine`, `sessionStorage`, `sessionRestore`, `conversationRecovery`, `clearConversation`, and `gracefulShutdown` all matter to the same subsystem story.

## Corrections To The Existing Mental Model

- Hooks are not only settings-defined shell commands; they can be callback, function, prompt, agent, and HTTP hooks, and they come from settings, plugins, SDK registration, and session-scoped frontmatter.
- Resume does not re-fire startup hooks. It fires `SessionStart` with source `resume`.
- `/clear` is not only a UI reset; it runs `SessionEnd`, regenerates session identity, and then runs `SessionStart` with source `clear`.
- Session persistence starts earlier than "assistant replied": accepted user messages are written before the query loop proceeds so killed requests remain resumable.
- The persisted transcript is filtered and sometimes transformed; it is not a raw dump of every in-memory UI message.
- Shutdown hook execution is intentionally bounded and more restrictive than ordinary in-turn hook execution.
