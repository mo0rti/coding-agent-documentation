# Authentication, Provider Resolution, And Credential Helpers

## Scope

This document covers how the CLI decides which authentication path is active, how first-party OAuth and API-key flows interact with third-party provider modes, how credential helper commands are cached and trust-gated, and how provider selection affects model resolution.

Primary source files:

- `utils/auth.ts`
- `utils/authFileDescriptor.ts`
- `utils/authPortable.ts`
- `constants/oauth.ts`
- `services/oauth/client.ts`
- `services/oauth/index.ts`
- `cli/handlers/auth.ts`
- `commands/login/login.tsx`
- `commands/logout/logout.tsx`
- `services/api/client.ts`
- `services/api/withRetry.ts`
- `utils/aws.ts`
- `utils/awsAuthStatusManager.ts`
- `utils/model/providers.ts`
- `utils/model/configs.ts`
- `utils/model/modelStrings.ts`
- `utils/model/model.ts`
- `utils/model/modelOptions.ts`
- `utils/model/validateModel.ts`
- `utils/status.tsx`
- `main.tsx`

## The Key Architectural Split

The code does not have one unified "auth system".

There are three related but separate decision layers:

1. first-party Anthropic auth source selection
2. provider transport selection for Bedrock, Vertex, Foundry, or first-party
3. model-name resolution for the selected provider

Those layers interact, but they are not the same thing.

For example:

- a user can be in first-party provider mode but authenticate with a direct API key, `apiKeyHelper`, or Claude.ai OAuth
- a user can be in Bedrock or Vertex mode, in which case first-party Anthropic OAuth is intentionally disabled
- model aliases like `opus` and `sonnet` resolve differently depending on subscription tier and provider

So the correct mental model is not "login determines everything". Login only affects a subset of the overall runtime.

## Provider Selection Is Environment-Driven

`utils/model/providers.ts` makes the selection rule explicit.

`getAPIProvider()` is determined only from environment flags:

- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`

If none are set, the provider is `firstParty`.

This matters because provider selection is not computed from account type or OAuth state. It is an environment/runtime choice.

Settings may still influence those environment variables indirectly through managed env application, but the provider resolver itself only reads `process.env`.

## First-Party Anthropic Auth Is Intentionally Disabled In Several Contexts

`isAnthropicAuthEnabled()` in `utils/auth.ts` is the gate for direct first-party auth support.

Verified behavior:

- `--bare` disables OAuth completely
- Bedrock, Vertex, and Foundry disable Anthropic auth completely
- external API-key-based auth disables Anthropic auth unless the process is in a managed OAuth context
- external bearer-token auth disables Anthropic auth unless the process is in a managed OAuth context
- SSH-style remote auth with `ANTHROPIC_UNIX_SOCKET` is a special case and only enables Anthropic auth when `CLAUDE_CODE_OAUTH_TOKEN` is present as the local-side placeholder

The managed OAuth context rule is important.

`isManagedOAuthContext()` exists specifically so Claude Desktop and remote-managed sessions do not accidentally fall back to the user's terminal-side `~/.claude/settings.json`, `ANTHROPIC_API_KEY`, or `ANTHROPIC_AUTH_TOKEN`.

So the code is deliberately preventing cross-context credential leakage, not merely picking the first available token.

## Auth Token Source And API Key Source Are Separate Resolution Paths

The code treats "auth token" and "API key" as different lookup problems.

### Auth token source

`getAuthTokenSource()` resolves bearer-token-style auth in this order:

1. bare-mode `apiKeyHelper` from `flagSettings` only
2. `ANTHROPIC_AUTH_TOKEN` when not in managed OAuth context
3. `CLAUDE_CODE_OAUTH_TOKEN`
4. OAuth token file descriptor or CCR fallback token file
5. configured `apiKeyHelper` without executing it
6. saved Claude.ai OAuth tokens in secure storage

This resolution returns metadata even when the code has not executed `apiKeyHelper` yet.

That is intentional so the runtime can know "an auth helper exists" without running arbitrary commands before trust is established.

### API key source

`getAnthropicApiKeyWithSource()` resolves API-key-style auth differently:

1. bare mode only allows `ANTHROPIC_API_KEY` or `apiKeyHelper` from `flagSettings`
2. in preferred third-party auth contexts, direct `ANTHROPIC_API_KEY` wins immediately
3. CI/test mode requires either `ANTHROPIC_API_KEY` or OAuth env input
4. approved `ANTHROPIC_API_KEY` values can win before file descriptor or login-managed key fallback
5. API key file descriptor can provide the key
6. configured `apiKeyHelper` wins over stored keychain/config values
7. login-managed key from config or macOS keychain is used last

The helper path is also intentionally non-blocking:

- if `apiKeyHelper` is configured but not fetched yet, the source still resolves as `apiKeyHelper`
- callers that need the actual value must await `getApiKeyFromApiKeyHelper(...)`

So source detection and credential retrieval are intentionally decoupled.

## CCR And Managed Sessions Have A Separate FD-And-File Credential Path

`utils/authFileDescriptor.ts` handles credentials injected by CCR and similar managed environments.

Verified behavior:

- OAuth tokens and API keys can be injected via file descriptor environment variables
- if FD reading succeeds in CCR, the token is also persisted to a well-known file under `/home/claude/.claude/remote/`
- subprocesses can then recover the token from disk when they cannot inherit the original pipe FD
- the result is cached in bootstrap state

This is not a general credential-storage mechanism.

It exists to solve a very specific managed-session problem:

- pipe FDs do not survive all subprocess boundaries
- remote subprocesses still need credentials

So CCR credential propagation is its own compatibility subsystem, not part of normal local auth.

## API Key Helpers Use Stale-While-Revalidate Semantics

The `apiKeyHelper` path in `utils/auth.ts` is more sophisticated than "run a shell command".

Verified behavior:

- cached values use a TTL from `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`, defaulting to 5 minutes
- stale values are returned immediately while a background refresh runs
- cold-cache reads are deduplicated through a shared in-flight promise
- cache invalidation bumps an epoch so orphaned in-flight executions cannot overwrite newer state
- helper failures during warm refresh do not necessarily discard a previously working cached key
- a sentinel value of `' '` is cached after hard failure so callers do not incorrectly fall back to OAuth

This is a real cache design, not a convenience wrapper.

The code is explicitly trying to:

- avoid blocking the UI
- avoid repeated helper execution
- avoid accidental auth-mode flips during transient helper failures

## Helper And Cloud Refresh Commands Are Trust-Gated

Several credential-related commands can come from settings:

- `apiKeyHelper`
- `awsAuthRefresh`
- `awsCredentialExport`
- `gcpAuthRefresh`
- `otelHeadersHelper`

For each of these, the code checks whether the configured command came from:

- `projectSettings`
- `localSettings`

If so, interactive execution is blocked until workspace trust is accepted.

This shows up in:

- `_executeApiKeyHelper(...)`
- `runAwsAuthRefresh()`
- `getAwsCredsFromCredentialExport()`
- `runGcpAuthRefresh()`
- `getOtelHeadersFromHelper()`

So trust gating is not limited to hooks and tools. Project-scoped auth helpers are treated as dangerous executable input too.

## AWS Bedrock Auth Uses Two Separate Helper Hooks

The Bedrock path in `utils/auth.ts` supports two different settings-driven helpers:

- `awsAuthRefresh` for interactive login refresh such as `aws sso login`
- `awsCredentialExport` for producing STS credentials as JSON

Verified behavior:

- Bedrock first probes `sts:GetCallerIdentity` before running either helper
- if AWS credentials are already valid, refresh/export commands are skipped
- `awsAuthRefresh` streams stdout and stderr through `AwsAuthStatusManager`
- `awsCredentialExport` must return valid STS-shaped JSON
- when refresh or export succeeds, `clearAwsIniCache()` forces the AWS SDK provider cache to pick up new credentials
- the combined `refreshAndGetAwsCredentials` path is memoized with a one-hour TTL

This means Bedrock auth is not simply "let the AWS SDK handle it".

The CLI adds its own:

- trust gate
- validity probe
- interactive refresh command support
- credential export command support
- UI-visible authentication status stream

## Vertex Auth Has Its Own Refresh And Probe Path

The Vertex path is parallel to Bedrock, but not identical.

Verified behavior:

- `gcpAuthRefresh` is an optional settings-driven refresh command
- `checkGcpCredentialsValid()` uses `google-auth-library` with a 5-second timeout to avoid long metadata-server hangs outside GCP
- `refreshGcpAuth(...)` streams output through the same `AwsAuthStatusManager`
- `refreshGcpCredentialsIfNeeded` is memoized with a one-hour TTL
- prefetch is available through `prefetchGcpCredentialsIfSafe()`

So the "AWS auth status manager" is actually cloud-provider-agnostic runtime state shared between Bedrock and Vertex auth refresh flows.

## Provider Prefetch Happens After First Render, Not During Early Init

`main.tsx` shows that AWS and GCP credential prefetch is deferred until after the initial render path.

Verified behavior:

- bare mode skips all of these prefetches
- Bedrock prefetch uses `prefetchAwsCredentialsAndBedRockInfoIfSafe()`
- Vertex prefetch uses `prefetchGcpCredentialsIfSafe()`
- both are skipped when their respective `CLAUDE_CODE_SKIP_*_AUTH` env flags are set

This matches the general architecture used elsewhere in the app:

- non-critical but potentially slow work is deferred until the REPL is already visible

## Login Has Two Different First-Party Flows

`cli/handlers/auth.ts` implements two separate CLI login paths.

### Refresh-token login

If `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` is present:

- the browser flow is skipped entirely
- `CLAUDE_CODE_OAUTH_SCOPES` is required
- the refresh token is exchanged directly for live tokens
- tokens are installed through `installOAuthTokens(...)`
- forced-organization validation runs afterward

### Browser OAuth login

Otherwise:

- `OAuthService.startOAuthFlow(...)` performs PKCE-based browser or manual-code login
- `forceLoginMethod` from settings can force Console versus Claude.ai login method
- `forceLoginOrgUUID` is passed into the OAuth flow and later validated server-side
- `--console` and `--claudeai` are only soft selectors when policy does not force one

So "claude auth login" is not one fixed flow. It has an env-driven refresh-token fast path and an interactive PKCE path.

## Installing OAuth Tokens Does More Than Save Them

`installOAuthTokens(...)` in `cli/handlers/auth.ts` is the post-login orchestration point.

Verified behavior:

1. it first performs logout-like cleanup through `performLogout({ clearOnboarding: false })`
2. it stores account info from the fetched profile or token exchange payload
3. it saves OAuth tokens to secure storage when appropriate
4. it clears OAuth token caches
5. it fetches and stores user roles on a best-effort basis
6. Claude.ai-style logins fetch and store first-token-date on a best-effort basis
7. Console/API-style logins create and store an API key as a required step
8. it clears auth-related caches afterward

That means successful login is really a bundle of state transitions:

- secure credentials
- config account metadata
- roles
- optional API key creation
- cache invalidation

## Interactive `/login` Also Refreshes Runtime State Beyond Credentials

The TUI command in `commands/login/login.tsx` does additional post-login work after auth succeeds.

Verified behavior:

- resets cost state
- refreshes remote managed settings
- refreshes policy limits
- resets cached user data
- refreshes GrowthBook feature flags
- clears and re-enrolls the trusted device token for bridge/remote control
- re-runs bypass/auto-mode killswitch checks with the new org
- increments `AppState.authVersion`
- strips signature-bearing message blocks so old API-key-bound signatures do not survive account switches

So login is not just credential replacement. It is a runtime identity transition that forces dependent subsystems to reload.

## Logout Also Clears More Than Credentials

`commands/logout/logout.tsx` shows similarly broad cleanup.

Verified behavior:

- telemetry is flushed before credentials are cleared to avoid org-data leakage
- `removeApiKey()` clears both config and macOS keychain storage
- secure storage is wiped
- OAuth, trusted-device, user, Grove, remote-managed-settings, policy-limit, beta, and tool-schema caches are cleared
- onboarding state may also be reset depending on the caller

So logout is not only "delete token". It is a deliberate full auth-context reset.

## Saved OAuth Tokens Are Restricted To Claude.ai-Style Sessions

`saveOAuthTokensIfNeeded(...)` only persists tokens when they are suitable for Claude.ai subscriber auth.

Verified behavior:

- non-Claude.ai scoped tokens are not persisted
- inference-only tokens without refresh token or expiry are not persisted
- secure storage is used as the backing store
- stored `subscriptionType` and `rateLimitTier` are not overwritten with `null` on transient refresh/profile failures

This explains why env-provided inference-only tokens behave differently from real logged-in sessions:

- they can authenticate requests
- but they do not become durable local login state

## OAuth Refresh Is Cross-Process Safe

The OAuth refresh path in `utils/auth.ts` is designed for multiple concurrent CLI processes.

Verified behavior:

- `.credentials.json` mtime is checked to detect when another process has changed credentials on disk
- memoized token caches are cleared if the on-disk credentials changed
- concurrent 401 handlers for the same failed access token are deduplicated
- refresh attempts acquire a lock under the Claude config directory
- if another process already refreshed the token, the current process reuses that result rather than refreshing again
- forced refresh can happen on server-side 401 or revoked-token errors even when local expiry checks do not think refresh is needed

So token refresh is explicitly built for multi-process safety, not only single-process correctness.

## The First-Party Client Chooses OAuth Versus API Key At Request Time

`services/api/client.ts` is the source of truth for actual SDK client construction.

For the first-party provider:

- the client always runs `checkAndRefreshOAuthTokenIfNeeded()` first
- if the user is a Claude.ai subscriber, the SDK client gets `authToken`
- otherwise the SDK client gets `apiKey`
- non-subscriber first-party clients may also receive an `Authorization` bearer header from `ANTHROPIC_AUTH_TOKEN` or `apiKeyHelper`

That means the runtime request path does not simply reuse the login command's notions of auth. It recomputes what headers and SDK constructor arguments should be used for the current request.

## Each Provider Has A Distinct Client Construction Path

`services/api/client.ts` dispatches into four distinct SDK paths.

### Bedrock

Verified behavior:

- uses `@anthropic-ai/bedrock-sdk`
- selects region from AWS env helpers, with a separate override for the small fast model
- can skip auth entirely for testing/proxy flows
- can use `AWS_BEARER_TOKEN_BEDROCK` bearer auth instead of STS credentials
- otherwise injects cached/refreshed AWS credentials into the SDK

### Foundry

Verified behavior:

- uses `@anthropic-ai/foundry-sdk`
- if `ANTHROPIC_FOUNDRY_API_KEY` is present, SDK API-key auth is used
- otherwise Azure AD auth is used through `DefaultAzureCredential`
- skip-auth mode can inject a mock token provider for proxy/testing scenarios

### Vertex

Verified behavior:

- uses `@anthropic-ai/vertex-sdk`
- refreshes GCP credentials first unless skip-auth is set
- creates a fresh `GoogleAuth` instance per client build
- supplies `projectId` only as a last-resort fallback from `ANTHROPIC_VERTEX_PROJECT_ID`
- this last-resort fallback exists specifically to avoid metadata-server hangs when project discovery would otherwise fall through too far
- region selection is model-specific

### First-party

Verified behavior:

- uses `@anthropic-ai/sdk`
- subscriber sessions prefer OAuth token auth
- API customers prefer API key auth
- staging OAuth can also override the base API URL

So "the Anthropic client" is actually a provider-specific factory, not one SDK wrapper with minor option changes.

## Retry Logic Knows About Provider-Specific Auth Failures

`services/api/withRetry.ts` has provider-specific auth recovery branches.

Verified behavior:

- Bedrock auth failures clear AWS credential caches
- Vertex auth failures clear GCP credential caches
- first-party `401` clears the `apiKeyHelper` cache
- OAuth revoked-token errors are treated as retryable
- CCR-managed `401` and `403` are treated as transient infrastructure blips and retried even when normal first-party rules might not retry

This means auth recovery is not fully encapsulated inside `utils/auth.ts`. The retry layer also participates in cache invalidation and retry policy.

## Model Resolution Is Provider-Aware At Multiple Layers

The model subsystem is not a flat alias table.

### Canonical model config

`utils/model/configs.ts` defines canonical first-party model IDs plus provider-specific IDs for:

- `firstParty`
- `bedrock`
- `vertex`
- `foundry`

This is the baseline mapping layer.

### Provider-specific runtime strings

`utils/model/modelStrings.ts` turns those configs into runtime strings.

Verified behavior:

- non-Bedrock providers use built-in provider mappings synchronously
- Bedrock fetches system-defined inference profiles in the background
- Bedrock model strings can upgrade from hardcoded IDs to inference-profile IDs when matches are found
- `settings.modelOverrides` can replace provider-derived model IDs with custom IDs such as ARNs
- `resolveOverriddenModel(...)` maps custom override values back to canonical IDs when needed for display or feature logic

So the effective model string for a request may not be the built-in provider string at all.

## Defaults And Picker Options Depend On Both Subscription And Provider

`utils/model/model.ts` and `utils/model/modelOptions.ts` together show that model choice is influenced by:

- provider
- subscription tier
- 1M-context availability
- fast mode
- custom env-based model overrides

Verified behavior:

- first-party Max and Team Premium default to Opus
- other subscriber and API users default to Sonnet
- third-party providers can intentionally lag first-party defaults
- 1M variants are only exposed when access checks pass
- 3P custom env defaults can replace the built-in Sonnet, Opus, or Haiku picker entries
- the `/model` picker preserves custom current models even when they are not standard options

So model selection is not only an alias resolver. It is a tier-aware product policy layer.

## `/model` Validation Uses Real API Calls For Custom Models

`utils/model/validateModel.ts` does not validate custom model names by string pattern alone.

Verified behavior:

- known aliases are accepted without network calls
- allowlist restrictions are checked first
- custom models are validated by making a real side query
- 3P providers can return fallback suggestions such as older provider-specific model IDs when a new one is unavailable

So custom model validation is request-backed, not schema-backed.

## Account And Status Reporting Are Intentionally Narrow

`getAccountInformation()` in `utils/auth.ts` only returns account data for the `firstParty` provider.

That means status surfaces intentionally separate:

- first-party account identity
- third-party provider identity

`authStatus(...)` and `utils/status.tsx` then present:

- login method or token source
- API key source
- provider name
- provider-specific configuration like AWS region, GCP project, Foundry resource, proxy, and auth-skip flags

So the status UI is deliberately split between account auth and transport/provider environment.

## Corrections To The Existing Mental Model

- "Authentication" is not one subsystem. The code separates first-party auth-source selection, provider selection, and provider-specific client construction.
- Provider choice is env-driven, not inferred from login state or account type.
- Claude.ai OAuth is intentionally disabled when external API keys, external bearer auth, or third-party providers are active, except for managed-session edge cases.
- `apiKeyHelper` is not a naive command execution path. It uses trust gating, stale-while-revalidate caching, deduplicated cold starts, and a sentinel failure value to prevent accidental fallback.
- Bedrock and Vertex auth each have their own refresh/probe/cache logic, and the retry layer also participates in clearing those caches.
- Logged-in OAuth state is not just "token saved somewhere". It includes secure storage, account metadata, role fetching, optional API-key creation, and broad runtime cache invalidation.
- Model resolution is not just aliases in `model.ts`. Provider-specific IDs, Bedrock inference-profile discovery, and settings-based `modelOverrides` all participate in the effective model used at runtime.
