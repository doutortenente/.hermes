---
name: cinnamon-theming
description: "Apply Cinnamon themes and wallpapers on Linux Mint via SSH."
---

# Tema e papel de parede no Cinnamon (Linux Mint) via ponte SSH

O host roda Linux Mint com desktop Cinnamon. Hermes opera via ponte SSH container→host (skill `tijolao-ssh-bridge`).

## Duas chaves gsettings independentes

| Componente | Chave gsettings | Exemplo |
|---|---|---|
| Tema shell (painel, menus, janelas do desktop) | `org.cinnamon.theme name` | `gsettings set org.cinnamon.theme name "CGreen"` |
| Papel de parede da área de trabalho | `org.cinnamon.desktop.background picture-uri` | `gsettings set org.cinnamon.desktop.background picture-uri "file:///home/dr/Imagens/wallpapers/CGreen.png"` |

**Armadilha principal**: o Cinnamon NÃO troca papel de parede automaticamente ao aplicar tema. São chaves separadas. Declarar "feito" só no tema deixa o wallpaper no padrão antigo.

## Procedimento completo (zip de tema baixado no host)

1. **Extrair tema** para `~/.themes/<NomeDoTema>/` (local padrão do Cinnamon)
   ```bash
   mkdir -p ~/.themes && unzip -o ~/Downloads/<Tema>.zip -d ~/.themes
   ```
2. **Verificar `theme.json`** para confirmar nome exato do tema
   ```bash
   cat ~/.themes/<NomeDoTema>/cinnamon/theme.json
   ```
3. **Aplicar tema** via gsettings (efeito imediato, sem reiniciar)
   ```bash
   gsettings set org.cinnamon.theme name "<NomeDoTema>"
   ```
4. **Copiar wallpaper** do tema para local persistente do usuário
   ```bash
   mkdir -p ~/Imagens/wallpapers
   cp ~/.themes/<NomeDoTema>/cinnamon/Wallpaper.png ~/Imagens/wallpapers/<NomeDoTema>.png
   ```
   *Nota: pasta `~/Imagens/wallpapers/` pode não existir — criar com `mkdir -p`. O arquivo do wallpaper precisa existir ANTES de setar a URI.*
5. **Aplicar wallpaper** via gsettings
   ```bash
   gsettings set org.cinnamon.desktop.background picture-uri "file:///home/dr/Imagens/wallpapers/<NomeDoTema>.png"
   ```
6. **Deletar zip** original
   ```bash
   rm ~/Downloads/<Tema>.zip
   ```
7. **Verificar tudo** num único comando (evita declaração prematura)
   ```bash
   gsettings get org.cinnamon.theme name
   gsettings get org.cinnamon.desktop.background picture-uri
   test -f ~/Imagens/wallpapers/<NomeDoTema>.png && echo WALLPAPER_OK || echo WALLPAPER_FALTA
   test -e ~/Downloads/<Tema>.zip && echo ZIP_EXISTE || echo ZIP_DELETADO
   ```

## Armadilhas adicionais

- **Nome do tema no `theme.json`** é a fonte da verdade; não confiar no nome da pasta do zip.
- **Temas de zip externos** geralmente só trazem parte Cinnamon (shell theme). GTK theme (janelas de apps) costuma vir separado — não prometer o que o zip não tem.
- **Refresh visual**: se na tela o papel de parede ainda estiver velho, é o Cinnamon não tendo redesenhado a área de trabalho — resolve fechando a sessão e abrindo de novo (logout/login). O gsettings já confirmou a troca.
- **Verificação por comando único** obrigatória antes de dizer "pronto" ao usuário.

## Comandos úteis de diagnóstico

```bash
# Listar temas instalados
ls ~/.themes/

# Ver tema atual
gsettings get org.cinnamon.theme name

# Ver wallpaper atual
gsettings get org.cinnamon.desktop.background picture-uri

# Reset para padrão Mint
gsettings set org.cinnamon.theme name 'Mint-Y'
gsettings set org.cinnamon.desktop.background picture-uri 'file:///usr/share/backgrounds/linuxmint-wallpapers/jpanchal_cpu.jpg'
```

## Integração com ponte SSH

Todos os comandos acima rodam no host via:

```bash
ssh -i ~/.ssh/id_ed25519 -o BatchMode=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null dr@localhost '<comando>'
```

Consulte skill `tijolao-ssh-bridge` para detalhes da ponte (chaves, flags, autenticação).