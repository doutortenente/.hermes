---
name: hermes-workspace
description: "Use when operating hermes-workspace on Tijolão."
---

# hermes-workspace no Tijolão

Interface web externa (não-oficial, fork do usuário) que já roda como container Docker próprio no host. É um command center: chat + terminal + arquivos + memória + skills numa UI só.

## Estado operacional (2-out-2026)

- **Repositório**: `~/src/hermes-workspace` no host (fork github.com/doutortenente/hermes-workspace; upstream = outsourc-e/hermes-workspace). Branch ativa `tijolao`; +4 commits à frente, 0 atrás do upstream (2-out-2026).
- **Container**: `hermes-workspace` (imagem `hermes-workspace:tijolao`), `docker restart` cuida. Stack/compose em `~/.config/hermes-stack/docker-compose.tijolao.yml` (com swarm.yaml; sobe via wrapper `tijolao` com --env-file do cofre). Medido 29-out-2026: agente v0.22.2, connection-status "enhanced", upstream parado desde 22-ago-2026.
- **Porta**: 3000 (127.0.0.1, node local) + exposta via **tailscale serve**: `https://tijolao.tail9e88c5.ts.net:10000/` (tailnet only, `--https=10000 http://127.0.0.1:3000`; desligar: `tailscale serve --https=10000 off`).
- **Integração**: env vars HERMES_API_URL (gateway 8642) + HERMES_DASHBOARD_URL (9119) + HERMES_API_TOKEN já definidas no container; probes `fetch http://127.0.0.1:8642/health` e `/api/status` respondem v0.21.1 de dentro.
- **Login**: HERMES_DASHBOARD_USER / HERMES_DASHBOARD_PASSWORD (cofre do host).
- **PWA**: site instalável no iPhone como app (Electron nativo nem existe no upstream — PWA é o caminho).

## Diretórios de trabalho (medido 2-out-2026)

| Container | WorkingDir (código) | Onde trabalha de fato |
|---|---|---|
| hermes-workspace | `/app` | `/home/dr/src` — env `HERMES_WORKSPACE_DIR` vence o config.yaml (`configuredDefaultWorkspace` no router-*.js) |
| hermes-agent | `/opt/hermes` | gateway/Telegram: `/opt/data` (s6 run faz `cd /opt/data`, HOME=/opt/data; `terminal.cwd: "."` vira HOME em gateway/run.py). CLI via docker exec: `/opt/hermes` |

- `/opt/data` = volume anônimo (`/var/lib/docker/volumes/197f05…/_data`): só root abre no host; o workspace NÃO monta → arquivo criado pelo Telegram com caminho relativo some da vista.
- Mounts do workspace (medir com `docker inspect hermes-workspace`): `src`, `.hermes`, `.local/bin` (ro), `~/.config/hermes-stack/swarm.yaml` → `/app/swarm.yaml`, `~/.local/state/tmux-swarm` (socket tmux host↔container).
- Medir: `docker inspect <c> --format '{{.Config.WorkingDir}}'` e `readlink /proc/<pid>/cwd`.

## Saúde em um comando (no host via ponte SSH)

curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:3000
curl -s http://127.0.0.1:8642/health | head -c 100   # gateway do agente
tailscale serve status                               # 10000 -> 3000 (tailnet only)
docker ps --filter name=hermes-workspace --format '{{.Status}}'

## Atualização (mensal, sob pedido)

1. `git -C ~/src/hermes-workspace fetch origin upstream`
2. `git -C ~/src/hermes-workspace rev-list --left-right --count HEAD...upstream/main` (ver dívida)
3. Rebuild via wrapper `tijolao` (compose `~/.config/hermes-stack/docker-compose.tijolao.yml`; exige --env-file do cofre, senão API_SERVER_KEY falta e o compose aborta)
4. Refazer prova de saúde acima.

## Armadilhas

- **"hermes workspace" não é comando do CLI** — hermes workspace é este projeto externo; o built-in do Hermes é `hermes dashboard` (9119).
- **NÃO citado na doc oficial do Hermes** = projeto não-oficial, mas funcional e já integrado no ambiente.
- **Acesso do iPhone exige a tailnet ligada** (aparelho ficava offline dias — sem tailnet, o URL não abre).
- **Credenciais só no cofre** — nunca imprimir em chat.
- **`/api/memory` (rota legada) dá 404 do agente** — inofensivo: a tela de Memória usa `/api/memory/list` e `/read?path=memories/MEMORY.md`. Prova funcional = `/api/connection-status` dar "enhanced".
- **`/api/health` do workspace** responde HTML (SPA); para versão usar endpoints do dashboard/gateway.

## Referências relacionadas

- `ninerouter-hermes` skill (funnel tailscale na mesma mecânica, porta 20128)
- `tijolao-ssh-bridge` skill (ponte SSH para rodar comandos no host)
- Doc oficial do dashboard (REST API que o workspace consome): https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard