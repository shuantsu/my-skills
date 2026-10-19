---
name: telegram-send
description: Use when user asks to send Telegram message/audio/video/document via CLI (Bot API) to the owner chat. Use ONLY for Telegram send tasks.
---

# Telegram Send

Send messages and files to the owner's Telegram chat via Bot API with curl. No bot CLI send command exists (`opencode-tg` only has run/setup/install/status/logs) — use the API directly.

## Environment

- Credentials: `/root/.env` (`TELEGRAM_BOT_TOKEN`, `ALLOWED_USERS`)
- Load without printing secrets: `set -a; source /root/.env; set +a`
- Default chat: first entry of `ALLOWED_USERS` → `${ALLOWED_USERS%%,*}`
- Bot API base: `https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}`
- Limits: Bot API files up to 50 MB. Larger files: send via WhatsApp/email instead.

## Workflow

### 1. Text message

```bash
set -a; source /root/.env; set +a
curl -s -m 30 -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" -d chat_id="${ALLOWED_USERS%%,*}" -d text="sua mensagem" 2>&1 | head -c 500
```

Success: `{"ok":true,"result":{"message_id":...`.

### 2. Audio (MP3)

```bash
set -a; source /root/.env; set +a
nohup curl -s -m 120 -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendAudio" -F chat_id="${ALLOWED_USERS%%,*}" -F "audio=@/caminho/arquivo.mp3" -F "caption=legenda + URL" > /tmp/tg_audio.log 2>&1 &
sleep 15; head -c 500 /tmp/tg_audio.log
```

### 3. Video

```bash
set -a; source /root/.env; set +a
nohup curl -s -m 120 -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendVideo" -F chat_id="${ALLOWED_USERS%%,*}" -F "video=@/caminho/arquivo.mp4" -F "caption=legenda + URL" > /tmp/tg_video.log 2>&1 &
sleep 15; head -c 500 /tmp/tg_video.log
```

### 4. Generic document (any file)

```bash
set -a; source /root/.env; set +a
nohup curl -s -m 120 -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument" -F chat_id="${ALLOWED_USERS%%,*}" -F "document=@/caminho/arquivo" -F "caption=legenda" > /tmp/tg_doc.log 2>&1 &
sleep 15; head -c 500 /tmp/tg_doc.log
```

## Rules

- NEVER print `TELEGRAM_BOT_TOKEN` in chat output; refer to it as `<TOKEN>`.
- Always quote file paths (may contain spaces, `｜`, accents).
- Use `nohup + sleep + log check` for file uploads (blocking foreground curl can stall the turn).
- Verify `"ok":true` in the log; on `413 Request Entity Too Large` fall back to WhatsApp/email.
