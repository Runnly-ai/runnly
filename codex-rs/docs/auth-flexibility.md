# Auth and Login Flow Map

This note records where Codex currently handles authentication, where the
OpenAI-specific assumptions live, and which layers need to change if the
project should support another OpenAI-compatible provider such as DeepSeek
without forcing the built-in OpenAI login flow.

## High-Level Flow

1. `codex-rs/cli/src/login.rs` exposes the CLI login entry points.
2. `codex-rs/login/src/auth/manager.rs` loads, stores, validates, and refreshes
   auth state.
3. `codex-rs/model-provider/src/auth.rs` converts the active auth snapshot into
   request headers.
4. `codex-rs/model-provider/src/provider.rs` adapts the configured provider into
   API client state and account metadata.
5. `codex-rs/app-server/src/request_processors/account_processor.rs` and the TUI
   onboarding screens decide whether login is required and which auth methods
   are exposed to the user.

## Auth Modes Already Supported

The auth manager already understands multiple auth shapes:

- `ApiKey`
- `Chatgpt`
- `ChatgptAuthTokens`
- `AgentIdentity`

The important distinction is that the codebase treats some of these as
OpenAI-backed auth and some as generic bearer-token auth.

## OpenAI-Specific Assumptions

These are the main places that still encode OpenAI/ChatGPT behavior:

- `codex-rs/model-provider-info/src/lib.rs`
  - `requires_openai_auth` controls whether the app presents login as required.
  - `chatgpt_base_url` is a first-class config value.
- `codex-rs/login/src/auth/manager.rs`
  - Handles ChatGPT OAuth/device-code flows and token refresh against
    `auth.openai.com`.
  - Reads/writes the auth storage format used by the OpenAI login system.
- `codex-rs/app-server/src/request_processors/account_processor.rs`
  - `account/login/start`, `account/read`, `account/rateLimits/read`, and
    related endpoints expose OpenAI-specific auth behavior.
  - Some endpoints require `auth.uses_codex_backend()`.
- `codex-rs/model-provider/src/provider.rs`
  - Builds provider account state and conditionally reports that auth is
    required.
- `codex-rs/core/src/connectors.rs`
  - Several connector-related paths only work when auth is considered
    Codex/OpenAI backend auth.

## What Controls Whether Login Is Required

The biggest switch is `ModelProviderInfo.requires_openai_auth`.

- If `true`, the UI and app-server behave as though OpenAI auth is required.
- If `false`, login screens are skipped for that provider and auth can come from
  the provider config or environment.

That means a DeepSeek-style provider should usually be configured with
`requires_openai_auth = false` and use its own bearer token / base URL path.

## Likely Change Strategy For A New Provider

If the goal is to support another OpenAI-compatible model provider without the
OpenAI login UX, the smallest path is usually:

1. Keep `codex-rs/login` for existing OpenAI/ChatGPT users.
2. Add provider-specific auth in `codex-rs/model-provider/src/auth.rs` or via
   `ModelProviderInfo.env_key` / `experimental_bearer_token` / `auth`.
3. Set `requires_openai_auth = false` for non-OpenAI providers.
4. Remove or relax any feature code that explicitly requires
   `auth.uses_codex_backend()` if that feature should work with the new provider.
5. Update account/login UI and app-server RPCs so they no longer imply that
   every provider needs OpenAI-managed login.

## DeepSeek Example

This repo already includes a built-in DeepSeek provider entry, so the practical
configuration is very close to the Node SDK example you shared:

```toml
[model_providers.deepseek]
name = "DeepSeek"
base_url = "https://api.deepseek.com"
api_key_file = "~/.runnly/secrets/deepseek-api-key"
requires_openai_auth = false
wire_api = "responses"
```

With that in place, the runtime behavior is:

- the model provider reads `~/.runnly/secrets/deepseek-api-key`
- request headers use `Authorization: Bearer <token>`
- requests go to `https://api.deepseek.com`
- the OpenAI login screen is skipped because `requires_openai_auth = false`

## Places To Inspect Next

- `codex-rs/model-provider-info/src/lib.rs`
- `codex-rs/model-provider/src/provider.rs`
- `codex-rs/model-provider/src/auth.rs`
- `codex-rs/login/src/auth/manager.rs`
- `codex-rs/app-server/src/request_processors/account_processor.rs`
- `codex-rs/tui/src/onboarding/auth.rs`
- `codex-rs/cli/src/login.rs`

## Practical Summary

The codebase already has a provider abstraction, but the auth/login UX is still
centered on OpenAI/ChatGPT. For a DeepSeek-compatible fork, the main work is not
in the request path itself; it is in removing the assumption that provider auth
must go through Codex-managed OpenAI login.
