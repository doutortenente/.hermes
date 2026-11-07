# ULTRON — guia operacional (início → fim)

Documento único: **o que é**, **onde fica no disco**, **estado medido**, **como provar saúde**, **melhorias de config aplicadas**, **fila restante**. Atualizado: 01-nov-2026.

---

## 1. Vocabulário (sem confusão)

| Termo | Significado | **Não é** |
|-------|-------------|-----------|
| **Tijolão** | Apelido desta máquina (Mint, host onde Hermes roda) | Pasta no disco |
| **`~/.hermes`** | Casa do agente (`HERMES_HOME`): config, DB, memória, planos | Repo Docker |
| **`~/.config/hermes-stack`** | Receita Docker (compose, n8n workflows, patch da imagem) | O agente em si |
| **`tijolao`** | Comando em `~/.local/bin/tijolao` que sobe a stack | Diretório |
| **ULTRON** | Nome do ecossistema (Hermes + stack + n8n); ver `SOUL.md` §9 | Personalidade separada do Hermes |

**Ligação:** o compose monta `~/.hermes` nos containers → um único `config.yaml` e os mesmos `.db` no host e no Docker.

**Cofre:** `~/.local/secrets/.env` (symlink `~/.hermes/.env`).

**Código UI:** `~/src/hermes-workspace` (branch `tijolao`). Caminho morto: `~/src/tijolao-ai/hermes-stack` (reorg out/2026 → XDG).

**Projetos novos (Claude Code toolkit):** `~/Development/` + `~/.claude/commands/novo-projeto.md` — ver **`~/Development/README-toolkit-tijolao.md`**. Não misturar com `~/src/` nem `~/.hermes`.

---

## 2. Mapa de caminhos

```
~/.hermes/                          ← agente (runtime + config.yaml + plans/ + memories/)
~/.local/secrets/.env               ← segredos (única fonte)
~/.local/secrets/compose.env        ← só API_SERVER_KEY (gerado pelo tijolao; evita warnings do Compose)
~/.config/hermes-stack/             ← docker-compose.tijolao.yml, swarm.yaml, n8n/, scripts/
~/.local/bin/tijolao                ← wrapper stack
~/.local/state/n8n/                 ← dados n8n + stack.env (N8N_ENCRYPTION_KEY)
~/src/hermes-workspace/             ← build do container workspace
~/src/                              ← código (bind mount)
```

**Planos ULTRON:** `~/.hermes/plans/ULTRON.md` (roadmap), este arquivo (operação), `~/.config/hermes-stack/ULTRON-N8N.md` (n8n).

**Repo público (subset):** [ultron-hermes](https://github.com/doutortenente/ultron-hermes) — runtime e `memories/` ficam fora do Git.

---

## 3. Arquitetura em uma página

| Peça | Porta loopback | Container / processo |
|------|----------------|----------------------|
| Hermes gateway + API | 8642 | `hermes-agent` |
| Dashboard Hermes | 9119 | mesmo container (s6) |
| hermes-workspace | 3000 | `hermes-workspace` |
| n8n ULTRON | 5678 | `ultron-n8n` |
| 9Router | 20128 | serviço no host |

Rede: `network_mode: host` nos três containers (9Router/Ollama em `127.0.0.1` do host).

**Papéis:** Hermes = conversa e tools; n8n = rotinas repetíveis; workspace = UI; compose = orquestra containers.

**Swarm / especialistas:** `swarm.yaml` com `workers: []` **de propósito** (desligado 18-out). `AGENTS.md` no hermes-stack é referência se voltar a frota — não é operação atual.

---

## 4. Subir, provar, operar

### 4.1 Subir stack

```bash
tijolao up
```

Faz: ancora tmux `hermes-swarm`, `docker compose up`, espera health, opcional `ultron-n8n-bootstrap.sh`.

### 4.2 Prova única ULTRON

```bash
~/.config/hermes-stack/scripts/ultron-health.sh
```

Exit 0 = gateway, dashboard, workspace, n8n, webhook `ultron-test`, Telegram connected, hook no YAML, SOUL §9, memories.

### 4.3 Status rápido

```bash
tijolao status
hermes doctor
curl -fsS http://127.0.0.1:8642/health
```

### 4.4 Após reboot ou update

```bash
tijolao up          # se containers parados
ultron-health.sh
```

Telegram: até ~30 s após restart do agent até `gateway_state.json` → `connected`.

### 4.5 Logs

```bash
tijolao logs agent
tijolao logs ws
tijolao logs n8n
```

---

## 5. Roadmap ULTRON (estado real)

| Fase | Conteúdo | Estado |
|------|----------|--------|
| 0 | Blueprint | Feito |
| 1 | Saúde repetível + doc swarm alinhada | Feito (health script; `AGENTS.md` + `swarm.yaml` vazios) |
| 2 | n8n no compose | Feito |
| 3 | Hook Hermes → n8n | Feito (`ultron-n8n-session-end` → `/webhook/ultron-hermes`) |
| 4 | Identidade SOUL §9 | Feito |
| 5 | Especialistas tmux | **Off** — só se você pedir |
| **6** | **Higiene config + doc + compose sem ruído** | **Em curso (este doc + commits hermes-stack)** |
| **7** | **n8n com nós úteis** (quando você definir a rotina) | Pendente — hook hoje só normaliza payload |

Detalhe histórico: `plans/ULTRON.md`.

---

## 6. Melhorias de config (aplicadas 01-nov-2026)

| # | Problema | Correção |
|---|----------|----------|
| 1 | Compose interpretava `$` dentro de `DASHBOARD_PASSWORD_HASH` (scrypt) → warnings | `tijolao` gera `~/.local/secrets/compose.env` só com `API_SERVER_KEY` para interpolação; containers continuam com cofre completo via `env_file` |
| 2 | Doc n8n dizia hook “opcional” | `ULTRON-N8N.md` alinhado ao `config.yaml` atual |
| 3 | `ULTRON.md` §6 obsoleto (“subir n8n”) | §6 substituído por fila Fase 6–7 |
| 4 | Sem README da stack | `~/.config/hermes-stack/README.md` |
| 5 | JSON n8n `active: false` vs runtime | Workflows versionados com `active: true` |
| 6 | Bootstrap reiniciava n8n sempre | Reinício só se import alterou checksum |
| 7 | Comentário compose citava `tijolao-ai` | Corrigido para XDG |
| 8 | Skill workspace citava mount tmux antigo | Atualizado para `~/.local/state/tmux-swarm` |
| 9 | `sessions` implícito no YAML | Bloco explícito em `config.yaml` (auto_prune, retention) |
| 10 | `gateway-starts.log` na raiz | Movido para `logs/` |

---

## 7. Fila restante (config — sem produto imposto)

Execute **quando quiser**, nesta ordem:

1. **`hermes doctor --fix`** — o que for auto-corrigível (terminal/Docker falso positivo se doctor rodou fora do host).
2. **Vulnerabilidades npm** (`web`, `ui-tui` workspaces) — acompanhar upstream; rebuild `tijolao build` após lockfile.
3. **Pin da imagem** `hermes-agent:tijolao` — trocar `FROM latest` por tag/digest quando fizer release consciente (`tijolao build --pull` + teste health).
4. **Patch `reasoning-duplicate-field`** — no build, se log disser “já no upstream”, remover patch.
5. **`state.db` ~150 MB+** — com gateway **parado**: `hermes sessions optimize` (skill `hermes-runtime-maintenance`).
6. **Hook n8n** — hoje gera execução vazia; próximo passo **seu**: desenhar nós no workflow `ultron-hermes-hook.json` **ou** remover bloco `hooks.outbound` em `config.yaml` até existir rotina.
7. **Commit git** em `~/.config/hermes-stack` (repo local) após revisar diff.
8. **`compression.enabled`** — avaliar ligar no `config.yaml` se RAM apertar (8 GB); decisão tua.
9. **MCP Playwright `@latest`** — pin de versão quando estabilizar ambiente.

**Fora de escopo desta fila (não sugerido como default):** digest Telegram, cron matinal genérico, religar frota de especialistas, “consertar túnel Cloudflare” como projeto paralelo.

---

## 8. n8n e hook Hermes

- Workflows versionados: `~/.config/hermes-stack/n8n/workflows/`
- Bootstrap: `scripts/ultron-n8n-bootstrap.sh`
- **Hook Hermes → n8n:** **desligado** (01-nov-2026) — removido de `config.yaml`. Snippet comentado em `n8n/hooks.ultron.example.yaml`.

Payload (se reativar): doc Hermes [hooks](https://github.com/nousresearch/hermes-agent/blob/main/website/docs/user-guide/features/hooks.md).

**URL pública n8n (Tailscale funnel):** definida no compose como `WEBHOOK_URL` — ver `ULTRON-N8N.md`. Uso local: `127.0.0.1:5678`.

**n8n → Hermes (API):** `POST /v1/chat/completions` + `Authorization: Bearer` + `API_SERVER_KEY` (mesmo token do workspace).

---

## 9. Manutenção `~/.hermes`

- **Nunca** mover/apagar raiz (`*.db`, `gateway.*`, `home/`) sem skill `hermes-runtime-maintenance`.
- **Config:** preferir `hermes config set`; backup em `backups/config/` antes de mudanças grandes.
- **Mapa de pastas:** `plans/MAPA-HERMES-HOME.md`.
- **MCP:** `plans/reference/MCP-AUDIT.md`.

---

## 10. Checklist pós-mudança (sempre)

```bash
~/.config/hermes-stack/scripts/ultron-health.sh
tijolao status 2>&1 | rg -i warning || echo "compose: sem warnings de variavel"
hermes config check
```

---

*Fim do guia operacional ULTRON.*
