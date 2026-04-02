# Agent, Workflow, and Bridge Worktree Consumers

## Scope

This document covers the subsystems that consume the shared agent/workflow worktree helpers rather than the worktree subsystem itself.

Primary files read for this slice:

- `tools/AgentTool/AgentTool.tsx`
- `tools/AgentTool/runAgent.ts`
- `tools/AgentTool/resumeAgent.ts`
- `tools/AgentTool/agentToolUtils.ts`
- `bridge/bridgeMain.ts`
- `utils/worktree.ts`
- `entrypoints/cli.tsx`

## This Slice Is About Consumers, Not The Core Helper

The previous worktree document covered creation, persistence, and cleanup mechanics in `utils/worktree.ts`.

This slice answers a different question:

- which runtime features actually create agent-style worktrees
- what isolation those callers enforce
- when they keep versus remove the worktree
- how they surface kept worktrees back to users or resumed tasks

The code shows that not all consumers use the helper in the same way.

## The Main Consumer Split

From the code in this workspace, the important consumers are:

- `AgentTool`
- bridge multi-session worktree mode in `bridgeMain.ts`

There are also verified references to:

- workflow worktrees via `wf_<runId>-<idx>` stale-cleanup patterns
- template job worktrees via `job-<template>-<8hex>` stale-cleanup patterns

But the owning workflow/template source files are not present in this snapshot, so those two are documented here only from shared helper comments and call-site contracts.

## `AgentTool` Opts Into Worktree Isolation Per Spawn

`AgentTool.tsx` does not always create a worktree.

Verified behavior:

- it computes an effective isolation mode
- only when that mode resolves to `worktree` does it call `createAgentWorktree(...)`
- the slug format is `agent-${earlyAgentId.slice(0, 8)}`

So agent worktrees are an optional spawn-time isolation choice, not a universal property of all subagents.

## Agent Worktree State Stays Local To The Spawn

When `AgentTool` creates a worktree, it stores a small local object:

- `worktreePath`
- `worktreeBranch`
- `headCommit`
- `gitRoot`
- `hookBased`

This is intentionally separate from:

- session-level `currentWorktreeSession`
- project-config `activeWorktreeSession`
- transcript `worktree-state`

So agent worktree ownership is local to the agent run, not promoted into the main session lifecycle.

## Agent Isolation Affects Prompting, CWD, And Metadata

Once a worktree is present, `AgentTool` uses it in three different ways.

Verified effects:

- `runAgent(...)` receives `worktreePath`
- the actual agent run is wrapped in `runWithCwdOverride(...)`
- fork-style agents get an extra worktree notice telling the child to translate paths and re-read possibly stale files

That means agent worktree isolation is not just "run commands elsewhere". It also changes the model-visible context and the persisted agent metadata.

## `runAgent(...)` Persists Worktree Location For Resume

`runAgent.ts` writes sidechain agent metadata before the query loop begins.

Verified metadata fields written there include:

- `agentType`
- `worktreePath` when present
- `description` when present

That makes `worktreePath` part of the subagent resume contract rather than only a transient runtime override.

## Agent Cleanup Is Change-Sensitive, Not Unconditional

This is the biggest consumer-specific difference from bridge sessions.

`AgentTool.tsx` builds a `cleanupWorktreeIfNeeded(...)` helper with this logic:

- if there is no worktree, return nothing
- if the worktree is hook-based, always keep it
- if it has a `headCommit`, call `hasWorktreeChanges(...)`
- if no changes are detected, call `removeAgentWorktree(...)`
- if changes exist, keep the worktree and return its path/branch

So agent worktree cleanup is content-sensitive. The worktree is disposable only when the agent left no local state worth preserving.

## Hook-Based Agent Worktrees Are Always Kept

For agent worktrees, hook mode has a stricter keep policy than git mode.

Verified reason in code:

- the runtime cannot reliably detect VCS changes for an arbitrary hook backend

So a hook-created agent worktree is treated as non-disposable and is surfaced back as preserved state instead of being auto-removed.

## Agent Cleanup Is Intentionally Idempotent

`cleanupWorktreeIfNeeded(...)` nulls out the captured `worktreeInfo` before doing further work.

That matters because the same cleanup closure can be reached from:

- success
- abort
- failure
- sync-agent finally paths

So the consumer layer explicitly defends against double cleanup if later code throws after one cleanup attempt already ran.

## Kept Agent Worktrees Are Reflected Back To The Parent

When an agent finishes and its worktree is preserved, that information is not thrown away.

Verified behavior:

- `cleanupWorktreeIfNeeded(...)` returns `{ worktreePath, worktreeBranch }`
- `runAsyncAgentLifecycle(...)` forwards that into the completion/killed/failed notification path
- sync completion also merges `worktreeResult` into the returned tool result
- `AgentTool` appends `worktreePath` and `worktreeBranch` into the textual tool-result metadata block when present

So kept worktrees are a first-class operator-visible outcome, not just a debug detail.

## Async Agent State Transitions Do Not Wait On Worktree Cleanup

`agentToolUtils.ts` makes this ordering explicit.

Verified sequence:

- the task is marked completed, killed, or failed first
- only after that does the code await `getWorktreeResult()`
- the worktree result is treated as notification embellishment, not as the status transition gate

This is an important design choice:

- a git hang during cleanup must not block `TaskOutput(block=true)` or the user-visible task status transition

So worktree cleanup is downstream of task completion semantics.

## Sync And Async Agents Share The Same Cleanup Policy

The sync and async paths differ operationally, but they intentionally share the same worktree decision logic.

Verified pattern:

- async agents pass `cleanupWorktreeIfNeeded` into `runAsyncAgentLifecycle(...)`
- sync agents call the same helper from their `finally` block unless the task has been backgrounded

So the consumer split is about lifecycle timing, not about different keep/remove rules.

## Backgrounded Agents Delay Cleanup Ownership

There is one important exception in the sync path.

When a foreground agent is backgrounded:

- the original sync path skips cleanup
- the background continuation becomes the owner of the worktree lifecycle

That prevents a worktree from being removed while the detached agent is still running inside it.

## Resumed Agents Reuse Preserved Worktrees Best-Effort

`resumeAgent.ts` shows how subagent resume treats worktrees.

Verified behavior:

- if metadata has a `worktreePath`, the code stats it first
- if the directory is gone, resume falls back to the parent cwd
- if the directory exists, its mtime is bumped so stale cleanup will not remove it
- resumed runs pass `worktreePath` back into `runAgent(...)`
- resumed execution is wrapped in `runWithCwdOverride(resumedWorktreePath, ...)`

So subagent resume treats preserved worktrees as reusable execution environments, but only on a best-effort basis.

## Resumed Agents Do Not Re-Run The Original Cleanup Heuristic

This is a subtle but important consumer difference.

For resumed agents, `resumeAgent.ts` passes:

- `getWorktreeResult: async () => resumedWorktreePath ? { worktreePath: resumedWorktreePath } : {}`

It does not call:

- `hasWorktreeChanges(...)`
- `removeAgentWorktree(...)`

So resumed subagents preserve and report the existing worktree, but this code path does not reapply the original change-sensitive auto-removal heuristic.

## Bridge Sessions Use Worktrees Differently From Agents

`bridgeMain.ts` uses the same helper family but with a different policy model.

Verified behavior:

- bridge worktree mode only applies in multi-session spawning
- the pre-created initial session does not get an isolated worktree
- on-demand additional sessions do
- each such session gets a slug `bridge-${safeFilenameId(sessionId)}`
- the resulting worktree path becomes the session spawn directory

So bridge worktrees are a concurrency-isolation mechanism for extra sessions, not a per-agent execution sandbox.

## Bridge Cleanup Is Unconditional

Unlike `AgentTool`, bridge sessions do not inspect for changes before cleanup.

Verified cleanup points:

- spawn failure after worktree creation
- normal session completion
- bridge shutdown cleanup of all remaining active-session worktrees

Each of those paths directly calls `removeAgentWorktree(...)`.

So the bridge consumer policy is:

- session isolation is temporary
- cleanup is unconditional
- worktrees are not surfaced back as preserved artifacts

This is a much more disposable model than the agent consumer path.

## Bridge Tracks Worktree Ownership In A Session Map

`bridgeMain.ts` keeps a `sessionWorktrees` map keyed by session ID, with values:

- `worktreePath`
- `worktreeBranch`
- `gitRoot`
- `hookBased`

That map is the bridge-specific ownership layer.

It allows bridge cleanup to happen from:

- per-session completion handlers
- spawn-failure handling
- global shutdown

without relying on shared global session worktree state.

## Bridge Analytics Treat Worktree Creation As A First-Class Metric

The bridge spawn path records:

- `spawn_mode`
- `in_worktree`
- `worktree_create_ms`

That reflects an important product-level fact from the code:

- worktree isolation is not incidental in bridge mode
- it is a major runtime branch with explicit telemetry and UX consequences

## Workflow And Template Job Consumers Are Only Partially Visible Here

`utils/worktree.ts` explicitly documents stale slug patterns for:

- workflow worktrees: `wf_<runId>-<idx>` and legacy `wf-<idx>`
- template job worktrees: `job-<templateName>-<8hex>`

And `entrypoints/cli.tsx` has a fast path for template-job commands through:

- `../cli/handlers/templateJobs.js`

But the owning workflow/template modules are not present in this workspace snapshot.

So what is verified from available code is:

- these subsystems are real consumers of the shared stale-cleanup mechanism
- their leaked worktrees are expected enough to have exact cleanup patterns
- template jobs are first-class enough to have an early CLI fast path

What is not verified here from source internals:

- their exact worktree creation sites
- whether they keep or auto-remove on success
- how they surface preserved worktrees back to users

## The Shared Stale-Cleanup Layer Is Consumer-Agnostic

`cleanupStaleAgentWorktrees(...)` is the backstop when a consumer dies before it can run its normal in-process cleanup.

This means the runtime has two layers of cleanup for agent-style worktrees:

- consumer-specific in-process policy
- shared delayed garbage collection for leaked ephemeral slugs

That distinction matters:

- `AgentTool` may intentionally keep a changed worktree
- bridge may intentionally remove on completion
- stale cleanup only handles abandoned ephemeral leftovers that survived both

## Source-Of-Truth Summary

The code’s actual consumer model is:

- `AgentTool` creates optional isolated worktrees per spawn, persists the path into agent metadata, and removes them only when unchanged
- resumed agents reuse preserved worktrees best-effort and keep reporting them, but do not re-run the original auto-removal heuristic here
- bridge multi-session mode creates temporary per-session worktrees for additional sessions and removes them unconditionally on failure, completion, or shutdown
- workflow and template-job worktrees are definitely part of the shared cleanup contract, but their owning source modules are missing from this snapshot

The key correction to a simplified mental model is that the shared helper does not imply shared policy. Different consumers use the same worktree primitive for different isolation contracts and cleanup rules.
