# Workflow Runtime, Transcript Grouping, and Orchestration Boundaries

## Scope

This document covers the workflow-specific runtime contracts that are directly verifiable from this workspace snapshot: task typing, UI/task integration, SDK event shape, transcript grouping, workflow command surfacing, and nearby orchestration boundaries.

Primary files read for this slice:

- `tasks/types.ts`
- `tasks/pillLabel.ts`
- `components/tasks/BackgroundTask.tsx`
- `components/tasks/BackgroundTasksDialog.tsx`
- `utils/task/framework.ts`
- `utils/task/sdkProgress.ts`
- `utils/sessionStorage.ts`
- `tools/AgentTool/runAgent.ts`
- `utils/suggestions/commandSuggestions.ts`
- `utils/hooks/sessionHooks.ts`
- `utils/worktree.ts`
- `entrypoints/sdk/coreSchemas.ts`
- `components/permissions/PermissionRequest.tsx`

## Snapshot Caveat

The workflow subsystem is only partially present in this workspace snapshot.

Verified missing or indirect modules from imports/call sites:

- `tasks/LocalWorkflowTask/LocalWorkflowTask.js`
- `components/tasks/WorkflowDetailDialog.js`
- `tools/WorkflowTool/WorkflowTool.js`

So this document is intentionally about verified runtime boundaries around workflows, not the unseen internal algorithm of the workflow executor itself.

## Workflows Are A First-Class Task Type

`tasks/types.ts` includes `LocalWorkflowTaskState` in both:

- `TaskState`
- `BackgroundTaskState`

That means the surrounding runtime does not treat workflows as a disguised local agent or a generic shell task. They have a dedicated task identity in the app-state union.

## The Task UI Treats Workflows As Their Own Operator Surface

The task UI has explicit workflow branches rather than a fallback generic renderer.

Verified behavior:

- `tasks/pillLabel.ts` renders `local_workflow` as `1 background workflow` / `N background workflows`
- `components/tasks/BackgroundTask.tsx` uses `workflowName ?? summary ?? description` as the label source
- that same component renders running workflows with an `agentCount`-based status label
- `components/tasks/BackgroundTasksDialog.tsx` gives workflows their own grouped `Workflows` section
- workflows can be selected, viewed, and stopped from the background-task dialog

So workflows are a distinct operator-facing runtime surface, not just internal plumbing.

## Workflow Detail And Control Are Build-Gated

The workflow UI/control path is behind `feature('WORKFLOW_SCRIPTS')`.

Verified pattern in `BackgroundTasksDialog.tsx`:

- `WorkflowDetailDialog` is loaded via gated `require(...)`
- the workflow task module is also loaded via gated `require(...)`
- `killWorkflowTask`
- `skipWorkflowAgent`
- `retryWorkflowAgent`

This matters for interpreting the snapshot:

- workflows are clearly supported by the surrounding runtime
- but some owning workflow modules are intentionally absent in builds where the feature is compiled out

So the missing files here are partly a snapshot/build-shape issue, not just a documentation gap.

## Workflows Have Dedicated Task-Started Metadata

`utils/task/framework.ts` shows that workflow tasks contribute extra metadata when registered.

Verified behavior for `task_started` SDK events:

- every task emits `task_id`, `description`, and `task_type`
- if the task has `workflowName`, that value is emitted as `workflow_name`
- if the task has `prompt`, that value is emitted as `prompt`

`entrypoints/sdk/coreSchemas.ts` makes the contract even more explicit:

- `workflow_name` is described as `meta.name from the workflow script`
- it is only expected when `task_type` is `local_workflow`

So workflow naming is part of the formal SDK/task contract, not just UI decoration.

## Workflow Progress Has Its Own Structured Event Shape

`utils/task/sdkProgress.ts` shows that workflow progress is not flattened into plain text summaries.

Verified behavior:

- `emitTaskProgress(...)` accepts `workflowProgress?: SdkWorkflowProgress[]`
- that value is emitted as `workflow_progress`
- comments explicitly say the helper is shared by agents and workflows
- the workflow side provides its own state shape, rather than pretending to be an agent `ProgressTracker`

So workflows have a structured progress channel in the SDK/event stream, not merely generic task output.

## Workflow Runs Group Related Subagent Transcripts Under A Dedicated Subdirectory

The transcript-storage layer makes workflow grouping explicit.

Verified behavior in `utils/sessionStorage.ts`:

- there is an in-memory `agentTranscriptSubdirs` map
- comments explicitly say workflow runs write to `subagents/workflows/<runId>/`
- `getAgentTranscriptPath(...)` inserts that subdirectory before `agent-<agentId>.jsonl`

`tools/AgentTool/runAgent.ts` provides the write-side contract:

- callers can pass `transcriptSubdir?: string`
- comments name `workflows/<runId>` as the example grouping
- `setAgentTranscriptSubdir(agentId, transcriptSubdir)` is called before the agent run begins

So workflow transcript topology is not ad hoc. The storage layer explicitly supports grouping all workflow-spawned subagents under one workflow-run namespace.

## Workflow Orchestration Reuses Agent Execution Rather Than Replacing It

The transcript-grouping contract above reveals an important architectural boundary.

Because `runAgent.ts` supports:

- per-agent transcript subdirectories
- normal agent query execution
- normal agent metadata persistence

the workflow layer appears to orchestrate multiple agent runs rather than introducing a completely separate transcript format or execution substrate.

What is verified here is the boundary, not the hidden workflow scheduler:

- workflow orchestration can group related child agents
- those child agents still use the standard agent runtime underneath

## Workflow Commands Are A Distinct Prompt-Command Kind

`utils/suggestions/commandSuggestions.ts` treats workflows as a dedicated slash-command category.

Verified behavior:

- `cmd.type === 'prompt' && cmd.kind === 'workflow'` is detected explicitly
- workflow suggestions get `tag: 'workflow'`
- their descriptions use the raw workflow description path instead of the normal source-formatted command description

So authored workflows are part of the command surface as their own command kind, not just as hidden internal automation.

## Workflow Runtime Likely Manages Many Concurrent Agent Hooks

`utils/hooks/sessionHooks.ts` contains one of the clearest indirect design clues about workflow scale.

Verified comment signals:

- the session-hooks container uses a `Map` for O(1) mutation
- the comment compares this to `agentControllers on LocalWorkflowTaskState`
- it explicitly mentions high-concurrency workflows and many parallel schema-mode agents

That does not expose the missing workflow task implementation, but it does verify a surrounding runtime assumption:

- workflow execution can involve many concurrent agent-scoped hook/controller registrations

So concurrency is a real workflow design pressure in the code, not speculation from the docs.

## Workflow Worktree Ownership Exists, But Only Indirectly Here

`utils/worktree.ts` includes workflow-specific stale-cleanup patterns:

- `wf_<runId>-<idx>`
- legacy `wf-<idx>`

The helper comments explicitly attribute those to `WorkflowTool`.

That verifies two things:

- workflows can create throwaway worktrees
- the core cleanup subsystem recognizes them as a dedicated producer category

What is not visible in this snapshot is the owning workflow implementation that decides when those worktrees are created or preserved.

## Workflow Permission Handling Is Also A Dedicated Tool Surface

`components/permissions/PermissionRequest.tsx` includes a gated workflow-specific branch:

- `WorkflowTool`
- `WorkflowPermissionRequest`

So workflows are not only present in task UI and transcript storage. They also have a dedicated permission-request presentation path when the feature is enabled.

That is another signal that the runtime treats workflow execution as a first-class subsystem.

## What Is Verified Vs. Not Verified

Verified from code in this snapshot:

- workflows are a dedicated task type in app-state unions
- workflows have dedicated operator-facing task UI treatment
- workflow detail/control paths are build-gated behind `WORKFLOW_SCRIPTS`
- workflow tasks emit dedicated SDK metadata via `workflow_name`
- workflow progress can be emitted as structured `workflow_progress`
- workflow-spawned agents can be transcript-grouped under `subagents/workflows/<runId>/`
- workflow commands are a dedicated prompt-command kind
- workflow runtime design assumes high-concurrency agent orchestration
- workflow-created ephemeral worktrees are recognized by cleanup helpers
- workflow permission UI has its own gated tool surface

Not verified from source internals in this snapshot:

- the internal `LocalWorkflowTaskState` field schema beyond its observed call-site usage
- how workflow scripts are parsed and executed
- how the workflow scheduler chooses agent order, retries, or dependencies
- how workflow progress batches are computed internally
- the exact semantics of `skipWorkflowAgent(...)` and `retryWorkflowAgent(...)`
- the owning `WorkflowTool` implementation

## Source-Of-Truth Summary

The code in this snapshot shows workflows as a real runtime subsystem with their own:

- task identity
- SDK event metadata
- progress schema
- transcript-grouping model
- UI/control surface
- command kind
- worktree footprint

The main correction to a shallow reading is that workflows are not "just prompt commands" and not "just subagents". The surrounding runtime clearly models them as an orchestration layer that coordinates standard agent execution while preserving workflow-specific naming, grouping, progress, permissions, and operator controls.
