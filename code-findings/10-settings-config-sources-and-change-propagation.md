# Settings, Config Sources, And Change Propagation

## Scope

This document covers how settings are sourced, parsed, merged, cached, watched, and applied to runtime state.

Primary source files:

- `utils/settings/settings.ts`
- `utils/settings/constants.ts`
- `utils/settings/settingsCache.ts`
- `utils/settings/changeDetector.ts`
- `utils/settings/applySettingsChange.ts`
- `utils/settings/validation.ts`
- `utils/settings/internalWrites.ts`
- `utils/settings/managedPath.ts`
- `utils/settings/pluginOnlyPolicy.ts`
- `utils/settings/mdm/settings.ts`
- `utils/settings/types.ts`
- `utils/settings/validateEditTool.ts`
- `state/AppState.tsx`
- `state/onChangeAppState.ts`
- `main.tsx`
- `cli/print.ts`
- `services/remoteManagedSettings/index.ts`
- `services/remoteManagedSettings/syncCacheState.ts`

## The Key Architectural Split

The code does not treat "settings" as only JSON parsing.

There are four different layers:

1. source resolution and precedence
2. parsing, validation, and merge rules
3. cache and file-change detection
4. runtime application of changed settings into app state and side effects

That split matters because a value being present on disk is not the same as:

- being included in the effective merged settings
- winning precedence against another source
- being watched for changes
- having its runtime side effects applied yet

## The Settings Cascade Has More Than Four Sources

`utils/settings/constants.ts` defines the nominal settings sources:

- `userSettings`
- `projectSettings`
- `localSettings`
- `flagSettings`
- `policySettings`

But `settings.ts` also inserts a plugin-provided base layer before those sources.

The effective merge shape is:

1. plugin settings base
2. enabled user/project/local sources
3. `flagSettings`
4. `policySettings`

With one major exception:

- `policySettings` is not merged the same way as ordinary sources; it uses a first-source-wins model internally

So the real precedence model is more nuanced than the usual "user/project/local/policy" summary.

## `--setting-sources` Does Not Control Every Source

`getEnabledSettingSources()` is load-bearing here.

Verified behavior:

- bootstrap state decides which ordinary sources are enabled
- `--setting-sources` only affects `userSettings`, `projectSettings`, and `localSettings`
- `policySettings` is always included
- `flagSettings` is always included

So a user can narrow normal file-backed sources, but they cannot opt out of managed policy or CLI/SDK flag settings through `--setting-sources`.

## Source Paths Have Different Semantics

`settings.ts` resolves each source to a different root and file path.

Important verified behavior:

- `userSettings` lives under the Claude config home
- `projectSettings` maps to `.claude/settings.json` under the original cwd
- `localSettings` maps to `.claude/settings.local.json` under the original cwd
- `flagSettings` can come from a path provided by CLI or SDK
- `policySettings` maps to the managed settings location for the platform

There is also a user-settings filename switch:

- cowork mode uses `cowork_settings.json`
- otherwise it uses `settings.json`

So even the user-settings filename is not globally fixed.

## Managed Policy Settings Use Their Own Precedence Model

`getSettingsForSource('policySettings')` does not merge all policy-like sources together.

It uses first-source-wins precedence:

1. remote managed settings
2. admin MDM settings (`HKLM` on Windows or plist on macOS)
3. file-based managed settings (`managed-settings.json` and drop-ins)
4. `HKCU` policy settings on Windows

The same first-source-wins model appears in:

- `getPolicySettingsOrigin()`
- `loadSettingsFromDisk()` when it merges the overall settings cascade
- `utils/settings/mdm/settings.ts`

So policy is not "merge all enterprise inputs together". The highest-priority available source becomes the active policy source.

## File-Based Managed Settings Have Their Own Internal Merge Rule

Within the file-based policy source, `loadManagedFileSettings()` uses a different merge rule:

- `managed-settings.json` is the base
- `managed-settings.d/*.json` files are loaded alphabetically
- later drop-ins override earlier ones

That means there are two precedence layers inside policy settings:

1. choose the winning policy source
2. if file-based policy won, merge base file plus drop-ins inside that source

## Remote Managed Settings Are Cached And Hot-Reloaded

The remote managed settings path is split between:

- `services/remoteManagedSettings/syncCacheState.ts`
- `services/remoteManagedSettings/index.ts`

Important verified behavior:

- the sync cache is a leaf module so settings reads can stay synchronous
- remote settings are stored in a local cache file
- `getRemoteManagedSettingsSyncFromCache()` only returns data when eligibility has already been established
- when cached remote settings first become visible, it resets the merged settings cache so future reads include the policy layer
- `loadRemoteManagedSettings()` and background polling call `settingsChangeDetector.notifyChange('policySettings')` to feed into the normal hot-reload path

So remote managed settings are not a separate runtime config system. They eventually flow through the same settings cache reset and app-state refresh mechanism as file changes.

## Parsing And Validation Are Intentionally Lenient In A Few Places

`parseSettingsFile(...)` validates with `SettingsSchema()`, but it does not fail hard on every malformed detail.

The most important special case is permission rules:

- `filterInvalidPermissionRules(...)` removes invalid permission rules from raw JSON before schema validation
- warnings are recorded for those dropped rules
- the rest of the settings file can still load

This is a deliberate design choice:

- one bad permission rule should not poison the entire settings file

So effective settings can still be produced even when validation warnings exist.

## Validation Errors Are Structured And User-Facing

`utils/settings/validation.ts` turns Zod errors into structured `ValidationError` objects with:

- `file`
- `path`
- `message`
- `expected`
- `invalidValue`
- optional suggestions and docs links

`main.tsx` then surfaces non-MCP validation errors after trust is established in interactive mode.

There is also a separate leaf helper in `utils/settings/allErrors.ts` that combines normal settings errors with MCP config errors when a caller needs the full set.

So the code distinguishes:

- effective settings
- settings validation errors
- MCP config errors

rather than flattening them into one generic failure mode.

## The Merged Settings Cache Is Session-Scoped

`settingsCache.ts` defines three different caches:

- merged session settings cache
- per-source settings cache
- parsed-file cache

This is important because the code optimizes multiple layers:

- `getInitialSettings()` / `getSettingsWithErrors()` cache the merged snapshot
- `getSettingsForSource()` caches individual resolved sources
- `parseSettingsFile()` caches file parse results so repeated startup reads do not redo disk + Zod work

The change detector resets these caches centrally through `resetSettingsCache()`.

So the cache design is part of the architecture, not an incidental optimization.

## Merge Rules Differ By Data Type And API

The code does not use one universal merge rule for everything.

### Effective settings merge

`loadSettingsFromDisk()` and `getSettingsForSource('flagSettings')` use `mergeWith(..., settingsMergeCustomizer)`.

That customizer:

- concatenates arrays
- deduplicates arrays
- uses lodash default merge behavior for objects and scalars

### File update merge

`updateSettingsForSource(...)` uses different semantics for writes:

- arrays are replaced, not concatenated
- `undefined` means delete this key
- objects are otherwise deep-merged

That difference is easy to miss but important:

- runtime merge of sources is additive for arrays
- editing a concrete source file is replacement-oriented for arrays

## Editable Sources Are Restricted

`updateSettingsForSource(...)` only writes to editable sources:

- `userSettings`
- `projectSettings`
- `localSettings`

It intentionally ignores writes to:

- `policySettings`
- `flagSettings`

Additional verified behavior:

- it creates parent directories as needed
- for `localSettings`, it asynchronously adds the file to `.gitignore`
- if validation failed previously but the file still contains valid JSON syntax, it may fall back to the raw object rather than clobbering the file

So the write path is cautious about both mutability and preserving user data.

## Plugin Settings Are A Real Base Layer

`settingsCache.ts` includes `pluginSettingsBase`, and `loadSettingsFromDisk()` merges that base before any normal setting source.

This means plugin-contributed settings are not treated like a separate post-merge patch. They are the lowest-priority base of the settings cascade.

That is a meaningful architectural fact because it explains why:

- regular user/project/local/flag/policy settings can override plugin defaults
- plugin settings are still part of the effective merged settings snapshot

## Strict Plugin-Only Policy Is A Loader Constraint, Not A Generic Kill Switch

`pluginOnlyPolicy.ts` implements `strictPluginOnlyCustomization`.

Verified behavior:

- the policy can be `true` or a list of locked customization surfaces
- when locked, user-level and project-level customizations are blocked for that surface
- managed, plugin, and built-in sources remain trusted

This is not only a plugin feature. It is a managed policy that affects how some customization surfaces are loaded and registered across the app.

## Settings Watchers Are Selective

`changeDetector.ts` is careful about what it watches.

Important verified behavior:

- `flagSettings` is explicitly not watched
- only directories that had at least one existing settings file at init time are watched
- all potential settings files in those directories are then recognized, so later-created siblings can still be detected
- the managed drop-in directory is watched separately when it exists
- the watcher ignores non-settings files in watched directories
- `.git` directories and special file types are ignored

This corrects a simplistic mental model of "the app watches every possible settings path all the time". It does not.

## Internal Writes Are Suppressed Explicitly

The watcher does not rely on best-effort heuristics alone.

`internalWrites.ts` tracks in-process writes by path and timestamp.

The flow is:

1. `updateSettingsForSource(...)` calls `markInternalWrite(path)`
2. chokidar fires later
3. `changeDetector.ts` calls `consumeInternalWrite(...)`
4. if the write is still inside the suppression window, the change is ignored

So self-writes are intentionally filtered out, not simply tolerated.

## Change Fan-Out Is Centralized

`fanOut(source)` in `changeDetector.ts` is one of the key architectural chokepoints.

Verified behavior:

- it resets the settings caches once
- then emits the change signal to subscribers

The code comments explain why this was centralized:

- resetting inside each subscriber caused N-way cache thrashing and repeated disk reloads

So the change detector is not just a watcher. It is the single producer of cache invalidation for settings changes.

## Programmatic And File-Based Changes Use The Same Notification Path

`settingsChangeDetector.notifyChange(source)` feeds into the same `fanOut(...)` path as chokidar events.

That means these sources all converge on the same invalidation and subscription mechanism:

- local file changes
- managed settings polling changes
- remote managed settings arrival and polling changes
- SDK-applied flag settings changes

This is one of the cleanest parts of the subsystem: different producers, one notification path.

## ConfigChange Hooks Can Block Settings Application

`changeDetector.ts` does not blindly apply every detected file change.

Verified behavior:

- before fan-out, it executes `ConfigChange` hooks
- if the hook result is blocking, the settings change is skipped for this session

So settings changes are themselves subject to hook-based lifecycle control.

## Interactive And Headless Consumers Subscribe Differently

The code has two subscription paths for runtime settings application.

### Interactive/TUI path

`state/AppState.tsx` uses `useSettingsChange(...)`, which subscribes to `settingsChangeDetector`.

On change it calls `applySettingsChange(...)` through the store's `setState`.

### Headless/SDK path

`cli/print.ts` subscribes directly because there is no React tree and `useSettingsChange` never runs.

That direct subscription also handles a headless-only sync for the denormalized `fastMode` field.

So "settings hot reload" exists in both modes, but the subscriber wiring is different.

## `applySettingsChange(...)` Does More Than Replace `state.settings`

The runtime application step is not a simple settings assignment.

Verified behavior from `applySettingsChange.ts`:

- re-read fresh merged settings
- reload permission rules from disk
- update the hooks snapshot
- sync permission context from disk
- re-apply dangerous-rule stripping and bypass disabling logic
- transition plan/auto permission mode state if needed
- optionally sync top-level effort value

So a settings change updates several subsystem snapshots, not only the visible settings object.

## `onChangeAppState(...)` Applies The Side Effects

`applySettingsChange(...)` updates app state, but `state/onChangeAppState.ts` is where many external side effects happen.

Important verified behavior:

- auth helper and cloud credential caches are cleared when settings change
- environment variables are re-applied when `settings.env` changes
- permission mode changes are forwarded to session metadata / CCR listeners
- some other UI/session preferences are persisted back out to global config

This is the second major split in the subsystem:

- `applySettingsChange(...)` updates app state
- `onChangeAppState(...)` performs operational side effects from those new values

## Environment Application Is Intentionally Phased

The entrypoints do not apply every settings-derived environment variable at the same time.

Verified behavior:

- `init.ts` applies only safe config environment variables before trust
- `main.tsx` and `cli/print.ts` apply full environment variables after trust or in trusted headless mode
- later settings changes can re-apply environment variables again through `onChangeAppState(...)`

So the environment model is phased:

- safe early boot
- full post-trust or headless application
- additive hot reload after settings changes

## Some Security-Sensitive Reads Exclude `projectSettings`

`settings.ts` has a few helper readers that intentionally exclude project settings for security reasons.

Examples include:

- dangerous-mode dialog opt-in
- auto-mode opt-in
- auto-mode-during-plan behavior
- auto-mode config assembly

The repeated theme in comments is:

- a malicious project should not be able to silently opt the user into a risky behavior

So the effective settings snapshot is not always the right source for security-sensitive decisions. Some readers deliberately consult only trusted subsets.

## Settings Validation Also Protects File Editing Workflows

`validateEditTool.ts` connects the settings schema to file-edit tools.

Verified behavior:

- if a Claude settings file was valid before edit, the tool requires it to remain valid after edit
- if the file was already invalid before the edit, the edit is allowed through

That means the schema is not only used at startup. It also actively guards tool-mediated edits of settings files.

## The SDK Has A Source-Aware Settings View

`getSettingsWithSources()` returns:

- `effective`
- ordered `sources`

`cli/print.ts` uses this for the SDK `get_settings` control response, along with an `applied` section describing the runtime-applied model/effort view.

So the system exposes not only the merged result but also the per-source breakdown that produced it.

## Corrections To The Existing Mental Model

- Settings precedence is not only user/project/local/policy. There is also a plugin base layer, `flagSettings`, and a special first-source-wins policy branch.
- `--setting-sources` does not disable managed policy or flag settings; those are always included.
- Policy settings are not merged from every enterprise source at once. The highest-priority available source wins, and only file-based policy then does an internal base-plus-drop-ins merge.
- Hot reload is not "just chokidar": cache invalidation is centralized, programmatic notifications use the same path, and `ConfigChange` hooks can block application.
- Applying settings is not only replacing `state.settings`; it also refreshes permissions, hooks, env-dependent state, and selected denormalized fields.
- Some security-sensitive decisions intentionally do not trust `projectSettings`, even though project settings participate in the ordinary effective settings merge.
