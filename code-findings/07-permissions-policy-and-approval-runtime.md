# Permissions, Policy, And Approval Runtime

## Scope

This document covers how permission rules are assembled, how per-tool decisions are made, how approval is collected at runtime, and where organization policy fits into the picture.

Primary source files:

- `types/permissions.ts`
- `utils/permissions/PermissionMode.ts`
- `utils/permissions/permissionSetup.ts`
- `utils/permissions/permissions.ts`
- `utils/permissions/permissionsLoader.ts`
- `utils/permissions/PermissionUpdate.ts`
- `utils/permissions/denialTracking.ts`
- `utils/permissions/bypassPermissionsKillswitch.ts`
- `hooks/useCanUseTool.tsx`
- `hooks/toolPermission/PermissionContext.ts`
- `hooks/toolPermission/handlers/interactiveHandler.ts`
- `hooks/toolPermission/handlers/coordinatorHandler.ts`
- `hooks/toolPermission/handlers/swarmWorkerHandler.ts`
- `components/permissions/PermissionRequest.tsx`
- `components/permissions/PermissionPrompt.tsx`
- `components/permissions/rules/PermissionRuleList.tsx`
- `services/policyLimits/index.ts`
- `main.tsx`
- `cli/print.ts`
- `setup.ts`

## The Key Architectural Split

The code does not implement "permissions" as one subsystem.

There are three distinct layers:

1. `permissionSetup.ts` and related loaders build a `ToolPermissionContext`
2. `permissions.ts` decides `allow`, `ask`, or `deny` for a concrete tool use
3. `useCanUseTool.tsx` and the tool-permission handlers turn `ask` into an actual runtime approval flow

Organization policy is adjacent, not embedded into the per-tool checker:

- `services/policyLimits/index.ts` gates product capabilities such as remote control, remote sessions, and feedback
- those checks are feature gates, not the same thing as `hasPermissionsToUseTool(...)`

That split matters because "policy blocked" and "tool permission denied" come from different code paths.

## Core Types And Mental Model

`types/permissions.ts` defines the stable shapes used across the subsystem.

Important concepts:

- `PermissionMode`
- `PermissionRule`
- `PermissionUpdate`
- `PermissionDecision`
- `PermissionDecisionReason`
- `ToolPermissionContext`

The `ToolPermissionContext` is the central state object passed into permission checks. It contains:

- active mode
- additional working directories
- allow, deny, and ask rules grouped by source
- flags such as `isBypassPermissionsModeAvailable`
- runtime flags such as `shouldAvoidPermissionPrompts`
- plan/auto bookkeeping such as `prePlanMode` and `strippedDangerousRules`

This is the real source of truth for tool-approval behavior during a session.

## Permission Modes Are More Than Ask vs Bypass

The runtime mode set is broader than the old docs imply.

External modes:

- `default`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `plan`

Internal/runtime-only extensions:

- `auto`
- `bubble` type support exists in the union, but it is not part of the normal user-addressable mode list in this build path

Important verified behavior from `PermissionMode.ts` and `permissionSetup.ts`:

- CLI and settings resolve to a `PermissionMode`
- `dontAsk` converts runtime `ask` results into `deny`
- `bypassPermissions` is not equivalent to "ignore everything"; some asks still survive
- `auto` is classifier-backed and has its own gate checks and rule stripping
- `plan` can preserve or temporarily adapt the previous mode via `prePlanMode`

## Rule Sources And Precedence Shape The Context

The checker does not read one permission file.

Rules come from multiple sources:

- `userSettings`
- `projectSettings`
- `localSettings`
- `policySettings`
- `flagSettings`
- `cliArg`
- `command`
- `session`

`permissionsLoader.ts` loads disk-backed rules. `permissionSetup.ts` then layers CLI-derived rules and runtime state into the final context.

Important verified behavior:

- `loadAllPermissionRulesFromDisk()` uses all enabled setting sources by default
- if `allowManagedPermissionRulesOnly` is enabled in `policySettings`, only managed policy rules are loaded
- CLI `--allowed-tools` become allow rules under `cliArg`
- CLI `--disallowed-tools` become deny rules under `cliArg`
- `--tools` is handled differently: it is turned into broad deny rules for tools not in the selected base set
- extra workspace directories are tracked separately from allow/deny/ask rules

So the effective permission context is assembled, not read verbatim from a single file.

## Startup Builds The Permission Context Early

`main.tsx` resolves the initial permission mode first, then calls `initializeToolPermissionContext(...)`.

That setup function does more than the name suggests:

- parses CLI allow/deny/base-tool lists
- loads permission rules from disk
- computes whether bypass mode is available this session
- detects dangerous auto-mode-bypassing allow rules
- detects overly broad shell allow rules
- validates and adds extra workspace directories
- returns warnings plus the built `toolPermissionContext`

`main.tsx` then places that context into initial app state for interactive mode and headless mode.

## Bypass Permissions Is Deliberately Constrained

The code treats bypass mode as dangerous enough to require several separate gates.

### Startup environment restrictions

`setup.ts` enforces hard restrictions when bypass mode is active or available:

- on Unix, root/sudo is rejected unless already inside a sandbox
- for ant builds outside trusted launcher contexts, bypass mode is only allowed in Docker/bubblewrap/sandbox environments without internet access

So `--dangerously-skip-permissions` is not only a UI toggle. Startup can reject the process entirely.

### Availability restrictions

`initialPermissionModeFromCLI(...)` and related helpers can disable bypass mode when:

- a cached organization gate disables it
- settings disable it

Later, `bypassPermissionsKillswitch.ts` can asynchronously revoke it after startup.

### Runtime behavior

Even in bypass mode, some prompts remain mandatory.

From `permissions.ts`, bypass does not override:

- deny rules
- ask rules that explicitly target a tool or subcommand
- tool-level `requiresUserInteraction()` asks
- safety-check asks

So bypass mode is "skip ordinary prompts when allowed", not "skip every guard".

## Auto Mode Is A Separate Safety Model

`auto` is not just another alias for bypass mode.

The code treats it as classifier-backed automation with its own safety preservation rules.

### Dangerous rules are stripped on entry

`stripDangerousPermissionsForAutoMode(...)` removes allow rules that would bypass the classifier before it can evaluate an action.

Examples of dangerous rules the code explicitly cares about:

- `Bash(*)`
- interpreter-style Bash rules such as `Bash(python:*)`
- dangerous PowerShell launcher/eval rules
- Agent-task allow rules

The stripped rules are stored in `strippedDangerousRules` and restored when leaving auto mode.

### Auto mode availability is dynamic

`verifyAutoModeGateAccess(...)` checks whether auto mode is currently allowed based on:

- dynamic config / circuit breaker state
- settings
- model support
- fast-mode breaker logic

This means the session can be kicked out of auto mode after startup if the live gate says it is unavailable.

### Plan mode can carry auto semantics

`prepareContextForPlanMode(...)` and `transitionPlanAutoMode(...)` show that plan mode and auto mode are linked:

- entering plan mode from auto may preserve auto intent
- opting into "auto during plan" can activate classifier semantics inside plan mode
- dangerous rules may be stripped while still in plan mode

So plan mode is not fully independent from auto mode in the current implementation.

## The Actual Permission Decision Pipeline Lives In `permissions.ts`

`hasPermissionsToUseTool(...)` is the main decision function.

The verified high-level order is:

1. check whole-tool deny rules
2. check whole-tool ask rules
3. run the tool's own `checkPermissions(...)`
4. honor tool-level denies
5. honor user-interaction-required asks
6. honor content-specific ask rules
7. honor safety-check asks
8. apply mode-level bypass rules
9. check whole-tool allow rules
10. convert remaining passthrough results into `ask`
11. apply mode-specific transformations such as `dontAsk` or `auto`
12. for headless/non-prompting contexts, run hooks or auto-deny

That ordering is important:

- whole-tool and content-specific rule checks happen before bypass-mode allow
- allow rules are not the first thing checked
- tool-specific `checkPermissions(...)` is part of the central pipeline, not an afterthought

## Decisions Carry Structured Reasons

The permission layer does not only return `allow` / `ask` / `deny`.

It also returns a `PermissionDecisionReason`, which can identify:

- a matching rule
- the active mode
- a hook result
- classifier output
- a sandbox override
- a working-directory problem
- a safety check
- an async/headless limitation
- a permission-prompt-tool result

This is why the UI can explain why a prompt happened and why the SDK can surface structured denial data.

## Headless And Interactive Paths Diverge After The Decision

`permissions.ts` only decides the result. The runtime response to `ask` is handled elsewhere.

### Interactive path

In the REPL, `useCanUseTool.tsx` is the entry point.

It:

- calls `hasPermissionsToUseTool(...)`
- resolves immediate allows and denies
- loads the tool description for prompt UI
- routes `ask` decisions through specialized handlers

The handlers split further:

- `coordinatorHandler.ts` waits for hooks/classifier before showing a dialog
- `swarmWorkerHandler.ts` forwards approval to the leader mailbox
- `interactiveHandler.ts` pushes a prompt into the local confirmation queue and races local approval with hooks, classifier, bridge, and channels

### Headless path

In non-interactive contexts, `shouldAvoidPermissionPrompts` changes behavior:

- hooks get a chance to allow or deny first
- if nothing resolves the ask, the action is denied automatically

This is why the same permission decision can produce a dialog in REPL mode but an automatic denial in a headless or background-agent path.

## Auto Mode Classifier Logic Sits On Top Of The Ask Result

The auto-mode classifier is not the first permission step.

It only runs after the regular rule/tool checks produce an `ask` and the current mode says classifier handling should apply.

Verified fast paths before full classifier inference:

- if `acceptEdits` mode would allow the same action, auto mode allows it without classifier inference
- allowlisted tools can skip classifier inference
- PowerShell may still require explicit approval depending on feature flags

If the classifier blocks repeatedly, `denialTracking.ts` is used to fall back from pure classifier blocking back to user prompting:

- 3 consecutive denials
- or 20 total denials

In headless contexts, hitting that fallback threshold aborts instead of opening a prompt.

## Approval UI Is Typed By Tool

Permission UI is not one generic modal with a text label.

`components/permissions/PermissionRequest.tsx` maps tools to specialized request components, including:

- file edit/write
- Bash
- PowerShell
- notebook edit
- filesystem reads/searches
- web fetch
- skill execution
- plan-mode enter/exit
- ask-user-question

Unknown or unsupported tools fall back to a generic renderer.

This means the approval system is aware of tool shape and can collect richer input than a simple yes/no.

## `PermissionPrompt.tsx` Is The Shared Approval Input Primitive

Specialized permission request components often build on `PermissionPrompt.tsx`.

That shared component owns:

- option rendering
- keybindings
- optional accept/reject feedback text
- tab-to-amend behavior
- escape handling
- analytics for feedback entry and submission

So the permission UI system is compositional:

- `PermissionRequest.tsx` chooses the tool-specific wrapper
- specialized request components shape the content
- `PermissionPrompt.tsx` provides the common selection/feedback mechanics

## Runtime Approval Is A Multi-Party Race, Not Just A Local Dialog

`interactiveHandler.ts` makes this explicit.

Once a local ask is enqueued, multiple actors can resolve it:

- the local user in the CLI
- permission hooks
- async classifier approval
- bridge responses from remote control / CCR
- channel replies from allowed messaging channels

The handler uses resolve-once guards so only the first winner applies.

That is a major architectural fact: the permission prompt queue is a coordination point for several approval transports, not just a UI list.

## Permission Updates Are First-Class Data

Approving a request can produce `PermissionUpdate[]`, not just a yes/no.

`PermissionUpdate.ts` handles:

- applying updates in memory
- persisting updates when the destination is editable
- adding/removing workspace directories
- changing default mode

Editable destinations are intentionally limited:

- `userSettings`
- `projectSettings`
- `localSettings`

Session and CLI destinations are in-memory only.

This explains why some approvals can become permanent and others cannot.

## Managed Policy Can Restrict Which Rules Matter

`permissionsLoader.ts` supports a managed-policy mode:

- when `allowManagedPermissionRulesOnly` is true, only `policySettings` rules are loaded for execution
- "always allow" options are hidden from approval UI in that mode
- syncing rules from disk can clear non-policy sources from the active context

So enterprise policy can constrain not just feature access, but which permission rules are respected at all.

## `/permissions` Is A Real Management Surface

The `/permissions` command is not a thin debug view.

`commands/permissions/permissions.tsx` launches `PermissionRuleList`, which provides an interactive management UI over:

- current allow/ask/deny rules
- recent auto-mode denials
- workspace directory access
- rule deletion
- rule creation

The code also respects rule mutability:

- managed `policySettings` rules can be viewed but not deleted
- editable sources can be updated and persisted

So the permission subsystem includes a user-facing management interface, not only background checks.

## QueryEngine Tracks Denials For SDK Consumers

`QueryEngine.ts` wraps `canUseTool` so non-allow outcomes are recorded into `permissionDenials`.

That data is then included in SDK-oriented outputs.

This is separate from the decision logic itself, but important for the system model:

- QueryEngine is not deciding permissions
- it is recording permission outcomes for downstream consumers

## Print Mode Has Its Own Approval Path

`cli/print.ts` does not use the interactive REPL prompt queue.

Instead, headless mode can use:

- stdio-driven permission handling
- or an MCP `--permission-prompt-tool`

`createCanUseToolWithPermissionPrompt(...)` wraps the normal `hasPermissionsToUseTool(...)` decision function and, for `ask` outcomes, delegates approval to the configured prompt tool.

That wrapper:

- preserves ordinary allow/deny results
- races the prompt tool against abort signals
- converts the MCP tool output back into a normal permission decision

So headless permission handling is pluggable, but still layered on top of the same core checker.

## Organization Policy Is Separate From Tool Permissions

`services/policyLimits/index.ts` implements organization-level policy restrictions.

Important verified behavior:

- eligibility depends on provider/auth context
- restrictions are fetched from an API with retry and cache support
- restrictions are cached in memory and on disk
- policy checks generally fail open if policy data is unavailable
- one essential-traffic exception (`allow_product_feedback`) can fail closed on cache miss

The policy service is then used by feature entrypoints such as:

- remote control
- remote sessions
- feedback-related features

Examples from the code:

- `entrypoints/cli.tsx` waits for policy limits before bridge remote-control startup
- `bridge/initReplBridge.ts` blocks bridge init if `allow_remote_control` is not allowed
- `main.tsx` and `cli/print.ts` check `allow_remote_sessions` before remote-session features

This is not the same as a tool allow/deny rule. It is a product capability gate.

## Corrections To The Existing Mental Model

- The permission system is not just a prompt around tool execution; it starts with context assembly and mode resolution.
- `bypassPermissions` is not absolute. Rule-based asks, safety checks, and user-interaction-required tools can still stop it.
- `auto` mode is classifier-backed and actively strips dangerous allow rules instead of trusting all preexisting allow config.
- Interactive permission approval is not local-only; bridge, channels, hooks, and classifier paths can all resolve the same ask.
- Headless mode does not simply "skip prompts"; it can auto-deny, run hooks, or delegate approval to a permission prompt tool.
- Organization policy limits are a separate feature-gating system, not part of the per-tool permission matcher.
