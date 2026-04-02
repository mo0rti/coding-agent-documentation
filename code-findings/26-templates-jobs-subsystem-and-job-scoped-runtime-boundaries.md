# Templates/Jobs Subsystem and Job-Scoped Runtime Boundaries

## Scope

This document covers the parts of the templates/jobs subsystem that are directly verifiable from this workspace snapshot: CLI dispatch, config-surface integration, job-scoped permission behavior, turn-time state classification, and related security/runtime boundaries.

Primary files read for this slice:

- `entrypoints/cli.tsx`
- `query/stopHooks.ts`
- `utils/permissions/filesystem.ts`
- `utils/markdownConfigLoader.ts`
- `utils/subprocessEnv.ts`

## Snapshot Caveat

The jobs/templates subsystem is only partially present in this workspace.

Verified missing modules from call sites:

- `../cli/handlers/templateJobs.js`
- `../jobs/classifier.js`
- the `jobs/` source directory itself

So this document is intentionally limited to verified runtime contracts around those modules, not their unseen implementation details.

## Templates/Jobs Have A Dedicated CLI Fast Path

`entrypoints/cli.tsx` treats template jobs as a first-class early-dispatch surface.

Verified behavior:

- when `feature('TEMPLATES')` is enabled
- and the first CLI arg is `new`, `list`, or `reply`
- the process imports `../cli/handlers/templateJobs.js`
- runs `templatesMain(args)`
- then exits the process directly

This means template jobs do not flow through the normal generic Commander path after startup. They are handled more like a specialized mode.

## Templates Are Part Of The Config/Content Surface

`utils/markdownConfigLoader.ts` includes `templates` in `CLAUDE_CONFIG_DIRECTORIES` when the feature gate is on.

Verified directory set includes:

- `commands`
- `agents`
- `output-styles`
- `skills`
- `workflows`
- `templates` when `TEMPLATES` is enabled

That means templates are treated as repo/user config content alongside other Claude-managed markdown-backed surfaces, not as an unrelated external subsystem.

## Jobs Use A Dedicated Runtime Directory Root

The filesystem permission logic establishes a clear expected root for jobs:

- `~/.claude/jobs`

`utils/permissions/filesystem.ts` does not trust `CLAUDE_JOB_DIR` blindly.

Verified checks:

- it reads `process.env.CLAUDE_JOB_DIR`
- it computes normalized lexical and resolved path forms for both the job dir and `~/.claude/jobs`
- every resolved form of the job dir must remain under some resolved form of the jobs root
- only then can the job-dir write carve-out apply

So `CLAUDE_JOB_DIR` is treated as an attacker-influenced input that must be anchored under the Claude jobs root before it can grant anything.

## Job-Scoped Write Permission Is Narrow And Explicit

Once the jobs-root anchoring passes, the permission carve-out is still narrow.

Verified behavior:

- a target path is only auto-allowed when all of its checked path forms stay inside the current job directory
- the resulting decision reason is:
  - `Job directory files for current job are allowed for writing`

This is not a blanket "job mode can write anywhere" rule.

It is:

- current job only
- inside the validated job directory only
- still mediated through the normal filesystem permission decision function

## The Job Directory Carve-Out Is A Security Boundary, Not A Convenience Feature

The comments in `filesystem.ts` make the intent explicit.

Verified threat model covered there:

- env-var hijack attempts
- symlink escapes from inside the job dir to arbitrary targets
- symlinked Claude config roots such as macOS `/tmp` to `/private/tmp`

So the job carve-out exists because jobs need local write capability inside their own workspace, but the implementation is designed to avoid turning `CLAUDE_JOB_DIR` into a generic privilege-escalation channel.

## Job State Classification Runs At Turn End

`query/stopHooks.ts` adds a special job-mode hook at the end of a main-thread turn.

Verified conditions:

- `feature('TEMPLATES')` must be enabled
- `process.env.CLAUDE_JOB_DIR` must be set
- `querySource` must start with `repl_main_thread`
- `toolUseContext.agentId` must be absent

When those conditions hold, the runtime:

- gathers assistant messages from the full turn message set
- calls `jobClassifierModule!.classifyAndWriteState(process.env.CLAUDE_JOB_DIR, turnAssistantMessages)`
- logs any error as `[job] classifier error: ...`
- awaits the result, but races it against a `60_000ms` timeout promise

So jobs have a per-turn state-write side effect that is part of stop-hook processing, not part of the main query loop body.

## Classification Is Scoped To The Main Thread

The stop-hook comments make an important boundary explicit.

The classifier is intentionally gated away from:

- background forks
- extract-memories work
- auto-dream work
- agent-side turns

The stated reason is that background activity should not pollute job state with unrelated assistant messages.

So job state is modeled as the main-thread conversation’s state, not as a union of every subprocess or subagent activity that happened during the turn.

## `state.json` Freshness Matters To The Runtime Contract

The stop-hook comments also reveal a user-facing dependency:

- classification is awaited so `state.json` is written before the turn returns
- otherwise `claude list` would show stale state during the gap

Even though the implementation module is missing here, that contract is still source-of-truth:

- job state persistence is expected to be visible to a list/read UI immediately after the turn

So this classifier is not just telemetry or best-effort enrichment. It is part of the job UX state machine.

## The Runtime Avoids Static Imports For Jobs On Purpose

Both the classifier usage and the comments in `stopHooks.ts` show that jobs are kept behind require-gated imports.

Verified rationale from comments:

- avoid leaking jobs-specific strings into builds where the feature is off
- keep tree-shaking effective
- avoid importing job constants directly just to read env-key names

So jobs/templates are implemented as a gated subsystem whose code footprint is intentionally removable from some builds.

## Job Env Usage Is Reflected In Subprocess Security Policy

`utils/subprocessEnv.ts` contains a job-relevant exception:

- `GITHUB_TOKEN` / `GH_TOKEN` are intentionally not scrubbed

Verified reason in comments:

- wrapper scripts such as `gh.sh` need them for GitHub API calls
- the token is treated as job-scoped and expiring with the workflow

This is a meaningful boundary:

- many secrets are scrubbed from subprocesses
- GitHub workflow/job tokens are intentionally retained for job-driven tooling

So jobs/templates influence subprocess-env policy, not just top-level CLI dispatch.

## The Subsystem Is Tied To Workflow-Oriented Content Surfaces

Even without the missing jobs source files, two code-level signals connect templates/jobs to workflow-style execution:

- `markdownConfigLoader.ts` includes both `workflows` and gated `templates`
- `sessionStorage.ts` comments describe workflow-related subagent transcript grouping under `subagents/workflows/<runId>/`

That does not prove the full internal jobs implementation here, but it does show the surrounding runtime is already shaped to support authored workflow/template content plus execution state.

## What Is Verified Vs. Not Verified

Verified from code in this snapshot:

- template jobs are early-dispatched from the CLI
- templates are a gated markdown-backed config surface
- jobs use `CLAUDE_JOB_DIR` as a runtime anchor
- writes inside the current validated job dir are auto-allowed
- per-turn job state classification runs from stop hooks on main-thread turns
- the classifier writes job state synchronously enough to keep list views fresh
- subprocess policy intentionally keeps GitHub workflow/job tokens available

Not verified from source internals in this snapshot:

- how `templatesMain(...)` parses or executes commands
- the shape of job state files beyond the `state.json` comment
- how templates are authored or rendered
- how jobs create their worktrees
- how `classifyAndWriteState(...)` decides statuses

## Source-Of-Truth Summary

The code that is present shows templates/jobs as a gated but real runtime subsystem with four important boundaries:

- a dedicated CLI entry surface
- a dedicated config content surface
- a dedicated job-directory permission boundary
- a dedicated turn-end state-classification hook

The important correction to a shallow reading is that jobs are not just “some CLI command behind a feature flag”. Even in this partial snapshot, the runtime has explicit security, persistence, and UX contracts for job-scoped execution.
