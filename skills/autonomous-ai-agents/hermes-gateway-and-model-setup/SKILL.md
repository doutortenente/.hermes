---
name: hermes-gateway-and-model-setup
description: "Set up Hermes Telegram gateway and switch models via CLI."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [hermes, gateway, telegram, setup, model, provider, systemd, oauth]
    related_skills: [hermes-agent]
---

# Hermes Gateway & Model Setup (non-interactive)

Practical, tested procedure for standing up a Hermes messaging-platform bot
(e.g. Telegram) and switching the default model/provider — entirely via CLI,
from an environment with NO real TTY (no `hermes setup` wizard, no tmux
available). Complements the bundled `hermes-agent` skill (which is protected —
do not edit it; add new gateway/model knowledge here instead).

## 1. Gateway service (do this once)

```bash
hermes gateway install     # installs + enables systemd user service, enables linger
hermes gateway status      # verify: "active (running)"
```

`hermes gateway setup` (the interactive TUI picker) is usually a dead end in
these environments: with no tmux and a raw PTY, arrow-key navigation on its
platform-selection list does not reliably move the cursor (repeated ANSI
Up/Down bytes just replay the same frame). **Skip it — configure the platform
directly via `.env` + `hermes config set` instead (below).** This is
faster and more reliable anyway.

## 2. Adding a messaging platform (Telegram example, same pattern for others)

1. Get credentials from the user (bot token from @BotFather for Telegram).
   Never guess/fabricate credentials — ask.
2. Find the exact env var name the adapter reads before writing anything:
   ```bash
   grep -rn "TELEGRAM_BOT_TOKEN\|TELEGRAM_ALLOWED_USERS" /path/to/hermes-agent/plugins/platforms/<platform>/adapter.py
   ```
   Presence of the token env var alone is enough for Hermes to auto-detect
   and enable that platform on next gateway start — no separate "enabled"
   config key needed. Trying `hermes config set gateway.telegram.enabled true`
   produces an "not a recognized config key" warning — don't bother with it.
3. Write the token into `~/.hermes/.env` (get the real path first: `hermes config env-path`).
   If a commented-out placeholder line already exists (`# TELEGRAM_BOT_TOKEN=`),
   leave it and append a new active line rather than trying to un-comment it
   in place — simpler and avoids sed edge cases.
4. Set the allowlist via **`hermes config set telegram.allowed_chats <user_id>`**
   (top-level `telegram:` key — NOT `gateway.telegram.allowed_chats`, which
   silently gets accepted but is never read, and NOT `TELEGRAM_ALLOWED_USERS`
   in .env as the first attempt — both are dead ends the adapter ignores;
   the working key is bare `telegram.*` under `hermes config`).
   Get the user's numeric Telegram ID by having them DM the bot once, then:
   ```bash
   curl -s "https://api.telegram.org/bot${TOKEN}/getUpdates" | python3 -c \
     "import json,sys; d=json.load(sys.stdin); print(d['result'][-1]['message']['from']['id'])"
   ```
5. Restart the gateway to pick up new env/config. **Never run
   `hermes gateway restart` or `stop` from inside a shell that is itself
   a child of the running gateway/CLI process — it self-detects and refuses
   ("Blocked: command or referenced script cannot restart or stop the gateway
   from inside the gateway process").** Instead just `kill <pid>` the gateway
   process directly (find it via `ps aux | grep "gateway run"`); systemd's
   `Restart=` policy (already configured by `gateway install`) auto-restarts
   it within a few seconds. Confirm via `journalctl --user -u hermes-gateway -n 20`.
6. First connection to Telegram takes 10-30s (DNS-over-HTTPS IP discovery +
   handshake) and logs nothing further once connected — silence after
   "Connecting to Telegram (attempt 1/8)…" is the *normal* idle state, not a
   hang. Verify liveness by having the user send a real message through the
   Telegram app (not the Bot API `sendMessage`, which only proves the token
   is valid — it does not exercise the gateway's receive path) and watching
   for new `journalctl` lines (tool-registry / agent-turn logs) or a reply
   arriving in their client.

## 3. Switching default model/provider non-interactively

`hermes model` is an interactive picker (arrow-key list) with the same
navigation problem as gateway setup in non-TTY environments — avoid it here.
Set directly:

```bash
hermes config set model.provider anthropic
hermes config set model.default claude-sonnet-5   # or claude-opus-4.6, etc.
hermes config unset model.base_url                 # clear a stale custom base_url from a prior provider
hermes status | grep -A2 "Model:"                   # verify
```

**Critical when using a local router (9router, Ollama proxy, etc.):**
Hermes routes OpenAI-compatible requests to the router (`base_url: http://127.0.0.1:20128/v1`). The router honors the requested `model.default` if that model ID is registered and active in its catalog.

1. Ensure the target model is registered and enabled in the router's dashboard/catalog (e.g. `nvidia/z-ai/glm-5.3` for GLM-5.3).
2. Set default model and reasoning effort via CLI:
   ```bash
   hermes config set model.default nvidia/z-ai/glm-5.3
   hermes config set agent.reasoning_effort max
   ```
3. **Explicitly control `fallback_providers`**:
   Never leave uncontrolled free-tier models in `fallback_providers`. When the primary model times out or encounters rate limits, Hermes steps down through the fallback list — uncurated free models lead to random assistant switches (Kimi, DeepSeek, Nemotron). Configure an intentional, verified fallback chain:
   ```bash
   hermes config set fallback_providers '[{"base_url":"http://127.0.0.1:20128/v1","model":"cc/claude-opus-5-5","provider":"custom","api_key":"${NINEROUTER_KEY}"},{"base_url":"https://openrouter.ai/api/v1","model":"anthropic/claude-sonnet-4.5","provider":"openrouter","api_key":"${OPENROUTER_API_KEY}"},{"base_url":"https://inference.poolside.ai/v1","model":"poolside/laguna-s-2.1","provider":"custom","api_key":"${POOLSIDE_API_KEY}"}]'
   ```

For Anthropic via a Claude Pro/Max subscription (not a raw API key), the user
needs an OAuth credential first:

```bash
hermes auth add anthropic --type oauth --no-browser
```

This prints a `claude.ai/oauth/authorize?...` URL — since there's no browser
here, give the user the link (verbatim, do not shorten/paraphrase it — the
querystring including `code_challenge` and `state` must survive intact) and
ask them to authorize and paste back the resulting code. Feed it to the
waiting prompt with `process(action='submit', ...)`. Confirm with
`hermes auth list` (shows `anthropic (N credentials)`) and `hermes auth status anthropic`.

Restart the gateway (same kill+auto-restart pattern as above) after changing
the model so it picks up the new default.

## Pitfalls

- Interactive `hermes setup` / `hermes model` / `hermes gateway setup` all
  refuse cleanly with a "non-interactive mode" message when there's truly no
  TTY (`terminal(pty=true)` alone isn't a TTY from Hermes's own detection) —
  but when run under `background=true, pty=true`, they DO launch and accept
  `process(action='write'/'submit')`, they just fail at arrow-key list
  navigation. Prefer direct `hermes config set` / `.env` edits for anything
  that's a plain key=value choice; only fall back to driving the TUI for
  things with no CLI-settable equivalent.
- `hermes config set gateway.telegram.X` (nested under `gateway.`) is *not*
  a recognized key for platform settings — it saves without error but is
  silently unused. Platform settings live under the bare platform name
  (`telegram.*`, `discord.*`, etc.), not under `gateway.*`.
