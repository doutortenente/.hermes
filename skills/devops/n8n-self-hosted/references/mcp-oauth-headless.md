# Hermes -> n8n MCP OAuth login without a browser

Verified working on n8n 2.46.6 + Hermes 0.22.2.

1. `config.yaml`: `hermes config set mcp_servers.n8n.url https://n8n.nagaitaltda.com/mcp-server/http` and `hermes config set mcp_servers.n8n.auth oauth`.
2. Start login in background with pty: `/home/dr/.hermes/home/.local/bin/hermes mcp login n8n`. It prints an authorize URL and listens on `http://127.0.0.1:<port>/callback`. If the URL host is not n8n.nagaitaltda.com, the instance base URL is wrong — fix `N8N_EDITOR_BASE_URL` first.
3. On the host (same machine as the callback listener), with a cookie jar:
   - `POST $B/rest/login` `{"emailOrLdapLoginId":..., "password":...}` -> 200
   - `GET <authorize URL>` -> 302 `Location: /oauth/consent?flow=<id>`
   - `GET $B/rest/consent/details?flow=<id>` -> `data.scopes`
   - `POST $B/rest/consent/approve` `{"approved":true,"flow":"<id>","scopes":<data.scopes>}` (scopes mandatory: empty -> `At least one scope must be granted`) -> `data.redirectUrl`
   - `GET <redirectUrl>` -> `Authorization Successful`
   (routes from `dist/modules/oauth-server/oauth-consent.controller.js`, `RestController('/consent')`).
4. Login prints `Authenticated — N tool(s) available`. Confirm with `hermes mcp test n8n` (Auth: OAuth 2.0, Connected, Tools discovered). Tools appear as `mcp_n8n_*` in a NEW session.
5. Delete cookie jar and temp json in the same command.
