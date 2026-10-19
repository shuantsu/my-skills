---
name: email-schedule
description: Use when scheduling/delaying emails via himalaya v2 CLI + at daemon (atq, atrm). Use ONLY for schedule-at, list-queue, cancel-scheduled tasks.
---

# Email Schedule

Schedule emails with `himalaya` v2 + `at` (`atd` already running). Himalaya has no native scheduler — persist the message to a file and queue a one-shot `at` job.

## Environment

- Binary: `/root/.local/bin/himalaya` (v2.x), config `/root/.config/himalaya/config.toml`
- Dados da conta: `/root/email_credentials.txt` (`key=value`, **fora do repo**, chmod 600) —
  campos `account`, `email`, `display_name`, `imap_server`, `smtp_server`. Nunca os cite na doc/chat.
- Helper: `email_field() { grep -E "^$1=" /root/email_credentials.txt | head -1 | cut -d= -f2-; }`
- Scheduler: `at` / `atd` (systemd `atd.service` active). Syntax: `echo "<cmd>" | at HH:MM [today|tomorrow|YYYY-MM-DD]`
- Queue dir: `/tmp/` — eml files `/tmp/email-<id>.eml`, logs `/tmp/himalaya-<id>.log`

## Workflow

### 1. Create the .eml file (headers define envelope, blank line before body)

```bash
email_field() { grep -E "^$1=" /root/email_credentials.txt | head -1 | cut -d= -f2-; }
FROM=$(email_field email)
printf 'From: %s\nTo: destino@example.com\nSubject: assunto\n\ncorpo\n' "$FROM" > /tmp/email-1640.eml
cat /tmp/email-1640.eml
```

For multiple recipients add `Cc:`/`Bcc:` headers. Use `printf` with `\n`, never bare `echo`.

### 2. Schedule with at (preferred)

```bash
echo "/root/.local/bin/himalaya message send --no-save < /tmp/email-1640.eml > /tmp/himalaya-1640.log 2>&1" | at 16:40 today 2>&1
```

Variants: `at 16:40 tomorrow`, `at 09:00 2026-10-13`, `at now + 30 minutes`. Always use absolute binary path (`/root/.local/bin/himalaya`) since `at` runs under `/bin/sh` with minimal PATH.

### 3. Verify / manage queue

```bash
atq 2>&1
at -c <jobid> 2>&1 | tail -n 20
atrm <jobid> 2>&1
```

`atq` output `1  Mon Oct 12 16:40:00 2026 a root` = job 1 queued. Check send result in `/tmp/himalaya-<id>.log` after run time (expect `Message successfully sent`).

### 4. Fallback without at (nohup + sleep, less robust)

Only if `atd` is down:
```bash
nohup bash -c 'sleep $(( $(date -d "today 16:40" +%s) - $(date +%s) )); /root/.local/bin/himalaya message send --no-save < /tmp/email-1640.eml > /tmp/himalaya-1640.log 2>&1' > /tmp/sched.log 2>&1 &
pgrep -af "email-1640" 2>&1
```

## Rules

- Do NOT run `sleep`/`at` in foreground; `at` returns immediately, `nohup ... &` for fallback.
- Pass `--no-save` unless user wants Sent copy.
- Never print the SMTP/IMAP password from config; refer as `***`.
- Never copy `email`/`account`/hostnames from `/root/email_credentials.txt` into docs, examples or chat.
- Confirm `date` / `date -d "<time>"` shows future time before queueing; if past, use `tomorrow`.
- Report back: job id, scheduled time (`atq`), eml path, log path.
