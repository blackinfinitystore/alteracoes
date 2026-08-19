# Bot Discord de salas + API no TCP

Registro técnico das alterações feitas no **servidor de jogo (TCP)** e na **VPS** para o bot Discord criar salas CS 4v4 e BR Classic. O jogador entra no jogo com ID + senha; o primeiro a entrar vira dono da sala.

Este repositório **não** contém token, senha do painel, `BOT_API_SECRET` nem `.env` de produção.

---

## Arquitetura

```
Discord  ──slash──►  bot.py (VPS TCP)
                       │
                       │  HTTP loopback :7744
                       │  header X-Bot-Secret
                       ▼
                  tcp-server
                  CustomRoom (BotOwned)
                       │
                       ▼
              cliente Free Fire (ID + senha)

bot.py também sobe o painel web em 0.0.0.0:7745
```

O bot **precisa** rodar na mesma máquina do `tcp-server`, porque a API escuta só em `127.0.0.1:7744` (não fica exposta na internet).

---

## O que mudou no código do TCP

### Arquivo novo

`tcp-server/internal/tcp/customroom_botapi.go`

- HTTP local com autenticação por secret.
- Cria / lista / consulta / fecha salas marcadas `BotOwned`.
- Gera senha numérica de 4 dígitos se o POST não mandar `password`.
- Expira sala vazia após **45 minutos**.

### Arquivos existentes (detalhe em `tcp-server/PATCHES.md`)

| Arquivo | Mudança |
|---------|---------|
| `internal/config/config.go` | `BOT_API_ADDR` e `BOT_API_SECRET` |
| `internal/tcp/server.go` | `s.startBotAPI()` no boot |
| `internal/tcp/customroom.go` | flag `BotOwned`, 1º jogador = dono, saída transfere dono, vazia não dismiss |

Comportamento da sala bot (diferente da sala normal):

1. Nasce **sem dono** (`CreatorID = 0`, nome "Discord").
2. Primeiro jogador a entrar vira **owner** e já fica ready.
3. Se o dono sair e ainda tiver gente, o próximo da lista vira dono.
4. Se esvaziar, o código continua válido até o TTL de 45 min.
5. `/fechar` no Discord ou `DELETE /bot/rooms/{id}` remove na hora.

---

## O que foi feito na VPS (produção)

Host: VPS do **tcp-server** (mesmo processo que escuta `:13890`).  
UDP (`match-server` `:10100`) **não** foi alterado para este bot.

### 1. Binário TCP

- Compilado Linux `amd64` (`CGO_ENABLED=0`).
- Publicado em `/root/tcp-server`.
- Reinício **somente** do processo `tcp-server` (`pkill -x tcp-server` + `screen -dmS tcp`).
- Confirmado no log:

```
[TCP] escutando em :13890
[ROOM-BOT] API em http://127.0.0.1:7744  (secret no header X-Bot-Secret)
```

Health check interno:

```bash
curl -sS -H "X-Bot-Secret: $BOT_API_SECRET" http://127.0.0.1:7744/bot/health
# {"ok":true}
```

### 2. Variáveis no `/root/.env`

O `tcp-server` lê o `.env` sozinho (não precisa `source` no bash — valores com `* ^ #` quebram o shell).

```
BOT_API_ADDR=127.0.0.1:7744
BOT_API_SECRET=<gerado, 48 hex>
```

O mesmo secret entra em `salas_discord/config.json` → `tcp_secret`.

### 3. Bot Discord

Caminho: `/root/salas_discord/`

| Item | Valor |
|------|--------|
| Código | `bot.py`, `config.json`, `requirements.txt` |
| Saldos | `/root/salas_discord/data/saldos.json` |
| Screen | `discordbot` |
| Log | `/root/discordbot.log` |
| Painel | `0.0.0.0:7745` |

Pacotes no sistema:

```bash
apt-get install -y python3-pip python3-venv
python3 -m pip install -r /root/salas_discord/requirements.txt --break-system-packages
```

Start:

```bash
screen -dmS discordbot bash -lc 'cd /root/salas_discord && exec python3 bot.py >> /root/discordbot.log 2>&1'
```

### 4. Firewall

Porta **7745/tcp** liberada (`ufw` + `iptables`) para o painel web.  
**7744 permanece localhost** — não abrir no firewall.

### 5. Configuração operacional (sem secrets neste repo)

- Application ID do bot Discord preenchido no `config.json`.
- Dois donos (IDs Discord) com permissão de `/setarsaldo`, `/sala_admin`, `/salas`, `/fechar`.
- `tcp_api`: `http://127.0.0.1:7744`
- `panel_port`: `7745`
- PIX: chave ainda placeholder até o dono gravar no painel.

---

## Painel web

URL de produção: `http://<IP_DA_VPS_TCP>:7745`

Tudo configurável pelo painel (grava `config.json` e aplica em runtime):

- ID e nome do bot (username no Discord)
- Token
- Lista de donos (um ID por linha)
- `tcp_api` / `tcp_secret`
- PIX (chave, valor da sala, nome, cidade)
- Senha do painel
- Foto (avatar) do bot
- Setar saldo por ID Discord + tabela de saldos

---

## Comandos Discord (source v7 adaptada ao TCP privado)

Saldo agora é **crédito de salas** (não R$). Keys no formato `SALA…`. Senha da sala: 4 dígitos (aleatória / fixa / 0000).

| Comando | Quem | Função |
|---------|------|--------|
| `/menu` | todos | hub: criar, perfil, saldo, senha, resgatar key |
| `/criar-sala` | todos | CS 4v4 ou BR Classic |
| `/saldo` | todos | créditos de sala |
| `/resgatar` | todos | resgata key |
| `/status` | todos | TCP online + manutenção |
| `/painel` | dono | posta card público no canal |
| `/painel-dev` | dono | keys, saldo, salas, manutenção |
| `/gerar-key` | dono | gera key com N salas |
| `/setarsaldo` | dono | define créditos |
| `/sala-admin` | dono | cria na hora |
| `/salas` / `/fechar` | dono | listar e fechar |
| `/bloquear` | dono | blacklist |

Convite (trocar o client id se necessário):

```
https://discord.com/oauth2/authorize?client_id=SEU_APPLICATION_ID&scope=bot%20applications.commands&permissions=8
```

Escopo obrigatório: `bot` + `applications.commands`.

---

## Como o jogador usa a sala

1. Recebe **ID** e **senha** no Discord.
2. No jogo: Salas → busca pelo ID → senha.
3. Quem entra primeiro vira dono e dá o start.

---

## Operação / troubleshooting

```bash
screen -ls
# tcp          → ./tcp-server
# discordbot   → python3 bot.py
# udp          → match-server (não mexer)

ss -lntp | grep -E ':13890|:7744|:7745'
tail -n 50 /root/discordbot.log
grep ROOM-BOT /root/tcp-server.log | tail
```

Reiniciar só o bot:

```bash
pkill -f '/root/salas_discord/bot.py' || true
screen -S discordbot -X quit || true
screen -dmS discordbot bash -lc 'cd /root/salas_discord && exec python3 bot.py >> /root/discordbot.log 2>&1'
```

Reiniciar só o TCP (não mata UDP):

```bash
pkill -x tcp-server
screen -S tcp -X quit || true
screen -dmS tcp bash -lc 'cd /root && exec ./tcp-server >> /root/tcp-server.log 2>&1'
```

Não fazer `source /root/.env` no bash: chaves com caracteres especiais quebram o script. O Go lê o arquivo direto.

---

## Layout deste repositório

```
discord-bot/
  bot.py                 # bot + painel
  requirements.txt
  config.example.json    # copie para config.json e preencha
tcp-server/
  internal/tcp/customroom_botapi.go
  PATCHES.md             # diffs nos arquivos já existentes
```

Para publicar o TCP: copiar `customroom_botapi.go` para o tree do `tcp-server-main`, aplicar os patches, rebuild Linux e substituir `/root/tcp-server`.
