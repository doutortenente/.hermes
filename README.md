# ULTRON / Hermes home

Pasta de **runtime** do [Hermes Agent](https://github.com/nousresearch/hermes-agent) (estado vivo + config), espelhada aqui como subset versionável.

## O que vive neste repo

| Arquivo/pasta | Papel |
|---------------|-------|
| `config.yaml` | Config do agente: modelo (9Router local), fallbacks, MCP servers, toolsets, memory, skills |
| `SOUL.md` | Identidade e postura do assistente (slot #1 do system prompt) |
| `skills/` | Skills instaladas, organizadas por categoria (`devops/`, `software-development/`, …) |
| `.env.example` | Nomes das variáveis de ambiente que o `config.yaml` espera — valores só no `.env` real (gitignored) |

## O que NÃO vive neste repo

A raiz real (`~/.hermes`) contém dezenas de arquivos oficiais de runtime (`*.db`, `gateway.*`, caches, `memories/`, `sessions/`, `logs/`, `plans/`, plugins locais) que são **ignorados pelo Git de propósito** — ver `.gitignore`. Não trate a raiz como "projeto Git" e não mova esses arquivos.

- Skills arquivadas ficam em `skills/.archive/` (não são carregadas pelo agente).
- Estado do Skills Hub (`skills/.hub/`) é artefato gerado — nunca versionado.
- Config é editado via `hermes config set`, não à mão (as refs `${ENV_VAR}` são preservadas).
- A cadeia operacional de providers (9Router → fallbacks) está documentada na skill `skills/devops/ninerouter-hermes/SKILL.md`.
