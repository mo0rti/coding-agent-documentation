# Commands, Skills, And Plugin Loading

## Scope

This document covers how slash commands and skills are represented and loaded.

Primary source files:

- `types/command.ts`
- `commands.ts`
- `skills/bundledSkills.ts`
- `skills/bundled/index.ts`
- `skills/loadSkillsDir.ts`
- `utils/plugins/loadPluginCommands.ts`
- `plugins/builtinPlugins.ts`
- `plugins/bundled/index.ts`
- `tools/SkillTool/SkillTool.ts`

## The Key Architectural Fact

This codebase does not treat "commands" and "skills" as completely separate systems.

Instead, they share the same `Command` type and differ mostly by:

- command subtype
- source / loaded-from metadata
- whether they are user-invocable
- whether the model may invoke them
- whether they run locally or expand into prompt content

## The `Command` Type (`types/command.ts`)

The command system has three concrete command forms:

- `type: 'prompt'`
- `type: 'local'`
- `type: 'local-jsx'`

### `prompt` commands

These are the most important for the skill system.

They define:

- `getPromptForCommand(args, context)`
- optional `allowedTools`
- optional `model`
- optional `hooks`
- optional `skillRoot`
- optional execution `context` of `inline` or `fork`
- optional `agent`
- optional `effort`
- source metadata such as `builtin`, `bundled`, `plugin`, `mcp`, or settings-based sources

This is the representation used by bundled skills, file-based skills, plugin skills, and MCP skills.

### `local` commands

These lazy-load an implementation module and execute locally.

### `local-jsx` commands

These lazy-load UI-producing command implementations and run inside the interactive environment.

## `commands.ts`: The Registry Aggregator

`commands.ts` is the main command assembly point.

It combines:

- hardcoded built-in commands from `COMMANDS()`
- bundled skills
- built-in plugin skills
- file-based skills from the current cwd/settings scopes
- workflow-backed commands
- marketplace/plugin commands
- plugin skills
- dynamic skills discovered later during the session

Two important implementation details:

- expensive loading is memoized by cwd
- availability and `isEnabled()` checks are re-run on every `getCommands(cwd)` call so auth and feature changes can take effect mid-session

## Built-In Commands Versus Skills

`COMMANDS()` in `commands.ts` is the built-in slash-command registry.

This is where commands such as `/clear`, `/config`, `/doctor`, `/permissions`, `/tasks`, `/plugin`, `/resume`, `/model`, `/review`, and many others come from.

These are not all skills.

Skills are the `prompt`-command subset that expand into model-facing prompt content.

## Bundled Skills (`skills/bundledSkills.ts`)

Bundled skills are registered programmatically at startup.

A bundled skill definition can provide:

- name and description
- aliases
- `whenToUse`
- `allowedTools`
- model override
- hooks
- execution context (`inline` or `fork`)
- optional extracted reference files
- `getPromptForCommand(...)`

When a bundled skill includes reference files, the implementation lazily extracts them to disk on first use and prefixes the generated prompt with a base-directory hint so the model can read those files on demand.

## Bundled Skill Initialization (`skills/bundled/index.ts`)

Bundled skills are registered by `initBundledSkills()`.

Verified examples include:

- `updateConfig`
- `keybindings`
- `verify`
- `debug`
- `remember`
- `simplify`
- `batch`
- `stuck`

Additional bundled skills are feature-gated.

Important startup detail:

- `main.tsx` explicitly calls `initBundledSkills()` before the parallel `getCommands(...)` load begins

That ordering is intentional so command memoization does not capture an empty bundled-skill list.

## File-Based Skills (`skills/loadSkillsDir.ts`)

This module loads skills from configured filesystem locations.

Verified source categories include:

- managed policy location
- user config home
- project `.claude/skills`
- legacy command paths
- plugin-provided skill directories

Important behavior in this loader:

- frontmatter is parsed and validated
- markdown body can supply descriptions when frontmatter does not
- aliases, arguments, hooks, tool allowlists, model overrides, effort, shell settings, and path filters are all derived from frontmatter
- duplicate files are suppressed using canonicalized real paths
- path-scoped visibility is supported
- model-invocation can be disabled per skill

This is the main "skill file to prompt command" compiler.

## Plugin Commands And Plugin Skills (`utils/plugins/loadPluginCommands.ts`)

Plugin-provided commands and skills are loaded separately.

### `getPluginCommands()`

Loads local or marketplace plugin command files from enabled plugins.

### `getPluginSkills()`

Loads prompt-style skills from enabled plugins.

Both are memoized and both honor `--bare` by skipping marketplace autoload unless inline plugins were explicitly supplied.

Within each enabled plugin:

- default directories are loaded first
- additional manifest-declared paths are loaded next
- duplicate file paths are suppressed per plugin
- failures are logged and isolated so one bad plugin does not abort overall loading

## Built-In Plugins

There is a built-in plugin registry in `plugins/builtinPlugins.ts`.

It is separate from bundled skills because built-in plugins are meant to be user-toggleable in the `/plugin` UI and can contribute:

- skills
- hooks
- MCP servers

Important current-state note:

- `plugins/bundled/index.ts` currently initializes no built-in plugins yet

So the built-in plugin mechanism exists, but in the current code it is scaffolding more than a major active source of features.

## How `getCommands(cwd)` Actually Builds The Final Command Set

The verified order is:

1. bundled skills
2. built-in plugin skills
3. file-based skill-directory commands
4. workflow commands
5. plugin commands
6. plugin skills
7. hardcoded built-in commands

After that:

- auth/provider availability filtering runs
- `isEnabled()` filtering runs
- dynamic skills are inserted before the built-in command block

That means the command list is intentionally layered, not just concatenated arbitrarily.

## `SkillTool`: Where Skills Reach The Model

`tools/SkillTool/SkillTool.ts` is the bridge from the tool system into the command/skill system.

Important verified behavior:

- it gathers local commands through `getCommands(getProjectRoot())`
- it also adds MCP-provided prompt commands from `appState.mcp.commands` when `loadedFrom === 'mcp'`
- it only treats MCP skills, not generic MCP prompts, as invocable through `SkillTool`
- it can execute skills inline or in a forked sub-agent depending on command metadata

This is the main reason commands and skills must be documented together: `SkillTool` operates on the command registry, not on a separate "skill registry" type.

## Practical Mental Model

Use this simplified model when reading the code:

- built-in slash commands live in `commands.ts`
- bundled and file-based skills are compiled into `prompt` commands
- plugin commands and plugin skills are loaded in parallel from enabled plugins
- MCP can contribute prompt commands too
- `getCommands(cwd)` builds the user-visible command universe
- `SkillTool` selects the model-invocable skill subset of that universe

## Corrections To The Existing Docs

- Commands and skills are not two independent registries. They converge on the shared `Command` type.
- File-based skills are compiled from markdown plus frontmatter, not just loaded as static prompt blobs.
- Built-in plugins are a separate mechanism from bundled skills, and the current built-in-plugin initializer is mostly scaffold.
- `SkillTool` can invoke local, bundled, plugin, and MCP-loaded prompt skills, but not arbitrary MCP prompts.

