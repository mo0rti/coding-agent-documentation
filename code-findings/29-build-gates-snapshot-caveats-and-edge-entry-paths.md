# Build Gates, Snapshot Caveats, and Edge Entry Paths

## Scope

This document covers two related meta-level concerns visible in the codebase:

- how build gates and dynamic imports shape what is present in this workspace snapshot
- which entry paths intentionally bypass the normal `setup()` to `main.tsx` interactive startup flow

Primary files read for this slice:

- `entrypoints/cli.tsx`
- `main.tsx`
- `bridge/bridgeMain.ts`
- `utils/sinks.ts`
- `utils/claudeInChrome/mcpServer.ts`
- `utils/computerUse/mcpServer.ts`
- `services/mcp/client.ts`
- `utils/toolPool.ts`
- `components/teams/TeamsDialog.tsx`

## Why This Slice Exists

By this point in the findings set, several docs have had the same caveat:

- a referenced module exists at the call site
- but the owning source file is not present in this workspace snapshot

The code itself explains much of that.

This repo uses:

- `feature(...)` gates
- dynamic `require(...)`
- dynamic `import(...)`
- build-specific stubs
- fast-path entrypoints that avoid the normal startup stack

So “missing from the snapshot” does not always mean “documentation missed something” or “the code is broken.” Often it means the runtime was deliberately structured for dead-code elimination, build-specific packaging, or alternate boot flows.

## The Codebase Is Deliberately Shaped For DCE

`entrypoints/cli.tsx`, `main.tsx`, `utils/toolPool.ts`, `components/tasks/BackgroundTasksDialog.tsx`, and many other files contain explicit comments about Bun dead-code elimination.

Verified recurring pattern:

- feature-gated modules are often loaded with inline `feature(...)` checks
- many gated modules are pulled in via `require(...)` instead of static imports
- comments repeatedly note that these forms are load-bearing for DCE

This means the source layout is optimized not just for runtime behavior, but for which strings, imports, and modules survive into different builds.

## A Missing Module In This Snapshot Can Be A Build Shape, Not A Logic Gap

The workspace confirms several owner modules are currently absent:

- `ssh/createSSHSession.js`
- `utils/ccshareResume.js`
- `services/compact/reactiveCompact.js`
- `services/compact/cachedMicrocompact.js`
- `services/compact/snipCompact.js`
- `services/compact/contextCollapse.js`
- `tasks/LocalWorkflowTask/LocalWorkflowTask.js`
- `tools/WorkflowTool/WorkflowTool.js`
- `cli/handlers/templateJobs.js`
- `jobs/classifier.js`

These absences line up with patterns already visible in the code:

- feature-gated workflow/task surfaces
- template/job fast paths with missing owner handlers
- compaction internals referenced through contracts
- ant-only or alternate-build runtime branches

So the source-of-truth docs should treat these as verified snapshot boundaries, not silently fill them in with guesses.

## `entrypoints/cli.tsx` Has Many Fast Paths Before The Full CLI Loads

The CLI bootstrap file is intentionally not just a thin wrapper around `main.tsx`.

Verified fast paths include:

- `--version`
- `--dump-system-prompt`
- `--claude-in-chrome-mcp`
- `--chrome-native-host`
- `--computer-use-mcp`
- `--daemon-worker`
- `remote-control` / `rc` / legacy bridge aliases
- `daemon`
- background session management commands
- template job commands
- `environment-runner`
- `self-hosted-runner`
- tmux/worktree pre-exec flow

This means “the app starts in `main.tsx`” is only true for the normal interactive path. A meaningful portion of the runtime surface is dispatched earlier.

## Not All Fast Paths Initialize The Same Subsystems

The fast paths in `entrypoints/cli.tsx` do not share one common bootstrap contract.

Verified differences:

- some paths call `enableConfigs()`
- some call `initSinks()`
- some call neither
- some exit after a narrow operation without loading the full CLI

Examples:

- `daemon` enables configs and initializes sinks
- bridge fast path enables configs and then jumps into `bridgeMain(...)`
- `--daemon-worker` stays intentionally lean and does not initialize general sinks/configs there
- Chrome/computer-use MCP subprocess paths do not go through `initSinks()`

So startup expectations must be read per-entrypath, not assumed globally.

## Bridge Is A Real Alternate Startup Funnel

`bridge/bridgeMain.ts` explicitly notes that it bypasses the usual `setup()` flow.

Verified behavior:

- it enables configs itself
- it calls `initSinks()` directly
- it performs async gate checks after that

That matches the earlier bridge docs, but this slice makes the architectural point explicit:

- bridge mode is not just a flag inside the main REPL path
- it owns a parallel startup path with its own bootstrap sequence

## Chrome And Computer-Use MCP Servers Use Narrower Bootstraps

`utils/claudeInChrome/mcpServer.ts` and `utils/computerUse/mcpServer.ts` are good examples of specialized subprocess entrypoints.

Verified behavior:

- both call `enableConfigs()`
- both initialize only the analytics sink, not the full sink pair
- both flush analytics explicitly on stdin close before exiting
- both run stdio MCP transports rather than the REPL

So these entrypaths are not miniature REPL launches. They are focused stdio services with intentionally narrow initialization.

## In-Process MCP Servers Further Complicate Surface Verification

`services/mcp/client.ts` shows that some MCP servers do not even require subprocess execution.

Verified behavior:

- Chrome MCP can run in-process
- Computer Use MCP can run in-process
- comments explicitly say the package `CallTool` handler is a stub and real dispatch happens elsewhere

That matters for documentation:

- a subprocess entrypoint may exist
- but the interactive app may still use an in-process version instead

So “where the server runs” is itself an entrypath distinction, not a fixed property of the feature.

## `main.tsx` Also Rewrites CLI Intent Before The Main Session Starts

Even on the normal path, `main.tsx` does more than handle already-parsed options.

Verified patterns:

- it strips and stashes `assistant` subcommand intent into `_pendingAssistantChat`
- it strips and stashes SSH intent into `_pendingSSH`
- it strips and stashes direct-connect intent into `_pendingConnect`

The comments explicitly say these are handled this way so the main command path can still launch the full interactive TUI.

So some commands that look like separate subcommands are actually rewritten into pending state and then resumed later inside the main control flow.

## Some Stub Behavior Is Intentional, Not Accidental

`main.tsx` explicitly documents stub fallthrough behavior.

Verified examples:

- `claude assistant --help` is allowed to fall through to the stub
- root-flag-before-subcommand cases can fall through to the stub
- build-time stubbing is explicitly called out for session-data uploading in external builds

This is another reason snapshot reading can be misleading if we assume every referenced branch must have a visible full implementation nearby.

## The Main Interactive Path Still Has Multiple Edge Branches

Inside `main.tsx`, the normal interactive launch still splits into several major alternate branches before the default REPL path:

- continue
- direct connect
- SSH remote
- assistant viewer mode
- resume / from-PR / teleport / remote session creation
- ccshare or transcript-file resume paths in ant-only branches

That means “default REPL startup” is itself only one branch inside a larger session-dispatch tree.

## Remote, Teleport, And Assistant Viewer Are Adjacent But Distinct

This slice is not re-documenting the full remote subsystem, but the entry orchestration makes one boundary especially clear:

- `--remote` can create CCR sessions and optionally launch a remote TUI
- `--teleport` resumes or attaches through a different path
- `claude assistant` is a pure viewer/client path
- SSH remote is another distinct launch mode

These are related at the product level, but they are not one unified entry implementation in code.

## The Snapshot Also Contains Fully Compiled Files Where Source Was Expected

Not every caveat is a missing module.

`components/teams/TeamsDialog.tsx` exists in this snapshot, but it is already in compiled React output form rather than hand-authored TSX source style.

That creates a different kind of documentation limit:

- the behavior is still readable and verifiable
- but some implementation intent is less obvious than in the original authored source

So snapshot limitations come in at least two forms:

- owner module absent
- owner module present only in build-emitted/compiled form

## What This Means For The Findings Set

The findings docs should continue following these rules:

- document call-site contracts when owner modules are absent
- call out build gates and snapshot limits explicitly
- avoid “rounding off” separate entrypaths into one generic startup story
- keep alternate boot paths distinct when they initialize different subsystems

This is especially important in this repo because startup behavior is intentionally fragmented for performance, packaging, and feature-gating reasons.

## Source-Of-Truth Summary

The code in this workspace shows that the project’s shape is heavily influenced by:

- dead-code-elimination-aware import structure
- feature-gated alternate modules
- specialized fast-path entrypoints
- multiple startup funnels that do not all share the same bootstrap steps

The main correction to a naive reading is that a large codebase snapshot like this should not be treated as one continuous, fully expanded source tree. Some subsystems are intentionally absent here, some are build-gated, some are stubbed or dynamically injected, and several important product surfaces boot through alternate entrypaths instead of the default interactive startup.
