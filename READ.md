# Alterações Barbosa Server — 21/08/2026

Pacote de release: binários Linux `amd64` + este changelog.  
Pasta local: `SERVIDOR/RELEASE_2026-08-21/`

| Arquivo | Função | Deploy VPS |
|---------|--------|------------|
| `tcp-server` | Lobby, fila, grupos, salas, handoff UDP | `204.157.166.35` → `/root/tcp-server` (screen `tcp`) |
| `match-server` | Partida UDP (CS/BR/treino) | `204.157.166.35` → `/root/match-server` (screen `match`) |

HTTP (`htp41.py`) fica na VPS `129.121.41.138` — alterações de etiquetas/whitelist estão no Mongo + arquivo remoto (não vão nestes bins).

---

## 1. Matchmaking — estimativa sempre 00:00

**Problema:** timer de procura subia, mas `ESTIMATIVA: 00:00`.

**Causa:** o servidor mandava `MatchmakingStartNtf` no subcmd **4** (`GROUPCANCEL`). No protocolo oficial (`EMatchmaking.Proto`), `START_NTF` é **11**.

**Arquivos:**
- `internal/tcp/dispatch.go` — `SubStartNtf = 11`, `SubGroupCancel = 4`, `SubStopNtf = 14`
- `internal/tcp/handlers_matchmaking.go` — `pushWaitEstimate` envia START(1) + START_NTF(11); piso de estimativa por modo
- `internal/tcp/group.go` — squad também usa START_NTF(11); piso mínimo da estimativa

**Resultado:** UI recebe `AvgWaitTimeSec` no canal certo.

---

## 2. BR Ranked — modo teste desligado (público)

**Antes (teste):** timeout **12s**, mínimo **1** jogador → partida quase vazia.

**Agora (produção):**
- Fecha na hora com **≥45** jogadores
- No timeout de **1:30** só inicia com **≥35**
- Abaixo de 35 → continua na fila

**Arquivos:**
- `internal/tcp/matchmaker.go` — `BRRankedIdealPlayers=45`, `BRRankedMinPlayers=35`, `BRRankedTimeout=90s`
- `internal/tcp/handlers_matchmaking.go` — piso da estimativa UI BR ranked = 90s

**Ilha / start delay:** `BR_START_DELAY=40` no `.env` do match (já estava).

---

## 3. Troca de modo sozinha (CS ↔ BR ↔ Treino)

**Problema:** ao selecionar um modo, a UI saltava para outro; precisava clicar Start rápido.

**Causas:**
1. `GroupChange` / `CREATE→change` zeravam `match_mode` quando o cliente mandava `0` (campo omitido no proto3) → UI caía em outro modo.
2. `keyOf()` no matchmaker **mutava** `SelectedMatchMode` / `SelectedGroupMode` da sessão (forçava BR ranked) sempre que estimava fila → NTF/estado desalinhados com a UI.

**Arquivos:**
- `internal/tcp/group.go` — só atualiza `MatchMode` se `!= 0`; defaults no CREATE novo (CS=6, BR=2, treino=5)
- `internal/tcp/matchmaker.go` — `keyOf` só altera a **chave da fila**, sem escrever na sessão

---

## 4. Etiquetas (Estilo de Batalha / Social) — equipar e aparecer no perfil

### 4.1 TCP (persistência auxiliar)
- `internal/store/career_stats.go` — `SocialInfo.BattleTags` / `SocialTags`; `SaveSocialInfo` grava + `show_tag_ids`; `EnsureBattleTagsUnlocked` (progresso 999)
- `internal/tcp/social.go` — update salva tags; `buildSocialBasic` devolve tags; helpers de wire packed

### 4.2 HTTP (`htp41.py` na VPS HTTP) — caminho real do cliente 1.70
O cliente grava via **`POST /UpdateSocialBasicInfo`**, não só TCP.

**Problemas:**
- Handler só salvava a bio (`signature`)
- `GetPlayerPersonalShow` não devolvia tags
- Proto local `generated_proto_fixed_pb2.SocialBasicInfo` **não tem** `battle_tag` → serialização engolia as tags

**Correções:**
- `/UpdateSocialBasicInfo` — parse com `tcpgame_pb2`, grava `social.battle_tag` / `social.social_tag` / `show_tag_ids`
- `/GetBattleTag` — libera tags 1101–1110 com count 999; devolve `show_tag_ids`
- `GetPlayerPersonalShow` — `_fill_social_tags()` via `tcpgame_pb2` + `MergeFromString` (campos preservados no wire)
- Mongo exemplo (owner): `battle_tag: [1101,…]`, `show_tag_ids: […]`

---

## 5. Whitelist / manutenção — desligada para público

**Config Mongo** (`freefire_server.server_config`):
```json
{ "_id": "authorization", "require_authorization": false }
```

- TCP: `Mongo.Authorized()` libera todos quando `require_authorization=false`
- HTTP: `is_account_authorized()` idem (cache ~30s)

**Não remove** a collection `authorized_players` — só deixa de exigir. Para religar: setar `true` de novo.

---

## 6. Outros (sessão anterior / já em produção)

Resumo do que já estava no servidor nesta sessão (contexto):

| Área | O quê |
|------|--------|
| Spectate admin | Redis `admin:spectate:{uid}` + admit caster no match |
| Novo Futuro (adrenalina) | consumo 1 a 1 / stacks |
| Loading infinito sala | `roomStartAbort` + clamp neve Bermuda |
| Morte vs DOWNED (BR custom) | não zera PreferTeam; `downOrKill` idempotente |
| Anti-movimento | `AC_MOVE=1` em CS **e** BR (treino isento) |
| Estimativa / fila CS ranked | timeout 0 = só fecha com 8 |

---

## 7. Deploy rápido (VPS game)

```bash
# TCP
chmod +x /root/tcp-server
screen -S tcp -X quit
screen -dmS tcp bash -lc 'cd /root && ./tcp-server >> /root/tcp-server.log 2>&1'

# UDP / match
chmod +x /root/match-server
screen -S match -X quit
screen -dmS match bash -lc 'cd /root && ./match-server 2>&1 | tee -a /root/match-server.log'
```

Conferir:
```bash
screen -ls
ss -tlnp | grep 13891
ss -ulnp | grep 10100
```

---

## 8. Arquivos tocados (mapa)

### Go — TCP (`src/tcp-server-main/`)
- `internal/tcp/dispatch.go`
- `internal/tcp/handlers_matchmaking.go`
- `internal/tcp/matchmaker.go`
- `internal/tcp/group.go`
- `internal/tcp/social.go`
- `internal/store/career_stats.go`

### Go — Match (UDP)
- Binário rebuildado a partir de `match-server/` (sem mudança obrigatória neste pacote para os itens 1–5; use o bin atual se já estiver no ar)

### Python — HTTP (remoto `/root/1.24/htp41.py`)
- `UpdateSocialBasicInfo`
- `GetBattleTag`
- bloco `social_info` do PersonalShow + helpers `_fill_social_tags` / `_parse_update_social_tags`

### Mongo
- `server_config._id=authorization.require_authorization = false`
- campos de conta: `social.battle_tag`, `social.social_tag`, `show_tag_ids`, `battle_tags`

---

## 9. GitHub

Neste workspace **não há** repositório git nem `gh` CLI instalado.  
Os arquivos estão em:

`C:\Users\GHOSTH\Desktop\SERVIDOR\RELEASE_2026-08-21\`

Para publicar:
1. Instalar [GitHub CLI](https://cli.github.com/) ou criar repo no site
2. Subir a pasta `RELEASE_2026-08-21` + fontes que quiser versionar  
   **Não** commitir `.env`, senhas, Mongo URI, tokens

Se mandar a URL do repositório GitHub, dá para inicializar o git e fazer o push na próxima mensagem.

---

## 10. Checklist pós-release

- [ ] Login sem whitelist (conta nova)
- [ ] Estimativa CS Rank ≠ 00:00
- [ ] BR Rank espera ~1:30 / ≥35 (não inicia solo em 12s)
- [ ] Trocar modo no lobby sem “pular” sozinho
- [ ] Equipar etiquetas → aparecem no perfil (Galeria)
- [ ] Partida CS/BR conecta no UDP `:10100`
