# Sandbox And Filesystem Safety Boundaries

## Scope

This document covers the shell sandbox runtime, how sandbox configuration is assembled, how Bash and PowerShell opt into or refuse sandboxing, and how path-level read/write safety is enforced independently of the shell sandbox.

Primary source files:

- `entrypoints/sandboxTypes.ts`
- `utils/sandbox/sandbox-adapter.ts`
- `tools/BashTool/shouldUseSandbox.ts`
- `tools/BashTool/BashTool.tsx`
- `tools/PowerShellTool/PowerShellTool.tsx`
- `utils/permissions/pathValidation.ts`
- `utils/permissions/filesystem.ts`
- `cli/structuredIO.ts`
- `components/permissions/SandboxPermissionRequest.tsx`
- `commands/add-dir/add-dir.tsx`
- `commands/sandbox-toggle/index.ts`
- `commands/sandbox-toggle/sandbox-toggle.tsx`
- `cli/print.ts`

## The Key Architectural Split

The code does not treat "sandboxing" as one thing.

There are two different protection layers:

1. the shell sandbox constrains process-level filesystem and network access for shell execution
2. the permission and path-validation layer decides which paths the app is willing to read, write, edit, or remove in the first place

Those layers overlap, but they are not equivalent.

Examples verified from code:

- Bash can run inside the runtime sandbox via `shouldUseSandbox(...)`
- file tools still run through `checkReadPermissionForTool(...)`, `checkWritePermissionForTool(...)`, `validatePath(...)`, and `checkPathSafetyForAutoEdit(...)`
- sandbox filesystem allowlists do not replace working-directory checks or dangerous-path checks

That distinction matters because the existing docs can make sandboxing sound like the whole safety model. The code says otherwise.

## Sandbox Settings Are A First-Class Schema

`entrypoints/sandboxTypes.ts` is the source of truth for the sandbox settings shape.

Important fields include:

- `enabled`
- `failIfUnavailable`
- `autoAllowBashIfSandboxed`
- `allowUnsandboxedCommands`
- `network.allowedDomains`
- `network.allowManagedDomainsOnly`
- `network.allowUnixSockets`
- `network.allowAllUnixSockets`
- `network.allowLocalBinding`
- `network.httpProxyPort`
- `network.socksProxyPort`
- `filesystem.allowWrite`
- `filesystem.denyWrite`
- `filesystem.allowRead`
- `filesystem.denyRead`
- `filesystem.allowManagedReadPathsOnly`
- `ignoreViolations`
- `enableWeakerNestedSandbox`
- `enableWeakerNetworkIsolation`
- `excludedCommands`

One important comment in the schema is load-bearing: some sandbox network and filesystem settings are merged with permission rules such as `Edit(...)`, `Read(...)`, and `WebFetch(domain:...)`.

So the sandbox config is not only a static settings blob. The runtime adapter merges settings and permission context together.

## The Adapter Builds Runtime Sandbox Config From More Than Sandbox Settings

`utils/sandbox/sandbox-adapter.ts` is the real integration layer between CLI state and `@anthropic-ai/sandbox-runtime`.

`convertToSandboxRuntimeConfig(settings)` does not just translate `settings.sandbox`. It assembles runtime config from:

- sandbox settings
- permission rules
- policy settings
- working-directory state
- session bootstrap state
- special internal safety exclusions

### Filesystem merging

Verified behavior:

- `allowWrite` is seeded with `.` and the Claude temp directory
- `Edit(...)` allow rules are converted into sandbox `allowWrite`
- `Edit(...)` deny rules become `denyWrite`
- `Read(...)` deny rules become `denyRead`
- explicit sandbox `filesystem.allowRead`, `allowWrite`, `denyRead`, and `denyWrite` are also merged in
- if `allowManagedReadPathsOnly` is enabled, only managed policy sources contribute sandbox read allowlists
- if the session is running inside a git worktree, the main repo path is added so the worktree can still operate correctly
- extra working directories from settings and bootstrap state are added so sandboxed shell commands can access them

### Network merging

Verified behavior:

- `network.allowedDomains` contributes to the runtime allowlist
- `WebFetch(domain:...)` allow rules also contribute
- deny rules for domains are collected too
- if `allowManagedDomainsOnly` is enabled, only managed policy settings are allowed to contribute domain allow rules

So sandbox network policy is partly driven by the same permission-rule machinery used elsewhere in the app.

## Sandbox Path Semantics Are Not The Same As Permission-Rule Path Semantics

The adapter has two separate path resolvers because the code intentionally keeps these syntaxes distinct.

`resolvePathPatternForSandbox(...)` handles permission-rule patterns:

- `//path` means absolute-from-root
- `/path` means relative to the settings file directory root
- other paths are passed through

`resolveSandboxFilesystemPath(...)` handles sandbox settings paths:

- `/path` is absolute
- `~/path` expands to the user home
- relative paths are resolved relative to the settings file directory
- `//path` is preserved as a compatibility escape hatch

That means a leading slash in permission rules and a leading slash in sandbox settings do not mean the same thing.

This is easy to miss if someone assumes both systems share one path grammar.

## Startup Enables Sandboxing Conditionally, And Can Refuse To Start

The adapter does not assume sandboxing is always available.

`isSandboxingEnabled()` checks:

- whether sandboxing is enabled in settings
- whether the current platform is supported
- whether required dependencies are present
- whether `enabledPlatforms` restrictions allow the current platform

`getSandboxUnavailableReason()` returns a user-facing reason when sandboxing was explicitly requested but cannot be used.

`isSandboxRequired()` becomes true when sandboxing is enabled and `failIfUnavailable` is set.

The verified startup path in `cli/print.ts` is:

1. read sandbox availability
2. if unavailable and `isSandboxRequired()` is true, print a refusal message and exit
3. otherwise initialize `SandboxManager` and pass in the headless sandbox ask callback

So `failIfUnavailable` is not a cosmetic setting. It can block the session from starting.

## Runtime Sandbox State Is Live, Not Frozen At Boot

`SandboxManager.initialize(...)` does more than create a runtime once.

Verified behavior:

- it initializes the runtime config
- it wraps the ask callback so managed-domain-only policy can hard-block permanent network approvals
- it subscribes to settings changes
- it pushes refreshed config when relevant settings change

The adapter also exposes `refreshConfig()` so code that changes working-directory or permission state can resync the runtime immediately.

That live refresh path is important because filesystem scope can expand during a session.

## Convenience Exclusions Do Not Define The Security Boundary

`tools/BashTool/shouldUseSandbox.ts` is intentionally narrower than the full safety model.

`shouldUseSandbox(input)` returns false when:

- sandboxing is not enabled
- `dangerouslyDisableSandbox` is requested and fallback unsandboxed commands are allowed
- there is no command
- the command matches an excluded-command pattern
- dynamic config disables sandboxing for that command

The file comment is explicit that `excludedCommands` is a user-convenience mechanism, not the real security boundary.

So `/sandbox exclude ...` should be documented as an execution-routing override, not as the primary source of tool safety.

## Bash Uses The Sandbox At Execution Time

`tools/BashTool/BashTool.tsx` passes `shouldUseSandbox(input)` down into the shared `exec(...)` call.

That means sandbox routing happens at actual command execution, alongside:

- timeout handling
- background-task handling
- cwd-change prevention
- progress streaming

The Bash tool also annotates stderr with sandbox failure details through `SandboxManager.annotateStderrWithSandboxFailures(...)`.

So the sandbox integration is not only preflight policy. It is part of the shell execution path and user-visible diagnostics.

## Native Windows Is Treated As A Special Case

`tools/PowerShellTool/PowerShellTool.tsx` makes the platform split very clear.

Verified behavior:

- native Windows does not use the sandbox runtime for PowerShell execution
- `runPowerShellCommand(...)` explicitly passes `shouldUseSandbox: false` on native Windows
- if policy requires sandboxing and unsandboxed commands are not allowed, the tool returns `WINDOWS_SANDBOX_POLICY_REFUSAL`
- the refusal guard exists in `call(...)` as well because some callers bypass `validateInput(...)`

This is an important architectural nuance:

- unsupported sandbox execution on native Windows is not silently treated as acceptable when policy forbids fallback
- the tool refuses instead of pretending the command can be sandboxed

## Network Approvals Reuse The General Permission Protocol

The sandbox can ask for network access dynamically.

In headless and structured I/O contexts, `cli/structuredIO.ts` bridges those requests through `createSandboxAskCallback()`.

That callback converts a sandbox network request into a synthetic `can_use_tool` control request using `SANDBOX_NETWORK_ACCESS_TOOL_NAME`, with descriptions such as "Allow network connection to host?".

If the control path fails or cannot respond, the network request is denied.

So the sandbox does not invent a separate approval channel for headless use. It reuses the existing permission-control protocol.

## Interactive Network Approval Has A Dedicated UI

`components/permissions/SandboxPermissionRequest.tsx` is the interactive UI for sandbox network access.

It offers:

- a one-time allow
- a persistent allow for the host
- a deny

But there is a policy-sensitive constraint:

- if `shouldAllowManagedSandboxDomainsOnly()` is true, the persistent allow option is hidden

So managed policy can prevent the user from permanently extending sandbox network access even when temporary approval is still possible.

## Filesystem Safety Is Enforced Separately From Shell Sandboxing

The app-level path system lives mainly in `utils/permissions/pathValidation.ts` and `utils/permissions/filesystem.ts`.

This layer applies to file-oriented tools whether or not a shell sandbox is involved.

Important functions include:

- `validatePath(...)`
- `validateGlobPattern(...)`
- `isPathAllowed(...)`
- `checkReadPermissionForTool(...)`
- `checkWritePermissionForTool(...)`
- `checkPathSafetyForAutoEdit(...)`
- `isDangerousRemovalPath(...)`

This is the subsystem that determines whether the app should even attempt a read or write.

## Working Directories Are A Core App-Level Boundary

The code has an explicit working-directory model that is separate from sandbox config.

`allWorkingDirectories(context)` combines:

- the original cwd
- any additional working directories in `ToolPermissionContext`

`pathInWorkingPath(...)` and `pathInAllowedWorkingPath(...)` resolve symlinks and require every resolved path variant to stay within an allowed workspace root.

That matters because many default read/write permissions are framed in terms of working directories, not sandbox allowlists.

Examples verified in the permission code:

- reads inside the working directory are normally allowed
- writes can be auto-allowed in working directories when mode and tool policy permit it
- out-of-tree access often becomes an `ask` with suggestions such as `addDirectories`

## Path Safety Checks Block Dangerous Targets Before Normal Allow Rules

`checkPathSafetyForAutoEdit(...)` is a load-bearing guard.

It checks both original and resolved paths and blocks dangerous targets such as:

- Claude-managed config locations
- `.git`
- `.vscode`
- `.idea`
- shell rc/profile files
- suspicious Windows path forms

It returns structured safety data including whether the action is approvable.

This is important because safety checks happen before many normal allow paths. A writable location is not automatically a safe location.

## Path Validation Uses An Ordered Decision Pipeline

`isPathAllowed(...)` in `pathValidation.ts` makes the ordering explicit.

The verified high-level sequence is:

1. deny rules
2. editable internal-path checks
3. safety checks
4. working-directory checks
5. readable internal-path checks
6. sandbox write allowlist fallback for writes outside the working directory
7. explicit allow rules

That ordering tells us two important things:

- sandbox write allowlists are additive fallback scope, not the first or only gate
- the sandbox-seeded `.` write scope does not bypass ordinary working-directory and safety logic

So the shell sandbox and app permission model are deliberately layered rather than flattened together.

## Internal Claude Paths Are Treated Specially

The permission layer distinguishes ordinary repo files from internal session/config files.

Verified behavior includes:

- some internal Claude-managed paths are readable or editable through dedicated internal-path checks
- settings files and managed drop-in directories are denied in sandbox write config
- `.claude/skills` under original or current cwd is denied in sandbox write config
- write handling includes a special session-scoped `.claude/**` allow-rule bypass before normal safety checks

This is another place where the code does not behave like a simple "inside repo good, outside repo bad" model.

## Bare Git Repositories Get Extra Protection

The sandbox adapter contains a specific mitigation for bare repositories.

Verified behavior:

- existing bare-repo files in cwd are added to sandbox `denyWrite`
- paths that do not exist yet but would be dangerous in a bare repo are tracked in `bareGitRepoScrubPaths`
- `cleanupAfterCommand()` scrubs those tracked paths after command execution

So the sandbox layer includes repo-shape-specific cleanup logic, not only static allow/deny lists.

## Adding A Workspace Directory Updates Both Permissions And Sandbox

`commands/add-dir/add-dir.tsx` is a good example of the split-and-link model.

When the user adds a directory, the command:

- applies an `addDirectories` permission update to the tool-permission context
- stores session-scoped directory state in bootstrap/session state
- calls `SandboxManager.refreshConfig()` so sandboxed shell commands can access the new directory immediately

This shows that working-directory permissions and sandbox config are different stores that must be kept in sync.

## `/sandbox` Is A Real Runtime Management Surface

`commands/sandbox-toggle/` is not a simple boolean toggle.

The command surfaces:

- whether sandboxing is enabled
- whether auto-allow is enabled
- whether unsandboxed fallback is allowed
- whether settings are locked by managed policy

It also validates:

- platform support
- dependency availability
- enabled-platform restrictions
- policy lock state

And it supports an `exclude` workflow that writes convenience exclusions into local sandbox settings.

So the sandbox subsystem includes an explicit interactive management path, not just hidden config.

## Corrections To The Existing Mental Model

- The shell sandbox is not the whole safety system; path validation and permission checks remain separate app-level gates.
- Sandbox runtime config is built from both sandbox settings and permission rules, especially for `Edit(...)`, `Read(...)`, and `WebFetch(domain:...)`.
- Working-directory access is a primary permission concept and is only partially mirrored into sandbox config for shell execution.
- Sandbox command exclusions are convenience routing controls, not the authoritative security boundary.
- Native Windows does not silently "fall back" when sandbox policy requires isolation; PowerShell explicitly refuses execution in that case.
- Network sandbox asks flow through the same approval/control infrastructure as other permission asks, with separate UI only at the presentation layer.
