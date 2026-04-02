# Worktree Creation, Isolation, and Cleanup Semantics

## Scope

This document covers the worktree subsystem itself: slug validation, git and hook-based creation, post-creation setup, session-vs-agent lifecycle, persisted worktree state, and cleanup rules.

Primary files read for this slice:

- `utils/worktree.ts`
- `tools/EnterWorktreeTool/EnterWorktreeTool.ts`
- `tools/ExitWorktreeTool/ExitWorktreeTool.ts`
- `setup.ts`
- `utils/sessionStorage.ts`
- `utils/sessionRestore.ts`
- `types/logs.ts`
- `utils/config.ts`

## The Worktree Layer Is More Than `git worktree add`

The code treats a worktree as a compound runtime feature, not just a git command.

Verified layers:

- name validation and path-safety checks
- git-backed or hook-backed creation
- optional sparse-checkout application
- post-creation repo shaping
- session state and transcript persistence
- exit semantics that distinguish keep vs remove
- separate agent/workflow worktree lifecycle and stale cleanup

So the real feature is "isolated execution environment with session bookkeeping", not just "create another checkout".

## Slugs Are Validated Before Any Side Effects

`validateWorktreeSlug(...)` is a hard precondition.

Verified constraints:

- maximum length is `64`
- `/` is allowed syntactically, but each segment must be non-empty
- `.` and `..` path segments are rejected
- only letters, digits, `.`, `_`, and `-` are allowed in each segment
- validation runs before hook execution, git commands, or `chdir`

This is important because worktree names flow into both filesystem paths and branch names.

## Nested Slugs Are Flattened For Both Path And Branch Safety

The code allows slash-separated slugs at the API boundary, but it does not preserve that nesting in the actual git worktree path or branch name.

`flattenSlug(...)` converts `/` to `+`, and both helpers use the flattened form:

- `worktreeBranchName(slug)` -> `worktree-<flattened>`
- `worktreePathFor(repoRoot, slug)` -> `<repo>/.claude/worktrees/<flattened>`

The code comments call out two concrete reasons:

- git ref D/F conflicts for names like `worktree-user` vs `worktree-user/feature`
- unsafe nested directories where removing a parent worktree could delete a child

So slash-separated names are user-facing input syntax, not persistent directory nesting.

## Git Worktrees Live Under `.claude/worktrees`

For git-backed worktrees, the runtime places them under:

- `<git-root>/.claude/worktrees/<flattened-slug>`

That location is part of the subsystem contract, not an incidental convenience.

It keeps Claude-managed worktrees grouped under repo-local state and gives cleanup logic one predictable place to scan.

## Hook Backends Take Precedence Over Git

Both session worktrees and agent worktrees first check for `WorktreeCreate` hooks.

Verified behavior:

- if a create hook exists, the runtime delegates creation to the hook
- if no hook exists, it falls back to git worktrees
- cleanup mirrors the same split through `WorktreeRemove`
- hook mode is the intended extension point for non-git or alternate VCS backends

This means "worktree support" in this codebase is explicitly broader than git worktrees.

## Git Creation Has A Fast Resume Path

`getOrCreateWorktree(...)` does not blindly recreate or refetch.

Verified behavior:

- it first reads the worktree `.git` pointer / HEAD state directly through `readWorktreeHeadSha(...)`
- if the worktree already exists, it returns a resume result without fetch or `git worktree add`
- only missing worktrees go through base resolution and creation

That fast path is load-bearing because a normal git fetch can block on credentials and adds noticeable startup cost.

## Base Resolution Is Conservative And PR-Aware

For new git worktrees, the base branch is resolved in two different ways:

- PR mode fetches `pull/<pr>/head` and uses `FETCH_HEAD`
- normal mode prefers an already-present `origin/<defaultBranch>` ref without fetching

If the origin ref is not available locally, only then does it fetch.

The code is intentionally optimized for "good enough and local-first", not "always fetch latest before every worktree".

## Sparse Checkout Is Part Of Creation, Not A Separate Feature

If `settings.worktree.sparsePaths` is set, creation changes behavior:

- `git worktree add --no-checkout` is used
- sparse-checkout is configured in the new worktree
- `git checkout HEAD` is run afterward

Most importantly, failures in sparse setup trigger immediate teardown of the new worktree.

That prevents a half-created sparse worktree from being mistaken for a healthy resumed one on the next run.

## Post-Creation Setup Shapes The Runtime Environment

`performPostCreationSetup(...)` does the operational setup after git creation.

Verified responsibilities:

- copy local Claude settings into the worktree
- configure git hooks path toward the main repo hooks or `.husky`
- symlink configured directories from the main repo to avoid duplication
- copy selected gitignored files listed in `.worktreeinclude`
- install the attribution hook into local `.husky` when needed

This is why the worktree feature is an execution-isolation layer rather than a bare checkout clone.

## `.worktreeinclude` Is Narrower Than "Copy Untracked Files"

The `.worktreeinclude` behavior is intentionally restrictive.

Files are only copied when they are both:

- matched by `.worktreeinclude`
- gitignored

Implementation detail that matters:

- the code uses `git ls-files --others --ignored --exclude-standard --directory`
- fully ignored directories are collapsed first for performance
- only targeted collapsed directories are expanded again when patterns require it

So this path is optimized for safely bringing over selected local-only files like secrets or generated config, not for mirroring the entire dirty working tree.

## Startup `--worktree` And Mid-Session `EnterWorktree` Are Not The Same

The helpers overlap, but the entry paths intentionally differ.

For startup `--worktree` in `setup.ts`:

- if running inside an existing worktree, the code first resolves the canonical main repo root
- it `chdir`s back to that main repo before calling `createWorktreeForSession(...)`
- after creation it sets `projectRoot` to the new worktree path
- hook config snapshots are refreshed from the worktree

For mid-session `EnterWorktreeTool`:

- it directly calls `createWorktreeForSession(...)`
- it switches `cwd` and `originalCwd` to the worktree
- it saves transcript worktree state
- it clears prompt and memory caches
- it does not reset `projectRoot`

That split is deliberate: startup `--worktree` makes the worktree the session's project identity, while `EnterWorktree` is a temporary execution shift inside an existing project session.

## Session Worktrees And Agent Worktrees Use Different Root Rules

This is one of the most important implementation details in the subsystem.

`createWorktreeForSession(...)` uses:

- `findGitRoot(getCwd())`

`createAgentWorktree(...)` uses:

- `findCanonicalGitRoot(getCwd())`

The agent path is explicitly documented in code as a fix for nested cleanup problems: agent worktrees must always land under the main repo's `.claude/worktrees/`, even when spawned from inside a session worktree.

So the runtime does not use one single "canonical worktree root" rule everywhere. Session entry paths and agent/workflow paths are intentionally different.

## Session Worktree State Lives In Two Places

The code persists worktree state in two separate layers:

- project config: `activeWorktreeSession`
- transcript metadata: `worktree-state`

These are not the same thing.

`activeWorktreeSession` in `ProjectConfig` is updated when a session worktree is created, kept, or cleaned up.

But transcript persistence is the real resume contract:

- `saveWorktreeState(...)` writes a tri-state `worktree-state` entry
- `undefined` means "never entered a managed worktree"
- object means "currently in a managed worktree"
- `null` means "explicitly exited the managed worktree"

That tri-state behavior is why the system can distinguish "crashed while still inside" from "user exited cleanly before shutdown".

## Transcript Persistence Intentionally Strips Ephemeral Fields

`saveWorktreeState(...)` does not serialize the full `WorktreeSession`.

It strips runtime-only fields like:

- `creationDurationMs`
- `usedSparsePaths`

So the transcript stores resume-relevant identity and location, not analytics-only metadata.

## Resume Restores The Worktree Carefully

`restoreWorktreeForResume(...)` is the verified resume boundary.

Verified behavior:

- if a fresh worktree already exists in the current process, that fresh state wins
- otherwise it only restores when transcript `worktreeSession` is non-null
- it uses `process.chdir(...)` as the existence check
- if the worktree path is gone, it writes `saveWorktreeState(null)` so stale state is not re-persisted
- if the path exists, it restores `cwd`, `originalCwd`, and module-level `currentWorktreeSession`

Important nuance:

- resume intentionally does not set `projectRoot`
- the transcript does not record whether the prior worktree came from startup `--worktree` or mid-session `EnterWorktree`
- the code chooses the safer stable-project interpretation

So resume restores execution location and worktree ownership, but not the stronger startup-only project-root rewrite.

## Keep And Remove Have Different Semantics

`keepWorktree()` and `cleanupWorktree()` are not two UI labels for the same operation.

`keepWorktree()`:

- changes back to `originalCwd`
- clears `currentWorktreeSession`
- clears `activeWorktreeSession`
- leaves the worktree directory and branch intact

`cleanupWorktree()`:

- changes back to `originalCwd`
- removes the hook-based or git-based worktree
- clears `currentWorktreeSession`
- clears `activeWorktreeSession`
- attempts to delete the temporary worktree branch after removal

So "keep" means "drop ownership but preserve environment", while "remove" means "destroy the managed environment and branch".

## Exit Safety Is Fail-Closed

`ExitWorktreeTool` does extra safety work before removal.

Verified behavior:

- it only operates when `getCurrentWorktreeSession()` is set
- it refuses to touch manually created or old-session worktrees
- remove without `discard_changes` first counts dirty files and new commits
- if git state cannot be verified, removal is refused unless explicitly forced

That means destructive exit only applies to worktrees the current session created or restored as its own managed worktree.

## The Tool Layer Restores More Than `cwd`

`ExitWorktreeTool` restores the higher-level session view after keep/remove:

- resets `cwd`
- resets `originalCwd`
- clears transcript worktree state with `saveWorktreeState(null)`
- clears prompt and memory caches
- restores `projectRoot` only when startup `--worktree` had changed it
- refreshes hook snapshots only in that startup-style case

So exiting a worktree is not a plain directory change. It is a coordinated reversal of session-layer mutations.

## Agent Worktrees Are Lightweight And Disposable

`createAgentWorktree(...)` reuses creation helpers but intentionally avoids session-level state.

Verified behavior:

- no `currentWorktreeSession`
- no `process.chdir`
- no project-config update
- no transcript `worktree-state`

If an agent worktree already exists, its mtime is bumped so stale-cleanup logic does not incorrectly remove an active resumed workspace.

This is a separate lifecycle from human session worktrees.

## Stale Cleanup Only Targets Ephemeral Worktrees

`cleanupStaleAgentWorktrees(...)` is intentionally narrow.

It only touches names matching ephemeral system patterns such as:

- `agent-a<7hex>`
- `wf_<runId>-<idx>`
- legacy `wf-<idx>`
- `bridge-...`
- `job-<template>-<8hex>`

It skips user-named worktrees entirely.

It also skips cleanup when:

- the target is the current session worktree
- the directory is newer than the cutoff
- `git status` fails
- tracked changes exist
- unpushed commits exist

After successful removals it runs `git worktree prune`.

So periodic cleanup is designed for abandoned automation worktrees, not general garbage collection of all Claude-created worktrees.

## The Early `--tmux --worktree` Path Is A Separate Entrypoint

`execIntoTmuxWorktree(...)` is not just a thin wrapper around normal startup.

Verified behavior:

- validates or generates the slug early
- supports PR references
- prefers hooks before git
- uses `findCanonicalGitRoot(...)` for git placement
- creates or resumes the worktree before execing the inner Claude process
- can use tmux control mode for iTerm2
- has a repo-specific dev-pane special case for internal builds

So the tmux path is its own optimized worktree-launch entrypoint, not simply "run normal startup and then attach tmux later".

## Source-Of-Truth Summary

The code’s actual worktree model is:

- safe slug -> managed directory/branch mapping
- hook-first, git-second backend choice
- reusable create-or-resume behavior
- post-create repo shaping for settings, hooks, symlinks, and copied ignored files
- different lifecycle rules for startup sessions, mid-session worktrees, and agent worktrees
- transcript-backed last-wins worktree state for resume
- fail-closed destructive exit and stale cleanup

The important correction to simplified architecture notes is that "worktree" here is a full isolation and lifecycle subsystem with multiple entrypoints, not a single git utility wrapper.
