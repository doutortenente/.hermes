---
name: n8n-self-hosted
description: Use when configuring or connecting n8n on Tijolão.
---

# n8n self-hosted (Tijolão)

Container `ultron-n8n` (2.x line), compose `/home/dr/.config/hermes-stack/docker-compose.tijolao.yml`, data `/home/dr/.local/state/n8n` (mounted at `/home/node/.n8n`; `config` holds encryptionKey). Public address = Cloudflare tunnel `https://n8n.nagaitaltda.com` -> 127.0.0.1:5678. Edit host files via SSH (skill tijolao-ssh-bridge).

## Always-on rules (user standard)
1. Research BEFORE touching config: Context7 `/n8n-io/n8n-docs` + docs.n8n.io, then confirm in the running version's code (`docker exec ultron-n8n grep ... /usr/local/lib/node_modules/n8n/dist/...` and `.../node_modules/@n8n/config/dist/configs/*.js`, where `Env('NAME')` defines every official variable). Never ask the user a question the docs answer.
2. Never invent a variable name, route or file. If no official name exists (e.g. MCP token has none), prefer the mechanism that needs no name (OAuth) instead of coining one.
3. Fix root cause, not symptom: if URLs come out wrong, fix the instance base URL (`N8N_EDITOR_BASE_URL`, `N8N_WEBHOOK_URL`) — do not add a per-feature override (`N8N_MCP_BASE_URL`) as a patch. Override only when MCP must live on a different host than the instance.
4. Finish completely: do not report a 'pendência' you can fix yourself; fix it, then report.
5. Never write to the DB by hand (forging JWTs, inserting settings). Use official env vars or the same REST routes the UI calls, with an owner session cookie.
6. Leave no residue: no loose key files, no backup copies, no '# antigo' comment lines in the vault; delete temp files in the same command.

## Where things live
| Thing | Place |
|---|---|
| Instance config (owner, MCP, URLs) | compose `environment:` of service n8n |
| `N8N_URL`, `N8N_API_KEY` (official names from n8n CLI docs) | vault `IV.env.infra.txt` section `003 - N8N` AND `/home/dr/.local/secrets/.env` (same line both; Airtable pattern) |
| Hermes -> n8n MCP | `config.yaml` `mcp_servers.n8n` {url: https://n8n.nagaitaltda.com/mcp-server/http, auth: oauth}; token in `~/.hermes/mcp-tokens/n8n*.json` |

Official env vars used: `N8N_INSTANCE_OWNER_MANAGED_BY_ENV/EMAIL/FIRST_NAME/LAST_NAME/PASSWORD_HASH` (bcrypt, `$` escaped as `$$` in compose), `N8N_MCP_MANAGED_BY_ENV`, `N8N_MCP_ACCESS_ENABLED`, `N8N_EDITOR_BASE_URL`, `N8N_WEBHOOK_URL` (`WEBHOOK_URL` is deprecated — the startup log warns). The Tailscale :8443 address must not appear in the compose.

## Procedures
- Apply compose change: `docker compose -f docker-compose.tijolao.yml --env-file /home/dr/.local/secrets/.env up -d --no-deps --force-recreate n8n`, wait `/healthz` 200, check log line `Editor is now accessible via:`.
- Upgrade: change image tag in compose, recreate as above; DB migrates on start.
- Delete a workflow in 2.x: deactivate -> archive -> `DELETE /rest/workflows/{id}` (published workflow gives 409 otherwise).
- Public API key: `POST /rest/api-keys` with owner cookie, body `{label, scopes, expiresAt:null}`; scopes = `OWNER_API_KEY_SCOPES` from `@n8n/permissions` (run node with `NODE_PATH=/usr/local/lib/node_modules/n8n/node_modules`). Outside Enterprise the key has full access anyway; do not quote a scope count you did not measure.
- Hermes MCP via OAuth (headless): see `references/mcp-oauth-headless.md`.
- Claude Code: `claude mcp add --transport http n8n https://n8n.nagaitaltda.com/mcp-server/http` then `/mcp` -> approve (OAuth).

## Pitfalls
- Changing the instance base URL invalidates the existing Hermes OAuth token (resource/audience changes): re-run `hermes mcp login n8n` and verify with `hermes mcp test n8n`.
- In background/pty shells `hermes` may not be on PATH: use `/home/dr/.hermes/home/.local/bin/hermes`.
- grep/find in the container do not follow pnpm symlinks; grep the concrete `dist/` dirs.
- The MCP URL opened in a browser returns `Unauthorized: Authorization header not sent` — expected, not a failure.
- Use the owner credentials the user gave; never generate a password for his account.
- Delivering secrets: the chat redactor truncates long keys; write them to a file in scratch and send via MEDIA:, and say the scratch file is pruned in 24h.
