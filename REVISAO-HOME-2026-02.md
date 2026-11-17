# Revisão da `~/.hermes` — referências: hermes-agent, hermes-workspace e docs oficiais

Fontes usadas, nesta ordem de autoridade:

1. **Código do `hermes-agent`** — o que executa de fato: `hermes_cli/config.py`, `hermes_cli/config_defaults.py`, `agent/skill_utils.py`, `tools/mcp_tool.py`, `tools/skill_linter.py`
2. **Documentação oficial** — `hermes-agent/website/docs/` (fonte do site hermes-agent.nousresearch.com): `user-guide/profile-distributions.md`, `user-guide/which-file-does-what.md`, `user-guide/features/personality.md`, `developer-guide/creating-skills.md`, `reference/mcp-config-reference.md`
3. **Código do `hermes-workspace`** — o que a UI espera da home: `src/routes/api/models.ts`, `src/routes/api/connection-status.ts`, `src/server/gateway-capabilities.ts`

**Nenhuma skill agent-authored foi usada como referência.**

---

## 1. Validação executada

| Check | Resultado |
|-------|-----------|
| `validate_config_structure()` (validador oficial, `hermes_cli/config.py`) contra o `config.yaml` | **0 issues** |
| Chaves raiz vs `DEFAULT_CONFIG` + `_EXTRA_KNOWN_ROOT_KEYS` | Todas válidas |
| Campos de `providers.*` vs `_KNOWN_KEYS` (`config.py:1356`) | Todos válidos |
| Chaves de `mcp_servers.*` vs `mcp-config-reference.md` + `tools/mcp_tool.py:26-27` | Todas válidas (`idle_timeout_seconds`/`max_lifetime_seconds` são do stdio recycle) |
| Schema version | Config declara `_config_version: 50`; checkout do hermes-agent (0.20.1) migra até 37 — runtime instalado é mais novo que o fonte auditado. Implica: chaves "mortas" no fonte podem ter leitura na versão nova |

## 2. Achado principal — integração hermes-workspace (cross-repo)

O picker de modelos do hermes-workspace (`src/routes/api/models.ts:233-241`) lê de cada `providers.*` apenas:

- endpoint: `base_url` / `baseUrl` / `api_base` / `apiBase`
- credencial: `api_key` / `apiKey` / `token` / `api_key_env` (com expansão `${VAR}` em `resolveConfiguredSecret`, `models.ts:191-197`)

O `config.yaml` usava `api:` e `key_env:` — aceitos pelo hermes-agent (`config.py:1390` aceita `base_url`/`url`/`api`), mas **invisíveis para o workspace**. Efeito: só o endpoint do bloco `model:` (9Router) participava da descoberta live de modelos; Poolside ficava só com a lista estática.

**Correção aplicada:** `providers.*` agora usa `base_url:` + `api_key: ${VAR}` — equivalente em runtime para o hermes-agent, e os dois endpoints passam a alimentar o picker do workspace.

## 3. Config — o que está correto (código + docs oficiais)

- **Modelo principal via 9Router local** (`provider: custom` + `base_url` localhost) — forma oficial para endpoint OpenAI-compatible (`cli-config.yaml.example`).
- **`fallback_providers`** no topo — formato atual (`hermes_cli/fallback_cmd.py:15`); credenciais por entrada via `api_key`/`key_env` (`fallback_config.py:14-35`).
- **`compression.codex_*`, `in_place`, `idle_compact_after_seconds`** — chaves reais da seção (`config_defaults.py`), não chaves perdidas.
- **`memory.nudge_interval`** lido em `agent/agent_init.py:1759`; **`skills.creation_nudge_interval`** em `agent_init.py:1863` (default 10).
- **`max_turns: 500`** = próprio default (`config_defaults.py:37`).
- **`mcp_servers`**: `auth: oauth` para HTTP é o fluxo oficial (OAuth 2.1 PKCE, tokens em `~/.hermes/mcp-tokens/` — gitignored, correto); `tools.exclude` com wildcards é suportado.
- **`telegram.allowed_chats`**, `tools.tool_search.*`, `kanban.review_dispatch`, `approvals.*`, `sessions.*`, `updates.*` — todos em `config_defaults.py`.
- **`security.redact_secrets: true`** e dashboard basic_auth via env refs.

## 4. Config — peso morto (não removido; ver caveat de versão)

1. `memory.flush_min_turns` — zero leituras no fonte (schema 37).
2. `code_execution.max_tool_calls` / `timeout` — o código só lê `code_execution.mode` (`tools/code_execution_tool.py:1801`); limites reais vêm de `tool_loop_guardrails`/`agent.max_turns`.
3. `terminal.container_*` — inertes com `backend: local` (usados só por backends containerizados, `terminal_tool.py:1736`).
4. `group_sessions_per_user` no nível raiz — forma legacy (bridge do gateway); forma atual é dentro de `gateway:`.

## 5. SOUL.md (doc oficial: `personality.md` + `which-file-does-what.md`)

- Localização correta: raiz da `HERMES_HOME`, slot #1 do prompt, nunca no working dir.
- Conteúdo dentro da guidance oficial ("durable voice and personality guidance… not task-specific instructions"): tom, protocolo de resposta, postura, formato. Seção 9 (ecossistema ULTRON/n8n) é identidade, não workflow — dentro do escopo.
- Nada a corrigir.

## 6. skills/ (doc oficial: `creating-skills.md` + loader `agent/skill_utils.py`)

- **Estrutura de categorias** confere com o padrão oficial (`skills/<categoria>/<skill>/SKILL.md`).
- **`skills/.archive/`** — excluído do carregamento (`skill_utils.py:32`); não vaza para o prompt.
- **`skills/.hub/`** — artefato gerado (lock, scan-cache, index-cache, taps vazio); estava **rastreado** no Git. O template oficial de profile distribution manda ignorar caches/artefatos gerados. Correção aplicada: desrastreado + regra explícita no `.gitignore`.
- **Frontmatter**: formato exigido é `name` + `description` (obrigatórios) + `version`/`author`/`license`/`platforms`/`metadata.hermes.*` (recomendados — `creating-skills.md:46-60`). Linter oficial (`tools/skill_linter.py`): 117 findings em 103 skills — 49 sem seção `## When to Use`, 45 sem metadados recomendados, 5 referências a arquivos inexistentes. Nenhum erro bloqueante; são melhorias de roteamento, não quebras.
- **`skills.disabled`**: 3 nomes que não existem nem no bundle oficial nem no repo (`nine-router`, `sasi-cockpit-pipeline`, `sasi-schema-planning`) — removidos.

## 7. README e .gitignore (doc oficial: `profile-distributions.md`)

- Template oficial de distribuição: conteúdo autoral versionável = `SOUL.md`, `config.yaml`, `skills/`, `cron/jobs.json`, `README.md`; tudo o mais (credenciais, memórias, sessões, caches, checkpoints) ignorado.
- O `.gitignore` cobre o template completo (e vai além). **README** apontava para `plans/*.md` removidos do repo — reescrito sem links quebrados, descrevendo o modelo de subset.
- **`.env.example`** criado com as 10 variáveis referenciadas pelo `config.yaml` (o template oficial prevê `.env.EXAMPLE` na distribuição).

## 8. Correções aplicadas nesta branch

1. `providers.*`: `api:` → `base_url:`, `key_env:` → `api_key: ${VAR}` (integração hermes-workspace; runtime hermes-agent equivalente)
2. `skills.disabled`: 3 entradas órfãs removidas
3. `skills/.hub/`: desrastreado + regra explícita no `.gitignore`
4. `README.md`: reescrito sem links quebrados
5. `.env.example`: novo

## 9. Pendente de decisão (não aplicado)

- **Peso morto do §4**: confirmar na versão instalada (schema 50) antes de remover — `hermes doctor` na máquina.
- **Cadeia de fallback**: config tem 2 degraus (9Router → Poolside). Se quiser um degrau OpenRouter no meio, é `hermes config set fallback_providers '[...]'` — decisão operacional sua.
