---
name: whatsapp-send
description: Use when sending WhatsApp messages via send_whatsapp.py, send_wa.py, or finding contacts/groups with list_chats.py and list_contacts_groups.py. Use ONLY for WhatsApp send/list tasks in /root/whatsapp.
---

# WhatsApp Send

Send WhatsApp messages using the existing scripts in `/root/whatsapp`. Never modify or create sender scripts.

## Environment

- Workdir: `/root/whatsapp`
- Interpreter: `/root/whatsapp/.venv/bin/python`
- System python (`/usr/bin/python3`, 3.9) does NOT have `neonize`. Always use the venv python.
- Always run with `-u` (unbuffered). Scripts call `os._exit(0)`, which drops buffered stdout.
- Session: `whatsapp_session.db` in `/root/whatsapp`. Already authenticated.
- Scripts use absolute session path (`/root/whatsapp/whatsapp_session.db`), so they work from any cwd.

## Scripts

| Script | Purpose | Usage |
|---|---|---|
| `envio/send_whatsapp.py` | Send to contact or group (preferred) | `python3.14 -u /root/whatsapp/envio/send_whatsapp.py "<alias_ou_jid>" "<text>"` |
| `envio/send_wa.py` | Send to individual number only (`s.whatsapp.net`) | `python3.14 -u /root/whatsapp/envio/send_wa.py <numero> "<text>"` |
| `listagem/list_contacts_groups.py` | List local contacts (SQLite, no connection) + groups (needs connection). Buggy with filter for groups. | `python3.14 -u /root/whatsapp/listagem/list_contacts_groups.py [filtro]` |
| `listagem/list_chats.py` | List groups + contacts by query | `python3.14 -u /root/whatsapp/listagem/list_chats.py "[query]"` |
| `listagem/list_whatsapp.py` | Dump all groups + contacts | `python3.14 -u /root/whatsapp/listagem/list_whatsapp.py` |

## JID formats

- Contact: `<numero>` or `<numero>@s.whatsapp.net`
- Group: `<id_do_grupo>@g.us` (numeric ID + `@g.us`)
- `send_whatsapp.py` accepts bare numbers, full JIDs, group JIDs **or an alias name**
  from `/root/whatsapp/aliases.json`. It auto-maps `-` or `120363...` prefix to `g.us`.

## Sending (PREFERRED — outbox queue, never connects)

Enqueue via `daemon/wa_enqueue.py` (offline SQLite; daemon sends in ~5s on its own
connection — zero session steal):

```bash
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_enqueue.py "<alias_ou_jid>" --text "mensagem" 2>&1
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_enqueue.py "<alias_ou_jid>" --file "/caminho/arquivo" --caption "legenda" 2>&1
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_enqueue.py "<alias_ou_jid>" --file "/caminho/audio.ogg" --caption "leg" --ptt 2>&1
```

Check status (offline):

```bash
/root/whatsapp/.venv/bin/python -u -c "import sqlite3;db=sqlite3.connect('/root/whatsapp/wa_messages.db');print(db.execute('SELECT id,target,status,error FROM outbox ORDER BY id DESC LIMIT 5').fetchall())" 2>&1
```

Direct `envio/send_*.py` scripts CONNECT and kick the systemd daemon — use ONLY if
the service is down (`systemctl is-active wa_daemon` != active), then it self-heals
via `Restart=always`.

## Workflow

### 0. Aliases — sempre antes de enviar (telefone/JID fica só no arquivo)

Alvos vivem em `/root/whatsapp/aliases.json` (gitignorado, `0600`). Antes de enviar,
BUSQUE o nome aí; se ainda não existir, PERSISTA. Nunca escreva telefone ou JID real
na documentação nem no chat quando existir um alias:

```bash
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_alias.py get <nome>
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_alias.py set <nome> <jid>
```

Todos os emissores (`send_whatsapp.py`, `send_media.py`, `send_video.py`,
`wa_enqueue.py`) expandem o alias sozinhos — passe o **nome**, não o JID.

### 1. Resolve target JID (OFFLINE ONLY — never connect)

```bash
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_recent.py 30 --per-chat 1 --chat "NomeDoGrupo" 2>&1 | head -20
grep -i "nomedogrupo" /root/whatsapp/wa_names.json | head -5
```

Legacy `listagem/list_contacts_groups.py` dump (CONNECTS — use ONLY if daemon is dead,
then restart daemon immediately after):
```bash
timeout 30 /root/whatsapp/.venv/bin/python -u /root/whatsapp/listagem/list_contacts_groups.py > /tmp/opencode/groups.txt 2>&1
```

Group JID spans multiple lines (`User: "120363..."` + `Server: "g.us"`). Read context around the match:

```bash
sed -n '1825,1870p' /tmp/opencode/groups.txt
```

Combine as `<User>@g.us`.

### 2. Send

```bash
timeout 30 /root/whatsapp/.venv/bin/python -u /root/whatsapp/envio/send_whatsapp.py "<alias_ou_jid>" "hello world" 2>&1
```

Success looks like:

```text
[whatsmeow.Client INFO] - Successfully authenticated
Conectado ao WhatsApp! Enviando mensagem...
Mensagem enviada com sucesso para <alias_ou_jid>!
```

### 3. Rules

- Do NOT `pip install`, do NOT edit scripts, do NOT write new sender scripts.
- Never paste a real phone number or group JID into docs, examples or chat output —
  use the alias name and let the sender expand it.
- Check `aliases.json` before assuming a target; persist new targets with `wa_alias.py set`.
- Keep `timeout 25-30` on every call; `client.connect()` blocks.
- Quote alias/JID and text arguments.
- For individuals prefer `envio/send_whatsapp.py`; use `envio/send_wa.py` only for plain phone numbers.
- After ANY send while daemon is alive: wait 20s, verify daemon reconnected
  (`pgrep -f "daemon/wa_daemon"` + fresh lines in `/tmp/opencode/wa_daemon.log`).
  A send that reports `StreamReplaced`/`connection reset` means the session was
  stolen — wait 30s and retry the send once.
