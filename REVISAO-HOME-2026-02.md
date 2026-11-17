# Revisão da `~/.hermes` — feita contra o código, não contra a doc

Método: cada chave do `config.yaml` foi conferida contra **onde o código do `hermes-agent` realmente a lê** (`grep` no fonte, `hermes_cli/config_defaults.py`, `agent/`, `tools/`), e o conjunto de skills contra o linter oficial. A doc oficial foi usada só onde o código não dá resposta. Suas próprias skills operacionais (`devops/ninerouter-hermes`, `devops/n8n-self-hosted`) foram tratadas como fonte de verdade da sua intenção — e foram elas que revelaram os drifts mais importantes.

**Caveat de versão:** seu `config.yaml` declara `_config_version: 50`; o checkout do `hermes-agent` aqui analisado (0.20.1, set/2026) usa schema 37. Ou seja: seu runtime instalado é mais novo que este fonte. Onde isso importa, está marcado.

---

## 1. O que o código confirma que está funcionando

| Config | Evidência no código |
|--------|---------------------|
| `model.provider: custom` + `base_url` 9Router + `api_key: ${NINEROUTER_KEY_DEFAULT}` | Formato suportado; `_KNOWN_KEYS` aceita `api`, `transport`, `key_env`, `default_model`, `discover_models` (`hermes_cli/config.py:1356`) |
| Separação `NINEROUTER_KEY_DEFAULT` (principal) vs `NINEROUTER_KEY` (fallback) | **Intencional** — documentado na sua skill `ninerouter-hermes` (evita colisão com `OPENAI_API_KEY`, nome reservado). Falso positivo da primeira rodada, retract aqui |
| `memory.nudge_interval: 10` | Lido em `agent/agent_init.py:1759` |
| `skills.creation_nudge_interval: 15` | Lido em `agent/agent_init.py:1863` (default é 10; seu 15 reduz nudges) |
| `compression.codex_*`, `in_place`, `idle_compact_after_seconds` | São chaves reais de `compression` (não chave perdida) — `config_defaults.py` |
| `mcp_servers`: `idle_timeout_seconds`, `max_lifetime_seconds`, `connect_timeout`, `tools.exclude`, `auth: oauth` | Todos lidos (`tools/mcp_tool.py:26-27`, `hermes_cli/mcp_config.py:304`) |
| `telegram.allowed_chats`, `tools.tool_search.*`, `kanban.review_dispatch`, `approvals.*`, `sessions.*`, `updates.*` | Todos presentes em `config_defaults.py` |
| `max_turns: 500` | É o próprio default (`config_defaults.py:37`) — a chave é redundante mas inofensiva |
| `skills/.archive/` | Excluído do carregamento (`agent/skill_utils.py:32`) — archive não custa prompt |
| `memory.provider: honcho` | Honcho lê `~/.honcho/config.json` + `HONCHO_API_KEY` — config viva fora da `~/.hermes`, nada a fazer aqui |
| `streaming.enabled` (top-level) vs `display.streaming` | Chaves distintas com escopos distintos (gateway/plataformas vs CLI) — não é duplicação |

**Cadeia hermes-workspace:** o workspace espera gateway em `:8642` e dashboard em `:9119` (`src/server/gateway-capabilities.ts:66`), lê `config.yaml` do `HERMES_HOME`. Seu config não tem seção `gateway:` (usa defaults — porta casa) e o `dashboard.basic_auth` via env fecha o par com `HERMES_API_TOKEN`. Integração consistente.

## 2. Chaves mortas ou inertes (peso morto no config)

1. **`memory.flush_min_turns: 6`** — zero leituras em todo o fonte (`grep flush_min_turns` → nada). Chave morta no schema 37; confirme na sua versão 50 antes de remover.
2. **`code_execution.max_tool_calls: 50` / `timeout: 300`** — o código só lê `code_execution.mode` (`tools/code_execution_tool.py:1801`, `EXECUTION_MODES = ("project","strict")`). Os limites reais de execução vêm de `tool_loop_guardrails` e `agent.max_turns`. Mesmo caveat de versão.
3. **`terminal.container_*` + `docker_mount_cwd_to_workspace`** — só usados por backends containerizados (`terminal_tool.py:1736`); com `backend: local` são inertes. Inofensivo — só não se engane achando que limitam algo.
4. **`group_sessions_per_user: true` no nível raiz** — forma legacy; o gateway ainda faz bridge, mas a forma atual é dentro de `gateway:`.

## 3. Drift entre o config e suas próprias skills (o achado principal)

- **Cadeia de fallback incompleta.** Sua skill `ninerouter-hermes` define a cadeia canônica com **3 degraus**: 9Router (`cc/claude-opus-5-5`) → **OpenRouter** (`anthropic/claude-sonnet-4.5`) → Poolside. O `config.yaml` só tem 2 — o OpenRouter saiu. Se a intenção operacional ainda é essa, falta o degrau do meio; se OpenRouter foi aposentado de propósito, a skill está defasada. Um dos dois precisa ser atualizado.
- **Seção `auxiliary:` ausente.** A skill afirma que modelo principal, delegation e os 17 `auxiliary.*` usam `${NINEROUTER_KEY_DEFAULT}`. Sem seção `auxiliary:`, tudo herda do provider principal — o que satisfaz a intenção, desde que a home real não tenha um bloco `auxiliary` que o subset público não reflete. Vale conferir se o `config.yaml` publicado é cópia fiel do da máquina.
- **`skills.disabled` com 3 fantasmas:** `nine-router`, `sasi-cockpit-pipeline`, `sasi-schema-planning` não existem mais (sasi foram para `.archive`). Envelhece o config e confunde auditoria futura.

## 4. Higiene do repositório

1. **`skills/.hub/` rastreado com 6 artefatos gerados** (`lock.json`, `scan-cache/`, `index-cache/`). A regra `skills/.*` do `.gitignore` não alcança arquivos já commitados. `git rm -r --cached skills/.hub` resolve; `taps.json` está vazio (`{"taps": []}`), então nem vale exceção.
2. **README aponta para 3 arquivos que não existem no repo** (`plans/ULTRON-OPERACIONAL.md`, `MAPA-HERMES-HOME.md`, `REPO-ULTRON-HERMES.md` — removidos em `b48346f`). Link quebrado em todo visitante.
3. **Linter oficial de skills: 117 findings em 103 skills** — 49 sem `## When to Use`, 45 sem metadados de frontmatter, 14 citando `grep`/`cat`/`sed` no lugar das tools nativas, 5 referências a arquivos inexistentes (inclui `ninerouter-hermes` citando `references/providers-and-models.md` e `references/configuration.md` que não estão no diretório dela). Fora de `.archive`, os mais relevantes são as refs pendentes da `ninerouter-hermes` — skill crítica da sua operação com documentação interna oca.

## 5. Snapshot de docs embutido no skill `hermes-agent`

`skills/autonomous-ai-agents/hermes-agent/references/` é uma cópia antiga da doc (ex.: `configuration.md` com 89 linhas vs ~2.500 na doc atual). Como o agente usa esse snapshot como referência de si mesmo, ele ensina o Hermes errado sobre o próprio Hermes. Re-sincronizar da doc oficial na próxima manutenção — ou apontar para a URL oficial e deletar o snapshot.

## 6. Resumo executivo

O config está **funcional e bem formado** — nenhuma chave inválida, nenhum segredo exposto, integração com workspace/9Router coerente. Os problemas reais são: (a) duas cadeias de fallback divergentes entre config e skill operacional, (b) ~4 chaves mortas que inflam o config, (c) `skills/.hub/` commitado e links quebrados no repo público, (d) snapshot de docs desatualizado ensinando o agente errado.

Ordem sugerida: drift do fallback (§3) → limpar chaves mortas (§2) → `git rm --cached skills/.hub` + README (§4) → re-sincronizar snapshot (§5).
