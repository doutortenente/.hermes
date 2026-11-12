---
name: hermes-secrets-management
description: 'Load Hermes secrets from split files or vaults, not .env.'
version: 0.1.0
author: hermes-agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [hermes, secrets, .env, config.yaml, security, bitwarden, onepassword, command-source]
    homepage: https://hermes-agent.nousresearch.com/docs/
    related_skills: [hermes-agent]
---

# Hermes Secrets Management (beyond a single .env)

## When to use this skill

Use when you need to load API keys, tokens, and other secrets into Hermes **without** putting them in a flat plaintext block in `~/.hermes/.env`, or when migrating from a monolithic `.env` to a split/vault-backed source.

## Hard rule

- **Secrets in `.env`, settings in `config.yaml`** — settings never belong in `.env`; secrets never belong in `config.yaml` as plaintext values.

## The one rule that decides everything

`hermes doctor` only reads the literal `~/.hermes/.env` file. It will report "No API key found" / ".env file missing" even when every key is loaded successfully via a command/bulk source. **Treat doctor's secret warning as a false alarm when `secrets.command` (or bitwarden/1Password) is enabled and verified.** Always verify with a real import test, not by staring at `doctor`.

## Procedure: replace a monolithic .env with a split-file command source

1. **Consolidate the parts in one place.** The Hermes runtime lives in the agent container (`/opt/hermes`) or `~/.hermes/home` — use whichever home directory is writable and is where the command runs. Put the split files there (e.g. `~/.local/secrets/[I-IV].env.*.txt`).
2. **Point a command helper at them** via the CLI (never hand-edit `config.yaml`):

   ```bash
   hermes config set secrets.command.enabled true
   hermes config set secrets.command.command \
     "cat /home/dr/.local/secrets/I.env.*.txt /home/dr/.local/secrets/II.env.*.txt \
      /home/dr/.local/secrets/III.env.*.txt /home/dr/.local/secrets/IV.env.*.txt"
   ```

   Resulting section:
   ```yaml
   secrets:
     command:
       enabled: true
       command: cat ...
   ```
3. **Move the old monolithic `.env` aside** (don''t delete on the first run — rename it so you can restore if a key was missing). Example: `mv ~/.hermes/.env ~/.hermes/.env.gerado-quarentena`.
4. **Leave a placeholder `.env`** that contains only a comment so the runtime/path check passes:

   ```
   # Secrets load from ~/.local/secrets via secrets.command in config.yaml
   ```

   Set `chmod 600 ~/.hermes/.env`.
5. **Start a fresh, zeroed environment** and import the loader to prove the source works in isolation:

   ```bash
   env -i PATH=/usr/bin:/bin HOME=/home/dr/.hermes/home \
     /opt/hermes/.venv/bin/python -c "
   from pathlib import Path; from hermes_cli.env_loader import load_hermes_dotenv
   load_hermes_dotenv(hermes_home=Path('/home/dr/.hermes')); import os; print('NVIDIA_API_KEY', bool(os.environ.get('NVIDIA_API_KEY')))"
   ```

   Expected line: `Command helper: applied N secrets`, and `True` per key.

## Pitfalls

- **Doctor lie.** `hermes doctor` flags a missing `.env` or missing key even when `secrets.command` loaded everything. This is by design — doctor is a static file checker. The verification command in step 5 is the source of truth.
- **Glob order matters.** The shell expands the globs in lexical order; if a later file redefines a key that appears in an earlier one, the duplicate is silently skipped (first-claim wins). Check `Command helper: applied N secrets` against your expected count — a lower number means a collision was dropped.
- **Don''t regenerate.** A previous mistake was generating a new `.env` from the parts and leaving it around; that reintroduces the single-file anti-pattern you are trying to escape. The parts are the source of truth.
- **Bulk command source runs once at startup** — it is not re-evaluated mid-session. To pick up edits to a part file, restart the agent (or `hermes gateway restart`).

## Other sources

- **Bitwarden / 1Password** — bundled sources (`agent/secret_sources/bitwarden.py`, `onepassword.py`); enable under `secrets.sources` / `secrets.bitwarden`.
- **Command source** is POSIX-only — needs `/bin/sh`. On Windows it resolves no value (falls back to `.env`/env).

## Verification commands

- Zero-env import test — step 5 above.
- `hermes doctor | grep -E '.env|API key|issue'` — informational only.
- A real chat turn: `hermes chat -q 'responda apenas: ok'` and confirm it returns.

Full upstream docs: https://hermes-agent.nousresearch.com/docs/user-guide/configuration
