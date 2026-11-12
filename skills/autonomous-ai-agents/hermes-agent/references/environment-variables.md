# Environment Variables (cheat sheet)

Secrets go in `.env`, non-secret settings go in `config.yaml`. When both are set, `config.yaml` wins for non-secret settings.

Source of truth: official docs `docs/reference/environment-variables`. When in doubt, query it fresh — do NOT answer "Hermes can't do X" from memory.

## Providers / models

| Variable | Use |
|----------|-----|
| `OPENROUTER_API_KEY` | OpenRouter (and keys starting with `sk-or-`)
| `OPENAI_API_KEY` | OpenAI provider, or custom OpenAI-compatible endpoint when `OPENAI_BASE_URL` is set. Scoped to that origin only.
| `OPENAI_BASE_URL` | Base URL for a custom OpenAI-compatible endpoint
| `ANTHROPIC_API_KEY` | Anthropic (also `CLAUDE_CODE_OAUTH_TOKEN`)
| `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Gemini
| `DEEPSEEK_API_KEY` | DeepSeek
| `XAI_API_KEY` | xAI (Grok)
| `MISTRAL_API_KEY` | Mistral
| `NVIDIA_API_KEY` | NVIDIA NIM / Nemotron
| `NVIDIA_BASE_URL` | Override NVIDIA base URL (default integrate.api.nvidia.com/v1)
| `GLM_API_KEY` / `ZAI_API_KEY` | GLM / Zhipu
| `AZURE_FOUNDRY_KEY` | Azure Foundary
| `HERMES_OPENROUTER_CACHE` | Enable OpenRouter response cache (1/true/yes/on)
| `HERMES_OPENROUTER_CACHE_TTL` | Cache TTL seconds (1-86400)
| `AI_GATEWAY_API_KEY` | Vercel AI Gateway

Per-alias keys live in config.yaml (`key_env:` / `api_key: "${VAR}"`), not in `.env`.

## Web / browser

| Variable | Use |
|----------|-----|
| `FIRECRAWL_API_KEY` | Web scraping + cloud browser (web_extract)
| `BROWSERBASE_API_KEY` | Browserbase automation
| `BROWSERBASE_PROJECT_ID` | Browserbase project ID
| `BROWSERBASE_ADVANCED_STEALTH` | Advanced stealth (Scale plan only) — do NOT ship for users without BROWSERBASE_API_KEY
| `BROWSER_INACTIVITY_TIMEOUT` | Browser session inactivity timeout (config.yaml owns this: browser.inactivity_timeout)
| `AGENT_BROWSER_ENGINE` | `auto`/`chrome`/`lightpanda`
| `AGENT_BROWSER_ARGS` | Extra Chromium launch flags

## Messaging / gateway

| Variable | Use |
|----------|-----|
| `TELEGRAM_BOT_TOKEN` | Telegram bot token
| `TELEGRAM_ALLOWED_USERS` | Comma-separated user IDs

## Dashboard auth

| Variable | Use |
|----------|-----|
| `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` | Username — NOT the old wrong name `HERMES_DASHBOARD_USER`
| `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD` | Plaintext password (hashed in-memory; use `_PASSWORD_HASH` scrypt hash preferred)
| `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH` | scrypt hash — preferred, no plaintext at rest
| `HERMES_DASHBOARD_BASIC_AUTH_SECRET` | HMAC signing key, >=32 random bytes (`openssl rand -base64 32`). Set it — without it sessions are lost on every restart.
| `HERMES_DASHBOARD_BASIC_AUTH_TTL_SECONDS` | Access token lifetime (default 12h)

## API server / dev

| Variable | Use |
|----------|-----|
| `API_SERVER_ENABLED` | Enable OpenAI-compatible API server
| `API_SERVER_KEY` | Bearer token (required when API server enabled)
| `API_SERVER_PORT` | API server port (default 8642)
| `SUDO_PASSWORD` | Password for sudo without prompt
| `GITHUB_TOKEN` | GitHub (Copilot third priority)
| `CONTEXT7_API_KEY` | Context7 MCP — needed when local MCP server can't reach context7.com but can reach mcp.context7.com

## Terminal — config.yaml owns these (do NOT duplicate in `.env`)

| config.yaml key | `.env-equivalent` (wrong place) |
|---------------|--------------------------|
| `terminal.backend` | (no env var) |
| `terminal.timeout` (180) | `TERMINAL_TIMEOUT` |
| `terminal.lifetime_seconds` | `TERMINAL_LIFETIME_SECONDS` |
| `terminal.modal_image` | `TERMINAL_MODAL_IMAGE` — only for the Modal backend

## When Context7 MCP is broken locally

The bundled Context7 MCP calls `context7.com`, which can fail with HTTP 000 (DNS/timeout in a container). The hosted endpoint `https://mcp.context7.com/mcp` often works and accepts the normal `CONTEXT7_API_KEY` header. Fallback procedure: query the hosted endpoint directly with curl using the same JSON-RPC schema (`resolve-library-id`, `query-docs`). Use this instead of answering from memory — the docs index is the only source of truth.
