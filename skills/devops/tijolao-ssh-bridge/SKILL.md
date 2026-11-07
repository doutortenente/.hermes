---
name: tijolao-ssh-bridge
description: "Use quando o Hermes operar o host via SSH."
---

# Ponte SSH container → host (Tijolão)

O Hermes roda em container Docker (`network_mode: host`) no notebook. Desde 2026-09-23 existe ponte SSH permanente pro host.

## Estado atual

- **sshd do host**: ligado (`openssh-server`, usuário `dr`).
- **Chave**: container gera `~/.ssh/id_ed25519`; pública (comentário `hermes-container`) em `~/.ssh/authorized_keys` do `dr` no host.
- **Comando**: `ssh -i /home/dr/.hermes/home/.ssh/id_ed25519 -o BatchMode=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null dr@localhost <cmd>` — sem senha, de dentro do container.
- **Por que as flags extras**: em container novo o `~/.ssh/known_hosts` não existe e o ssh puro falha com `Host key verification failed`. `StrictHostKeyChecking=no` + `UserKnownHostsFile=/dev/null` aceitam o host na primeira conexão sem arquivo. É localhost do próprio host — risco nulo.
- **Alcance**: tudo que o container NÃO monta — `~/.config` (inclui `rclone.conf` do OneDrive), `~/.local`, systemd, apt, processos.

## Autenticação com senha (1x, para replicar em outra máquina)

Container não tem `sshpass` nem `pexpect` nem root. PTY puro em Python stdlib:

```python
import os, pty, select, time

def ssh_try(password, cmd):
    pid, fd = pty.fork()
    if pid == 0:
        os.execvp('ssh', ['ssh', '-o', 'StrictHostKeyChecking=no',
                          '-o', 'UserKnownHostsFile=/dev/null',
                          '-o', 'ConnectTimeout=8',
                          '-o', 'PreferredAuthentications=password',
                          '-o', 'NumberOfPasswordPrompts=1',
                          'dr@localhost', cmd])
        os._exit(1)
    out = b''
    deadline = time.time() + 60
    sent = False
    while time.time() < deadline:
        r, _, _ = select.select([fd], [], [], 0.5)
        if r:
            try:
                chunk = os.read(fd, 4096)
            except OSError:
                break
            if not chunk:
                break
            out += chunk
            if (b'password' in out.lower() or b'permission denied' in out.lower()) and not sent:
                os.write(fd, password.encode() + b'\n')
                sent = True
    try: os.close(fd)
    except OSError: pass
    try: os.waitpid(pid, os.WNOHANG)
    except ChildProcessError: pass
    return out.decode(errors='replace')
```

Depois instalar a chave e nunca mais usar senha:

```python
PUB = open(os.path.expanduser('~/.ssh/id_ed25519.pub')).read().strip()
cmd = ('mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && '
       'chmod 600 ~/.ssh/authorized_keys && '
       '(grep -qF "hermes-container" ~/.ssh/authorized_keys || echo "' + PUB + '" >> ~/.ssh/authorized_keys) && '
       'echo CHAVE-INSTALADA')
```

Chaves com espaço/aspas: concatenar strings, não f-string tripla (quebra quando o código viaja em `python3 -c "..."`).

## Armadilhas

- **Chave sempre por caminho absoluto (`/home/dr/.hermes/home/.ssh/id_ed25519`).** Se o `~` chega cru ao ssh (ex.: dentro de `SSH="ssh -i ~/..."`), o ssh expande pelo passwd do usuário `hermes` (`/opt/data`), não pelo `$HOME` → `Identity file /opt/data/.ssh/id_ed25519 not accessible` + `Permission denied (publickey,password)`.
- **Tailscale SSH (RunSSH) não atende conexões originadas do próprio host.** Do container com `network_mode: host`, porta 22 via IP da tailnet ou MagicDNS dá `Connection refused`. Usar `localhost` direto.
- **Não chutar senha**: 3 erros e o sshd pode negar. Se duas credenciais conhecidas falharem, pedir ao operador.
- **OAuth OneDrive headless (rclone authorize)**: token impresso trunca fácil em copy manual (1 caractere perdido = `InvalidAuthenticationToken`). Recuperar do log bruto: `~/.hermes/logs/process-results/proc_<id>.json`.
- **rclone onedrive com `config_is_local=false` exige `drive_id` e `drive_type` manuais.** Descobrir via `GET https://graph.microsoft.com/v1.0/me/drive` com o access_token e escrever o config à mão.
- **MFA da Microsoft**: máquina nova pede código por e-mail alternativo. O navegador Playwright do Hermes resolve o fluxo inteiro (login, código, termos, stay-signed-in).
- **authorized_keys do host com dono errado (root)**: operador recebe "Permissão negada" rodando o script local. Causa: `~/.ssh` criado com sudo. Correção: `sudo chown -R $USER:$USER ~/.ssh`.
- **Unit systemd --user fantasma**: pode apontar WorkingDirectory pra pasta que não existe (extracao-clinica-sasi → ~/projetos/EXTRACAO-CLINICA-SASI). Antes de apagar pasta de projeto: grep refs em ~/.config/systemd/user/, ~/.hermes/config.yaml e ~/.local/bin/; ref viva = não apagar, reportar.
- **Estado do Docker (containers, daemon, imagens, volumes): SEMPRE via SSH ao host.** O socket `/var/run/docker.sock` não é montado no container — `docker` daqui falha com "Cannot connect to the Docker daemon". De dentro, só se vê a si mesmo (`ls /.dockerenv`, `/proc/1/cgroup`); o quadro completo (`docker ps -a`, `docker stats`, `docker inspect`) sai do host via ponte SSH.
- **Diagnóstico de ferramentas do operador ("git/gh no meu PC"): checar SEMPRE no host via SSH antes de afirmar.** O container tem HOME próprio (`/home/dr/.hermes/home`) pelado — sem `.gitconfig`, sem `gh` — e responde como se fosse a máquina do operador. Primeira checagem dentro do container = diagnóstico falso (aconteceu em 02-out-2026: afirmei que o PC não tinha gh; o host tinha git 2.43 configurado + gh 2.45 logado).
- **Antes de mover/apagar pasta aninhada dentro de um repo: `git -C <repo> ls-files <pasta>` primeiro.** Pasta aninhada pode ser trackeada (parte do repo) ou ignorada (lixo) — mover trackeada mostra arquivos como deletados no status (botch de 02-out-2026: arranquei EXTRACAO-CLINICA-SASI trackeada de dentro de 2 clones; correção = mover de volta pro lugar).
- **grep em diretórios com binários (`~/.local/bin`, `~/src/scripts/*/bin`): usar `grep -I` (ou `--files` + filtro de extensão).** Sem isso o grep casa string dentro de ELF (rclone, container-use) e despeja megabytes de lixo binário no output, queimando contexto. Resultado de grep em caminho com binário = primeiro suspeitar de casamento dentro de executável, não de achado real.
- **Identificar container desconhecido no host: cruzar StartedAt com data de instalação dos binários.** Ex.: `docker inspect <nome> --format '{{.State.StartedAt}}'` + `ls -la ~/.local/bin/` no mesmo dia = quem o subiu. Container dagger-engine é o motor do `container-use` (comando `cu`), NÃO parte do build do Hermes Desktop — não confundir na auditoria de containers.
- **Cleanup de estação: sempre para quarentena, nunca `rm`.** Mover para `~/.local/share/quarentena-config-$(date +%Y%m%d)/<origem-relativa>/`, preservando o caminho relativo. Assim dá qualquer um pra desfazer com `mv` de volta. Só fazer `rm -rf` da quarentena depois que o operador confirmar explicitamente, e só após provar before/after com `du -sh` e `df -h /`. `cp -p` para backups críticos antes (profiles.ini, mimeapps.list).

## Mapa do stack (verificar no compose antes de afirmar)

- Fonte da verdade do layout: `~/.config/hermes-stack/docker-compose.tijolao.yml` (arquivo próprio, não override do upstream; sobe via wrapper `tijolao` com --env-file do cofre).
- 3 serviços no compose `hermes`: `hermes-agent` (gateway 8642 + dashboard 9119 como processos FILHOS do mesmo container, via `HERMES_DASHBOARD=1` — não são 2 containers), `hermes-workspace` (UI web na 3000, `depends_on` agente healthy) e `n8n` (container `ultron-n8n`).
- Tudo `network_mode: host` e bind mount de `~/.hermes`, `~/src` e cofre — portas e caminhos aparecem iguais no host e nos containers.
- Container alheio ao stack: `dagger-engine-v*` (motor do `container-use`/`cu`).
- Ao perguntar "o que é X / onde roda X": ler o compose e `docker inspect --format '{{.Config.Env}}'`, nunca chutar a partir do `ps`.
- **`~/.local/share/n8n` NÃO é o banco vivo.** O volume do container é `/home/dr/.local/state/n8n:/home/node/.n8n`. Banco, `config` (encryptionKey) e `stack.env` estão em `~/.local/state/n8n`. A pasta em `share/` é resíduo.
- **n8n no ar ≠ chave de API.** Dono sem e-mail/senha deixa a API em 401 (`X-N8N-API-KEY` exigido) e `user_api_keys` vazia. Webhook ativo (`/webhook/<path>`) funciona sem essa chave. `encryptionKey` em `config` cifra credenciais salvas — não autentica cliente.
- **Pedido de dado de conexão: mostrar o valor, não esconder.** Quando ele pede URL, porta, chave ou token pra ligar outro programa, imprimir o valor lido. Não apagar o arquivo de config inteiro pra cumprir isso — tirar só a linha que censura.
- **Nunca forjar JWT no sqlite do n8n.** A API valida assinatura (`iss=n8n`, `aud=public-api` ou `aud=mcp-server-api`) e a chave forjada quebra no upgrade e some no undo. A chave nasce pela rota do próprio n8n, logado como dono.
- **Na 2.x o MCP já vem ligado.** Desligar de vez = `N8N_DISABLED_MODULES=mcp`. A flag antiga `N8N_ENABLED_MODULES=mcp` era da 1.x e não é o que liga o acesso. O acesso só vale com o par no compose: `N8N_MCP_MANAGED_BY_ENV=true` junto de `N8N_MCP_ACCESS_ENABLED=true` (desde 2.20). Sozinha, a segunda é ignorada. Com o par, a tela correspondente fica somente leitura e o valor reaplica na subida. Recriar: `docker compose up -d --no-deps --force-recreate n8n` — `restart` herda o env antigo.
- **A chave MCP nasce em `POST /rest/mcp/api-key/rotate`, com cookie de dono.** A documentação pública só descreve a tela; o controlador desta versão devolve a chave inteira nessa rota. Login: `POST /rest/login` com `emailOrLdapLoginId` + `password`. Dono gerido por env: `N8N_INSTANCE_OWNER_MANAGED_BY_ENV`, `EMAIL`, `FIRST_NAME`, `LAST_NAME`, `PASSWORD_HASH` (bcrypt). A chave pública da API (`X-N8N-API-KEY`) não substitui o Bearer do MCP.
- **Endereço do Claude = o hostname da Cloudflare, nunca o da Tailscale.** O túnel da Cloudflare roda por token (`cloudflared tunnel run --token-file`), sem `config.yml` local, e publica `https://n8n.nagaitaltda.com`. A Tailscale na 8443 é legado do `WEBHOOK_URL` e do `tailscale serve` — dois túneis no mesmo n8n não são duplicata, não apagar um por existir o outro. `Unauthorized: Authorization header not sent` nesse endereço prova que ele está no ar e que o navegador abriu sem chave; não é página. Cliente: `Authorization: Bearer <chave>` em `https://n8n.nagaitaltda.com/mcp-server/http`. Prova: initialize devolve `n8n MCP Server`.
- **Apagar fluxo na 2.x: desativar, arquivar, depois DELETE.** `DELETE /rest/workflows/{id}` num fluxo publicado devolve 409. `deactivate` sozinho continua 409 por alguns segundos; `POST .../archive` libera o `DELETE`. A CLI `n8n --help` não tem delete.
- **Arquivo de trabalho não fica no volume do serviço.** O bind `/home/dr/.local/state/n8n:/home/node/.n8n` é a pasta de dados do n8n. Escrever chave, dump ou rascunho ali (ou em `/tmp` do container, que some no recreate) mistura lixo com o banco. Gravar fora, em `~/.local/state/`, e apagar a cópia de dentro antes de encerrar. `docker cp` pra um caminho já montado não é a via — escrever no path do host direto.

## Armadilhas

- **`ps aux` do host MOSTRA processos de container Docker.** Ver gateway/dashboard/workspace na listagem do host NÃO significa que são processos nativos — são filhos do container rodando em rede host. Classificar serviço (container vs nativo) via `docker inspect`, não via ps; afirmar "nativo" por ps = diagnóstico falso.
- **Auditoria "estado da máquina" = cobrir TODAS as superfícies antes de concluir.** Containers (`docker ps -a`), processos nativos (`ps aux --sort=-%mem`), portas (`ss -tlnp`), systemd --user, e apps desktop (flatpak `ps aux | grep -i <app>`). Chegar ao fim da auditoria sem olhar uma dessas camadas gera "não tem X" falso — X pode existir fora do docker (app desktop, serviço nativo, flatpak system).
- **Desinstalar flatpak system (`sudo flatpak uninstall --delete-data`) exige senha de sudo — não dá pela ponte SSH sem senha.** O que dá sem sudo: `pkill -9 -f '<caminho-do-app>'` (processos rodam como `dr`). Matar processos + entregar o comando de uninstall ao operador é o pacote completo.
- **Antes de deletar app que guarda dados (cofre de senhas, cliente de e-mail): provar que está vazio primeiro.** Ex.: Bitwarden desktop: `data.json` sem `userId` = nunca logou; `blob_storage/` vazio = zero itens locais. Só então deletar; se houver dados, reportar antes.

## Artefatos no disco (montados nos dois lados)

- `~/src/scripts/nuvem/bin/rclone` — binário v1.75.1 (movido de ~/projetos quando a pasta foi queimada; link `~/.local/bin/rclone` aponta pra cá).
- `~/src/scripts/nuvem/liga-onedrive-no-pc.sh` — copia config pro `~/.config/rclone` do host e testa.
- Cofre: `~/.local/secrets/.env` (03-out-2026; antes era ~/projetos/.env). `~/.hermes/.env` → symlink.
- Remote: `~/.hermes/home/.config/rclone/rclone.conf` (container) e `~/.config/rclone/rclone.conf` (host, idêntico, 0600).

## Reorganização de raízes (03-out-2026)

- Raiz única de código = `~/src`. `~/projetos` foi esvaziada (repos → src, .env → ~/.local/secrets) e apagada por ordem do operador. Refs atualizadas com sed em: ~/.local/bin/*, wrappers do tijolao-ai, docker-compose.tijolao.yml.
- **Armadilha**: docker-compose montava `/home/dr/projetos:/home/dr/projetos` pro symlink ~/.hermes/.env funcionar no container. Agora o .env vive em ~/.local/secrets, que já é montado :ro no container — o mount de projetos não é mais necessário. Se o container reiniciar com `docker start` (config antiga), o docker pode recriar /home/dr/projetos vazia — apagar de novo.
- **Ao mover cofre (.env): provar integridade com md5sum antes e depois.** Hash 7041395c... manteve-se idêntico na mudança de 03-out-2026.
