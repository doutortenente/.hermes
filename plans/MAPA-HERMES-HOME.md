# Mapa de `~/.hermes` (Tijolão)

Fonte: doc Hermes (SOUL em `~/.hermes/SOUL.md` só) + `skills/devops/hermes-runtime-maintenance/references/paths-official-classification.md`.

**~45 arquivos na raiz é normal.** Quase tudo ali é caminho fixo do agente — **mover quebra** gateway, MCP ou SQLite.

**`installs/`:** não reorganizar nem apagar sem plano explícito (ambiente do container).

---

## Raiz — arquivo por arquivo

| Arquivo | Mexer? | Função |
|---------|--------|--------|
| `config.yaml`, `SOUL.md`, `.gitignore`, `README.md` | Não | Config, identidade, doc mínima do repo |
| `.env` → cofre | Não | Segredos (única fonte de verdade) |
| `auth.json`, `auth.lock` | Não | OAuth Hermes |
| `state.db`, `kanban.db`, outros `*.db` (+ shm/wal/locks) | Não | Bancos runtime |
| `gateway.pid`, `gateway.lock`, `gateway.sock`, `gateway_state.json` | Não | Gateway |
| `channel_directory.json`, `spawn-ledger.json`, `tasks.json` | Não | Roteamento / filas |
| `telegram_update_receipts_*.json` | Não | Estado Telegram |
| `models_dev_cache.json`, `provider_models_cache.json`, `context_length_cache.yaml`, … | Não | Catálogo / cache de modelos |
| `.skills_prompt_snapshot.json`, `shell-hooks-allowlist.json` | Não | Skills / hooks |
| `tui-theme-boot.json`, `web-ui-build-stamp.json`, `.update_check`, `install_id` | Não | UI / updates |
| Locks (`.mcp-discovery.lock`, `.env.lock`, …) | Não | Processo vivo |

Logs oficiais ficam em **`logs/`** (não na raiz).

---

## `state.db` vs pasta `state/` vs `gateway_state.json` (confusão real)

Não é bagunça só tua — o **upstream Nous** usa nomes sobrepostos:

| Nome | Onde | O quê (fonte: código Hermes 0.22.x no teu `installs/…/workspace/`) |
|------|------|---------------------------------------------------------------------|
| **`state.db`** | **Raiz** | SQLite grande — sessões e mensagens. Doc: `get_hermes_home() / "state.db"` (session-storage.md). |
| **`gateway_state.json`** | **Raiz** | JSON que o CLI/dashboard lê (`hermes gateway status`, Docker autostart). **Não** fica dentro de `state/`. |
| **Pasta `state/`** | Subdir | Ficheiros **mais novos** de supervisão: `gateway.heartbeat` (30s), `gateway.lifecycle.json` (morte limpa/suja), `dashboard_clients.heartbeat`, etc. |

Ou seja: **dois “estados” oficiais** — legado na raiz (`state.db`, `gateway_state.json`) + subpasta `state/` para heartbeats/sentinela. **Mover um para dentro do outro quebra** — não é reorganização, é patch no Hermes.

O que **não** era oficial e já saiu: `.local/state/hermes/` (nest vazio), `.hermes/.hermes/` se existisse.

---

## Onde colocar **seus** arquivos

| Tipo | Pasta correta | Errado (exemplos corrigidos) |
|------|---------------|------------------------------|
| Planos, auditorias, inventários | `plans/` (ativos), `plans/reference/` (inventários longos), `plans/archive/` (histórico) | `cache/auditoria-tijolao/` (movido), `skills/sasi_v3_inventory.md` → `plans/reference/sasi-v3-inventory.md` |
| Memória longa do agente | `memories/` (`USER.md`, `MEMORY.md`) | `pastes/`, `plans/` |
| Código / repos | `~/src` (host) | `~/.hermes/workspace/` (só mesa + JSON da UI workspace) |
| Skills tuas | `skills/devops/` (Tijolão + SASI + harness-insights), `skills/productivity/` (ex.: mestre-airtable, i-have-adhd) | `software-development/sasi-*` → **`skills/devops/sasi-*`**; `mestre-zed` → **`skills/devops/mestre-zed`** |
| Backups de config | `backups/config/` (2 `.good` recentes; resto em `backups/config/archive/`) | Raiz, `cache/` |
| Clipboard do agente | `pastes/` — **apagar** dumps de `.env`/senhas na hora | Nunca versionar |

**`workspace/`** — deixar: `knowledge-config.json`, `mcp-presets.json`, `mcp-hub-sources.json`, `cache/mcp-tools.json` (hermes-workspace lê daqui).

### Layout `plans/` (29-out)

| Caminho | Conteúdo |
|---------|----------|
| `plans/MAPA-HERMES-HOME.md` | Este índice |
| `plans/ULTRON.md` | Norte do projeto ULTRON |
| `plans/ULTRON-OPERACIONAL.md` | Guia operacional (caminhos, saúde, fila config) |
| `plans/REPO-ULTRON-HERMES.md` | Repo público |
| `plans/reference/` | Inventários e auditorias (SASI v3, MCP-AUDIT, …) |
| `plans/archive/` | Planos executados, relatórios, auditoria Tijolão |

**`kanban/attachments/`** — anexos de tarefa; cópia de relatório também em `plans/archive/` quando for documento teu.

---

## Pastas runtime (não reorganizar)

`sessions/`, `logs/`, `cache/` (só cache regenerável — **não** docs), `state/`, `cron/`, `runtime/`, `pending*`, `gateway/`, `hooks/`, `platforms/`, `home/` (HOME falso — **nunca apagar**), `bin/`, `tools/`, `plugins/`, `skills/` (bundled + tuas), `mcp-tokens/`, `vault/`, `shared/`, `kanban/`, `webui-mvp/`, `installs/`, caches (`audio_cache/`, `image_cache/`, …), `backups/`, `.curator_backups/`.

Não existe pasta `scripts/` no momento (Hermes recria se precisar). `profiles/` vazio é ok.

---

## Correções feitas (29-out-2026)

| Problema | Ação |
|----------|------|
| Doc SASI em `skills/` | → `plans/reference/sasi-v3-inventory.md` |
| Planos históricos soltos em `plans/` | → `plans/archive/` (2ª passagem: MCP-AUDIT → `reference/`) |
| Auditoria Tijolão em `cache/` | → `plans/archive/auditoria-tijolao/` |
| `.local/state/hermes/` morto | Removido |
| Paste com **senhas/tokens** em `pastes/` | **Apagado** — usar só cofre `.env` |
| Nota npm container em paste | → `plans/archive/container-npm-audit-2026.txt` |
| `mestre-zed` em `development/` | → `skills/devops/mestre-zed` |
| SASI + harness em `software-development/` | → `skills/devops/sasi-cockpit-pipeline`, `sasi-schema-planning`, `harness-insights` |
| `i-have-adhd` solto em `skills/` | → `skills/productivity/i-have-adhd` |
| 5 backups `config.yaml.good` no mesmo dia | 2 ativos + resto em `backups/config/archive/` |

**2ª passagem (29-out, tarde):** `plugins/hermes-honcho-plugin/docs/`, `kanban/attachments/`, `skills/*` por categoria, `workspace/*.json`, `state/*`, raiz runtime — **lugar correto; sem movimentação.**

**3ª passagem (29-out, noite):** skills custom **Nícholas/SASI** estavam em `software-development/` → movidas para `skills/devops/` (`sasi-cockpit-pipeline`, `sasi-schema-planning`, `harness-insights`). Restante de `~/.hermes` varrido (cache UV/npm, `home/` fake HOME, `logs/curator/REPORT.md` = runtime Hermes) — **sem mais realocação**.

### Veredicto por pasta (fim da auditoria)

| Pasta | Veredicto |
|-------|-----------|
| `plans/` | Organizado (ativos / reference / archive) |
| `skills/devops/` | **Todas** skills custom Tijolão + SASI + harness |
| `skills/productivity/` | Custom: mestre-airtable, i-have-adhd (+ bundled) |
| `skills/*` (outras categorias) | Bundled Hub — não mover |
| `memories/`, `SOUL.md`, `config.yaml` | Identidade — raiz ou memories |
| `workspace/` | 4 JSON da UI — não misturar com `~/src` |
| `kanban/`, `plugins/`, `tools/`, `bin/` | Runtime / anexos — ok |
| `cache/`, `logs/`, `sessions/`, `state/`, `installs/`, `home/` | Runtime Nous — **não reorganizar** |
| `backups/`, `pastes/`, `assets/`, `pets/` | Backups locais / vazio / UI — ok |
| Raiz `*.db`, gateway, caches JSON | Caminhos fixos upstream — **não mover** |

---

## Git / repo público

https://github.com/doutortenente/ultron-hermes — subset versionável; **`memories/` e runtime fora do Git**. Detalhes: `plans/REPO-ULTRON-HERMES.md`.

---

## Saúde

```bash
hermes doctor
curl -fsS http://127.0.0.1:8642/health
tijolao health
```

Manutenção pesada: skill `hermes-runtime-maintenance` (sem tocar em `installs/` até você mandar).
