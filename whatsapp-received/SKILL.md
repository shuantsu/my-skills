---
name: whatsapp-received
description: Use when listing received WhatsApp messages (recebidas, últimas mensagens recebidas, mensagens que recebi). Use ONLY for instant offline reads of received messages via wa_received.py in /root/whatsapp.
---

# WhatsApp Received

Lista as últimas N mensagens **recebidas** (`is_from_me=0`) via SQLite local. Nunca conecta no WhatsApp na consulta.

Arquitetura (`/root/whatsapp/`):
- `daemon/wa_daemon.py` — daemon persistente, conecta 1 vez, grava `MessageEv` + `HistorySyncEv` + `OfflineSync` em `/root/whatsapp/wa_messages.db`. Tem `migrate_all()` e roda migração no boot (`--migrate-once` só migra e sai).
- `daemon/wa_store.py` — helpers (schema, `normalize_ts`, `upsert`, `display_name`, `load_names`).
- `daemon/wa_received.py` — consulta **só o SQLite**, filtra só recebidas, resolve nome via `wa_names.json` + `whatsmeow_lid_map` (LID→telefone). É o único script a usar para mensagens recebidas.
- NÃO usar `listagem/list_recent_messages.py` (lento, 35-50s por conexão). Para últimos chats mistos (enviadas+recebidas) use a skill `whatsapp-instant` (`daemon/wa_recent.py`).

## Environment

- Workdir: `/root/whatsapp`
- Interpreter: `/root/whatsapp/.venv/bin/python`
- System python (`/usr/bin/python3`) NÃO tem `neonize`. Sempre use o venv python.
- Sempre rode com `-u` + `2>&1`.
- DB: `/root/whatsapp/wa_messages.db` (tabela `messages`).
- Nomes: `/root/whatsapp/wa_names.json` (`contacts` + `groups`).
- LID map: `/root/whatsapp/whatsapp_session.db` (tabela `whatsmeow_lid_map`, LID → telefone).
- Daemon log: `/tmp/opencode/wa_daemon.log`, pid: `/tmp/opencode/wa_daemon.pid`.

## Usage

```bash
# 15 últimas recebidas (padrão)
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_received.py 15 2>&1

# 50 últimas recebidas
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_received.py 50 2>&1

# JSON para pós-processar
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_received.py 50 --format json 2>&1

# filtro por chat/nome/pushname (resolve nome via LID)
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_received.py 15 --chat <nome_ou_jid> 2>&1
```

Sem `timeout` — resposta em ~0.2s. Não adicione `timeout 45`.

## Opções (`wa_received.py`)

| Flag | Efeito |
|---|---|
| `N` (posicional) | quantidade (default 15) |
| `--format text\|json` | saída (default text) |
| `--chat TEXTO` | filtra por substring de chat/nome/pushname (pós-resolução LID) |
| `--include-status` | inclui `status@broadcast` (default exclui) |
| `--include-newsletters` | inclui `@newsletter` (default exclui) |
| `--include-noise` | inclui `<sem texto...>` (default exclui) |

Sucesso parece com:

```text
1. [11/10 16:17] contato_exemplo (000000000000000@lid) | contato_exemplo: exemplo de mensagem recebida
```

Nota: se pedir 50 mas só existirem 34 com texto no cache, retorna 34. O resto está como status/ruído/mídia sem texto — use `--include-status --include-noise` para completar.

## Rules

- Do NOT `pip install`, do NOT editar `wa_*.py` salvo se pedido.
- Para mensagens RECEBIDAS use SEMPRE `daemon/wa_received.py`. Nunca reconecte, nunca use `listagem/list_recent_messages.py`.
- Sempre `-u` + `2>&1`. Nunca `timeout 40-50` na consulta.
- Responda em português com data (`dd/mm HH:MM`), nome resolvido, JID entre parênteses e texto.
