---
name: email-send
description: Use when sending emails via himalaya v2 CLI (message send, message compose --send, smtp send) or checking email accounts. Use ONLY for email send/check tasks.
---

# Email Send

Send emails using `himalaya` v2 CLI. Never downgrade config to v1 format, never hardcode passwords in commands (they live in the config file).

## Environment

- Binary: `himalaya` (`/root/.local/bin/himalaya`, v2.x)
- Config: `/root/.config/himalaya/config.toml` (v2 format) — guarda a senha em **texto plano**;
  nunca committar e nunca copie o conteúdo para documentação ou chat.
- Dados da conta (`account`, `email`, `display_name`, `imap_server`, `smtp_server`):
  `/root/email_credentials.txt` — `key=value`, **fora de qualquer repo** (gitignorado), chmod 600.
  Leia de lá quando precisar; **nunca** os cite em SKILL.md, exemplos ou saída de chat.
- Override config path with `himalaya -c <PATH> ...` or `HIMALAYA_CONFIG=<PATH>`.

## Reading account data (helper — defina uma vez por shell)

```bash
email_field() { grep -E "^$1=" /root/email_credentials.txt | head -1 | cut -d= -f2-; }
```

## v2 config shape (reference — valores vêm de `/root/email_credentials.txt`)

```toml
[accounts.<account>]      # email_field account
default = true
email = "<email>"         # email_field email
display-name = "<display_name>"

imap.server = "<imap_server>"
imap.sasl.plain.username = "<email>"
imap.sasl.plain.password.raw = "***"

smtp.server = "<smtp_server>"
smtp.sasl.plain.username = "<email>"
smtp.sasl.plain.password.raw = "***"

mailbox.alias.inbox = "INBOX"
mailbox.alias.sent = "Sent"
mailbox.alias.drafts = "Drafts"
mailbox.alias.trash = "Trash"
```

TLS note: o hostname alternativo do seu domínio costuma apontar para o VIP do provedor e
**falha na verificação do certificado** (o cert cobre só o curinga do provedor). Use
sempre os hostnames gravados em `imap_server` / `smtp_server`.

## Workflow

### 1. Validate account (when send fails or config changed)

```bash
himalaya account check 2>&1
himalaya account list 2>&1
```

Healthy output:

```text
Account: <account>
  imap: OK
  smtp: OK
```

`FAIL (invalid peer certificate)` = wrong server hostname — use exactly the
`imap_server` / `smtp_server` recorded in `/root/email_credentials.txt`.
`configures no supported backend ... v1 backend table is ignored` = config still in v1 format, migrate to `imap.server` / `smtp.server` + `*.sasl.plain.*`.

### 2. Send (preferred: shared `message send`)

Headers (`From:`, `To:`, `Subject:`) define envelope; blank line separates body:

```bash
FROM=$(email_field email)
printf 'From: %s\nTo: destino@example.com\nSubject: assunto\n\ncorpo da mensagem\n' "$FROM" | himalaya message send --no-save 2>&1
```

Success output:

```text
Message successfully sent
```

### 3. Send (alternative: `message compose --send`)

```bash
himalaya message compose --from "$(email_field email)" -t destino@example.com -s "assunto" --body "corpo" --send --no-save 2>&1
```

Useful flags: `--cc`, `--bcc`, `--body-file <PATH>`, `--attach <PATH>`.

### 4. Send (raw SMTP, explicit envelope)

```bash
FROM=$(email_field email)
printf 'From: %s\nTo: destino@example.com\nSubject: assunto\n\ncorpo\n' "$FROM" | himalaya smtp send -f "$FROM" -t destino@example.com 2>&1
```

Repeat `-t` for multiple recipients.

## Rules

- Do NOT `pip install`, do NOT create wrapper sender scripts; call `himalaya` directly.
- Prefer `message send` (shared API); use `smtp send` only when explicit `MAIL FROM` / `RCPT TO` control is needed.
- Pass `--no-save` unless user asks to keep a copy in Sent (avoids IMAP append on every send).
- Quote addresses/subjects; use `printf` with `\n` headers, never `echo` without proper escaping.
- Do NOT print or copy the password from the config into chat output; refer to it as `***`.
- Do NOT copy `email`/`account`/`imap_server`/`smtp_server` from `/root/email_credentials.txt`
  into SKILL.md, examples, commit messages or chat — call `email_field` at runtime.
- Never `cat` `/root/.config/himalaya/config.toml` or `/root/email_credentials.txt` into chat output.
- If `account check` fails, fix config first — do not retry send blindly.
