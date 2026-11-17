# Revisão da home Hermes (`~/.hermes`) — 2026-02

Auditoria do repo público `.hermes` contra três fontes de verdade:

1. **`hermes-agent`** (código-fonte): `hermes_cli/config_defaults.py`, `hermes_cli/config.py`, `agent/skill_utils.py`, `tools/skill_linter.py`, `cli-config.yaml.example`
2. **Docs oficiais** (`website/docs/`): `user-guide/which-file-does-what.md`, `user-guide/profile-distributions.md`, `user-guide/profiles.md`, `user-guide/managed-scope.md`, `reference/mcp-config-reference.md`
3. **`hermes-workspace`**: contrato de nomes (`docs/hermes-workspace-naming-contract.md`)

Nenhum arquivo de config/skill foi alterado. Este relatório é o único artefato da branch.

---

## 1. O que está correto (alinhado com a referência oficial)

| Item | Evidência |
|------|-----------|
| Estrutura raiz `SOUL.md` + `config.yaml` + `skills/` + `README.md` | Bate com o modelo de distribuição oficial (`profile-distributions.md`, seção "What the repo looks like") |
| `SOUL.md` na raiz da home, nunca no working dir | `which-file-does-what.md`: identidade do agente vive em `~/.hermes/SOUL.md`, slot #1 do system prompt |
| `.gitignore` espelha o template oficial de profile distribution | Credenciais (`auth.json`, `.env`, `*.pem`, `.ssh/`), dados do usuário (`memories/`, `sessions/`, `logs/`, `plans/`), caches, checkpoints e backups — todos cobertos, e o arquivo vai além do template mínimo |
| Zero segredos commitados | `config.yaml` usa somente `${ENV_VAR}` (interpolação suportada por `hermes_cli/config.py` → `_expand_env_vars`); scan de 558 arquivos rastreados não achou chave/token real |
| `security.redact_secrets: true` | Recomendação oficial de segurança |
| `dashboard.basic_auth` via env refs (`${DASHBOARD_USERNAME}` etc.) | Correto — nada de senha em texto plano |
| Todas as 38 chaves raiz do `config.yaml` são válidas | Validadas contra `DEFAULT_CONFIG` + `_EXTRA_KNOWN_ROOT_KEYS` do `hermes_cli/config.py` — nenhuma chave órfã ou renomeada |
| Defaults oficiais respeitados | `max_turns: 500` (= default em `config_defaults.py:37`), `database.journal_mode: wal`, `runtime.nofile_soft_limit: 4096`, `prompt_caching.cache_ttl: 5m` |
| `mcp_servers` no `config.yaml` com `auth: oauth` | Formato oficial (`mcp-config-reference.md`, seção OAuth) |
| `skills/.archive/` fora do prompt | `agent/skill_utils.py:32` exclui `.archive` do carregamento — archive não vaza para o agente |
| `terminal.cwd` aponta para `~/.hermes/home` (não para a raiz da home) | Consistente com a orientação do README de não tratar a raiz como projeto Git |

## 2. Problemas encontrados (ordenados por prioridade)

### P1 — Artefatos gerados rastreados no Git: `skills/.hub/`
6 arquivos commitados (`lock.json`, `taps.json`, `index-cache/`, `scan-cache/*.json`) são cache/estado gerado pelo Skills Hub. A regra `skills/.*` do `.gitignore` deveria excluí-los, mas entraram antes da regra e ficaram rastreados (gitignore não afeta arquivos já trackeados).

**Correção:** `git rm -r --cached skills/.hub && git commit`. `taps.json` é o único com conteúdo autoral (taps registrados) — se quiser versioná-lo, adicione exceção `!skills/.hub/taps.json`.

### P2 — README com 3 links quebrados
`README.md` aponta para `plans/ULTRON-OPERACIONAL.md`, `plans/MAPA-HERMES-HOME.md` e `plans/REPO-ULTRON-HERMES.md` — o diretório `plans/` é gitignored e não existe no repo (removido no commit `b48346f`).

**Correção:** remover os links ou movê-los para o repo privado onde os plans vivem.

### P3 — Entradas órfãs em `skills.disabled`
`nine-router`, `sasi-cockpit-pipeline`, `sasi-schema-planning` não existem mais (as sasi foram para `.archive`; `nine-router` não existe nem no bundle oficial). Inofensivo em runtime, mas envelhece o config. Nota: a skill ativa `ninerouter-hermes` (em `skills/devops/`) é nome distinto — não é coberta pela entrada órfã.

**Correção:** remover as 3 entradas.

### P4 — Linter oficial de skills: 117 findings em 103 skills
Rodado com `tools/skill_linter.py` do próprio hermes-agent:

| Finding | Qtd | Observação |
|---------|-----|------------|
| `missing-section` (sem `## When to Use`) | 49 | Skills sem gatilho explícito perdem roteamento |
| `missing-metadata` (version/author/license/tags) | 45 | Concentradas em `.archive` e skills importadas |
| `shell-utility-reference` (prosa cita `grep`/`cat`/`sed` em vez das tools nativas) | 14 | Inclui `hermes-agent-skill-authoring` |
| `dangling-reference` (arquivo citado não existe) | 5 | `ninerouter-hermes` cita `references/providers-and-models.md` e `references/configuration.md` inexistentes; sasi (arquivadas) idem |
| `forbidden-file` (SKILL.md + README.md) | 3 | `ascii-video`, `manim-video`, `p5js` |
| `description-length` | 1 | `i-have-adhd`: descrição de 242 chars; o índice trunca em 60 |

**Correção:** priorizar as skills ativas (fora de `.archive`); `.archive` pode ficar como está.

### P5 — Superfície de recon em repo público
`config.yaml` expõe `https://n8n.nagaitaltda.com/mcp-server/http` (seu n8n self-hosted). Não há token no arquivo (auth é OAuth), mas o hostname revela infraestrutura privada. O domínio `nagaitaltda.com` já aparece em skills (`n8n-self-hosted` etc.) — avalie se é aceitável ou vale mover para env ref.

### P6 — Config referencia plugins que não estão no repo
`plugins.enabled` lista `hades-coding-workflows`, `hermes-honcho-plugin`, `office-desk` — vivem em `~/.hermes/plugins/` (gitignored, correto) e não existem no hermes-agent. Runtime funciona; mas o repo público sozinho não reproduz a home. Vale uma linha no README dizendo que plugins são locais.

### P7 — Inconsistência de nomes de env var do 9Router
`model.api_key: ${NINEROUTER_KEY_DEFAULT}` vs `providers.nine-router.key_env: NINEROUTER_KEY`. Se ambos existem no `.env`, ok — mas confirme; se um deles não existir, o fallback silencioso quebra a cadeia de providers.

### P8 — Caminho absoluto hardcoded
`mcp_servers.context7.command: /home/dr/.hermes/home/.local/bin/context7-mcp` — quebra em outra máquina, usuário ou perfil. Alternativa: nome no `PATH` ou env ref.

### P9 — Snapshot de docs do skill `hermes-agent` desatualizado
`skills/autonomous-ai-agents/hermes-agent/references/configuration.md` tem 89 linhas; a doc oficial equivalente tem ~2.500. Os snapshots servem ao agente como referência — vale re-sincronizar da doc oficial na próxima passada de manutenção.

## 3. Verificações executadas

```bash
# Chaves raiz do config vs schema oficial
python3 (yaml.safe_load) + diff contra DEFAULT_CONFIG/_EXTRA_KNOWN_ROOT_KEYS  → OK

# Scan de segredos em 558 arquivos rastreados
grep -riE "(api_key|token|secret|password)\s*[:=]\s*[A-Za-z0-9_-]{16,}"       → só docs de exemplo, OK

# Linter oficial de skills (hermes-agent)
PYTHONPATH=. python3 tools/skill_linter.py ~/.hermes/skills                   → 117 findings

# Consistência .gitignore
git ls-files skills/.hub                                                      → 6 artefatos rastreados
git check-ignore skills/.hub/lock.json                                        → rc=1 (regra não pega arquivo trackeado)
```

## 4. Ordem sugerida de execução

1. `git rm -r --cached skills/.hub` (P1)
2. Corrigir links do README (P2)
3. Limpar `skills.disabled` órfão (P3) + env vars do 9Router (P7)
4. Passada do skill linter nas skills ativas (P4)
5. Decisão sobre o hostname do n8n (P5) e nota sobre plugins locais no README (P6)
