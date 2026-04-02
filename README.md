# AI Agent Engineering Findings

Read the article on Medium: [Inside Claude Code: What the Source Really Tells Us About Building Production AI Agents](https://medium.com/@mortitech/inside-claude-code-what-the-source-really-tells-us-about-building-production-ai-agents-cd9c3eb7c91b)

## What This Repository Is For

This repository contains engineering-oriented documentation about production-grade AI coding agent architecture.

It is not product marketing copy, a quick-start guide, or a user manual. It is a structured set of technical notes focused on how advanced coding agents can be designed and operated in practice.

The main goal is to make complex agent-runtime concepts easier to study and discuss, including:

- startup flow
- session lifecycle
- tool execution
- permissions and approvals
- sandbox behavior
- MCP loading and runtime integration
- prompt construction
- transcript persistence and resume behavior
- headless and SDK control protocols
- subagents, swarm/team coordination, and worktrees
- diagnostics, telemetry, and build or snapshot caveats

In short: this documentation exists to explain how production-level AI agent systems can be structured under real runtime constraints.

## What Makes These Documents Different

These files are written as engineering analysis rather than high-level product summaries.

That means the set is trying to answer questions like:

- Which file actually owns this behavior?
- What is the real control-flow boundary?
- Which subsystem is the primary authority?
- What is persisted, reconstructed, streamed, or repaired at runtime?
- Which common architectural shortcuts are inaccurate?

A repeated pattern across the findings is correction of simplified mental models. In systems like these, the runtime often turns out to be more layered than a short architecture summary suggests:

- headless mode is not just a direct call into the core query loop
- permissions are not just a yes/no prompt wrapper
- resume is not just "load the last chat"
- worktrees are not just `git worktree add`
- prompt assembly is not just one static system prompt string

So this repository is best understood as a systems-design reference and engineering analysis set.

## What This Documentation Is Not

This repository is not:

- official documentation for any vendor product
- a promise that every referenced feature is present in every build
- a polished end-user guide for everyday product usage
- a legal or compliance opinion about any implementation

Several documents explicitly call out snapshot boundaries, feature gates, dynamic imports, missing owner modules, and build-specific stubs. Where something is uncertain or incomplete, the notes are intended to say so rather than smooth over the gap.

## Intended Audience

This set is useful for people who want to understand production AI agent systems at the engineering level:

- researchers mapping advanced coding-agent architecture
- people comparing runtime behavior across coding agents
- anyone trying to separate real implementation details from high-level summaries

If you want to know "where does this behavior really live?" or "what happens before and after this subsystem?", these docs are the right layer.

## How To Read It

The findings are organized as small focused slices instead of one giant architecture document.

That structure is intentional. Large coding-agent systems usually have very large orchestration files and multiple overlapping runtime modes. Breaking the documentation into narrow slices makes it easier to keep each topic understandable and technically grounded.

Recommended reading order:

1. Start with [code-findings/01-entrypoints-startup-and-modes.md](code-findings/01-entrypoints-startup-and-modes.md) and [code-findings/02-query-tools-and-state-core.md](code-findings/02-query-tools-and-state-core.md) to get the basic runtime shape.
2. Continue through the numbered files in order. The numbering reflects a deliberate progression from startup and core loop into permissions, prompt shaping, persistence, SDK/headless control, orchestration, and finally observability and snapshot caveats.

## Documentation Method

The working method behind these files is visible in the writing style:

- document only what can be supported by the available technical evidence
- keep ownership boundaries explicit
- separate process setup, session setup, and turn execution
- distinguish interactive, headless, bridge, and remote paths
- call out when a workspace snapshot has missing or compiled-only source
- prefer correction of inaccurate shorthand over elegant but fuzzy summaries

That makes these documents especially useful as companion material while studying complex AI agent runtime design.

## Coverage Map

The current findings set covers these areas:

- [code-findings/01-entrypoints-startup-and-modes.md](code-findings/01-entrypoints-startup-and-modes.md): startup flow, mode dispatch, interactive vs headless launch
- [code-findings/02-query-tools-and-state-core.md](code-findings/02-query-tools-and-state-core.md): core query loop, tool layer, and state boundaries
- [code-findings/03-commands-skills-and-plugin-loading.md](code-findings/03-commands-skills-and-plugin-loading.md): command registry, skills, plugins, and model-invocable prompt commands
- [code-findings/04-mcp-configuration-and-runtime.md](code-findings/04-mcp-configuration-and-runtime.md): MCP config loading and runtime integration
- [code-findings/05-bridge-remote-control-and-remote-sessions.md](code-findings/05-bridge-remote-control-and-remote-sessions.md): bridge mode, remote control, and remote session entry paths
- [code-findings/06-repl-composition-and-ui-runtime.md](code-findings/06-repl-composition-and-ui-runtime.md): REPL composition and interactive runtime structure
- [code-findings/07-permissions-policy-and-approval-runtime.md](code-findings/07-permissions-policy-and-approval-runtime.md): permission context, approval flow, classifier/auto mode, and policy gates
- [code-findings/08-sandbox-and-filesystem-safety-boundaries.md](code-findings/08-sandbox-and-filesystem-safety-boundaries.md): sandboxing and filesystem safety rules
- [code-findings/09-hooks-and-session-lifecycle.md](code-findings/09-hooks-and-session-lifecycle.md): hook system and session lifecycle integration
- [code-findings/10-settings-config-sources-and-change-propagation.md](code-findings/10-settings-config-sources-and-change-propagation.md): config sources, precedence, and propagation
- [code-findings/11-authentication-provider-resolution-and-credential-helpers.md](code-findings/11-authentication-provider-resolution-and-credential-helpers.md): auth/provider resolution and credential helper behavior
- [code-findings/12-api-client-retries-and-transport-runtime.md](code-findings/12-api-client-retries-and-transport-runtime.md): API client behavior, retries, and transport layer details
- [code-findings/13-prompt-assembly-system-prompts-and-api-visible-conversation-shaping.md](code-findings/13-prompt-assembly-system-prompts-and-api-visible-conversation-shaping.md): system prompt construction, attachment rendering, normalization, and API-facing prompt shaping
- [code-findings/14-compaction-summarization-and-context-budget-management.md](code-findings/14-compaction-summarization-and-context-budget-management.md): compaction and context-budget behavior
- [code-findings/15-history-snip-and-context-collapse-runtime.md](code-findings/15-history-snip-and-context-collapse-runtime.md): history snip and context-collapse mechanisms
- [code-findings/16-transcript-persistence-resume-reconstruction-and-session-log-topology.md](code-findings/16-transcript-persistence-resume-reconstruction-and-session-log-topology.md): transcript persistence, replay, and resume reconstruction
- [code-findings/17-session-discovery-progressive-log-enrichment-and-resume-picker-behavior.md](code-findings/17-session-discovery-progressive-log-enrichment-and-resume-picker-behavior.md): session discovery and resume picker behavior
- [code-findings/18-cross-project-resume-session-switching-and-alternate-resume-entry-paths.md](code-findings/18-cross-project-resume-session-switching-and-alternate-resume-entry-paths.md): cross-project resume and alternate resume paths
- [code-findings/19-print-mode-headless-session-lifecycle-and-replay-semantics.md](code-findings/19-print-mode-headless-session-lifecycle-and-replay-semantics.md): print mode and headless session lifecycle
- [code-findings/20-structured-io-control-protocol-sdk-events-and-headless-control-flow.md](code-findings/20-structured-io-control-protocol-sdk-events-and-headless-control-flow.md): structured I/O protocol, SDK control flow, and headless session transport
- [code-findings/21-background-tasks-subagents-and-notification-flow.md](code-findings/21-background-tasks-subagents-and-notification-flow.md): background tasks, subagents, and notifications
- [code-findings/22-swarm-team-coordination-and-leader-teammate-protocols.md](code-findings/22-swarm-team-coordination-and-leader-teammate-protocols.md): swarm/team coordination and leader-worker protocols
- [code-findings/23-team-discovery-pane-worktree-backends-and-operator-ui.md](code-findings/23-team-discovery-pane-worktree-backends-and-operator-ui.md): team discovery UI and worktree-backed team flows
- [code-findings/24-worktree-creation-isolation-and-cleanup-semantics.md](code-findings/24-worktree-creation-isolation-and-cleanup-semantics.md): worktree creation, isolation, persistence, and cleanup
- [code-findings/25-agent-workflow-and-bridge-worktree-consumers.md](code-findings/25-agent-workflow-and-bridge-worktree-consumers.md): worktree consumers in agent, workflow, and bridge contexts
- [code-findings/26-templates-jobs-subsystem-and-job-scoped-runtime-boundaries.md](code-findings/26-templates-jobs-subsystem-and-job-scoped-runtime-boundaries.md): templates, jobs, and job-scoped runtime behavior
- [code-findings/27-workflow-runtime-transcript-grouping-and-orchestration-boundaries.md](code-findings/27-workflow-runtime-transcript-grouping-and-orchestration-boundaries.md): workflow orchestration and transcript grouping
- [code-findings/28-diagnostics-telemetry-and-runtime-observability.md](code-findings/28-diagnostics-telemetry-and-runtime-observability.md): diagnostics, analytics, telemetry, tracing, and observability
- [code-findings/29-build-gates-snapshot-caveats-and-edge-entry-paths.md](code-findings/29-build-gates-snapshot-caveats-and-edge-entry-paths.md): build gates, snapshot limitations, and alternate entry paths

## The Most Important Takeaway

This repository is best read as an engineering-oriented analysis of production-grade AI agent architecture, focusing on system design, runtime behavior, tooling, safety boundaries, and operational patterns rather than as official documentation for any specific product.

If you read it in that spirit, the value is:

- clearer ownership boundaries
- more accurate mental models
- better onboarding into a complex agent runtime
- fewer incorrect assumptions when discussing advanced coding-agent systems
