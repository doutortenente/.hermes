# ULTRON / Hermes home

Pasta de **runtime** do [Hermes Agent](https://github.com/nousresearch/hermes-agent) (estado vivo + config), espelhada aqui como subset versionável.

## O que vive neste repo

| Arquivo/pasta | Papel |
|---------------|-------|
| `config.yaml` | Config do agente: modelo (9Router local), fallbacks, MCP servers, toolsets, memory, skills |
| `SOUL.md` | Identidade e postura do assistente (slot #1 do system prompt) |
| `skills/` | Skills instaladas, organizadas por categoria |
| `.env.example` | Nomes das variáveis que o `config.yaml` referencia — valores só no `.env` real (gitignored) |

## O que NÃO vive neste repo

A raiz real (`~/.hermes`) contém dezenas de arquivos oficiais de runtime (`state.db`, `gateway.*`, caches, `memories/`, `sessions/`, `logs/`, plugins locais) que são **ignorados pelo Git de propósito** — ver `.gitignore`, que segue o template oficial de [profile distributions](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions). Não trate a raiz como "projeto Git" e não mova esses arquivos.

- Skills arquivadas ficam em `skills/.archive/` (o loader não as carrega).
- Estado do Skills Hub (`skills/.hub/`) é artefato gerado — nunca versionado.
- Config é editado via `hermes config set`, não à mão (refs `${ENV_VAR}` são preservadas).
