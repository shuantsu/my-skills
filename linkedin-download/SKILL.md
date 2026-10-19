---
name: linkedin-download
description: Use when user asks to download LinkedIn video/audio (post, feed update, learning, events) via yt-dlp and attach/send via Telegram/WhatsApp/email. Use ONLY for LinkedIn media download + send tasks.
---

# LinkedIn Download

Download LinkedIn videos with `yt-dlp` (native `linkedin` extractor) and send to the owner's chat.

## Environment

- yt-dlp: `/usr/local/bin/yt-dlp` (LinkedIn extractors: `linkedin`, `linkedin:events`, `linkedin:learning`, `linkedin:learning:course`)
- ffmpeg: `/usr/local/bin/ffmpeg`, static build at `/tmp/ffmpeg-7.0.2-amd64-static/`
- Downloads: `/root/projetos/downloads/`
- Telegram credentials: `/root/.env` (`TELEGRAM_BOT_TOKEN`, `ALLOWED_USERS`); load with `set -a; source /root/.env; set +a` (never print token)
- Bot API limit: 50 MB per file — larger files go via WhatsApp/email
- MCP `yt-dlp` (`@kevinwatt/yt-dlp-mcp`) is configured in `~/.config/opencode/opencode.jsonc` and exposes `ytdlp_download_video` / `ytdlp_download_audio` — use it when loaded; otherwise call yt-dlp directly as below.

## Accepted URL patterns

- `https://www.linkedin.com/posts/<slug>-<id>-<hash>/`  ← most common
- `https://www.linkedin.com/posts/<slug>-ugcPost-<id>-<hash>/`
- `https://www.linkedin.com/feed/update/urn:li:activity:<id>` and `urn:li:ugcPost:<id>`
- `https://www.linkedin.com/learning/...`, `https://www.linkedin.com/events/...`

## Workflow

### 1. Probe first (never guess the title)

```bash
timeout 90 yt-dlp --no-playlist --simulate \
  --print "%(extractor)s | %(title)s | %(duration)ss | %(filesize)s" "<URL>" 2>&1 | tail -4
```

Title is long (LinkedIn posts use the post text). Trim it for the caption. If this fails with a signup/404 redirect, the post needs login — see Cookies.

### 2. Download video (MP4)

```bash
nohup yt-dlp --no-playlist -f "bv*[ext=mp4]+ba[ext=m4a]/b[ext=mp4]/b" \
  --merge-output-format mp4 \
  -o "/root/projetos/downloads/linkedin-%(id)s.%(ext)s" "<URL>" > /tmp/li_dl.log 2>&1 &
sleep 15; tail -c 900 /tmp/li_dl.log; ls -lh /root/projetos/downloads/linkedin-*
```

### 3. Download audio only (MP3)

```bash
FFDIR=/tmp/ffmpeg-7.0.2-amd64-static/
nohup yt-dlp --no-playlist --ffmpeg-location "$FFDIR" -x --audio-format mp3 \
  --audio-quality 0 --embed-metadata --embed-thumbnail --convert-thumbnails jpg \
  -o "/root/projetos/downloads/linkedin-%(id)s.%(ext)s" "<URL>" > /tmp/li_dl.log 2>&1 &
sleep 15; tail -c 900 /tmp/li_dl.log
```

### 4. Send to the chat (see skill `telegram-send`)

```bash
set -a; source /root/.env; set +a
nohup curl -s -m 180 -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendVideo" \
  -F chat_id="${ALLOWED_USERS%%,*}" -F "video=@/root/projetos/downloads/linkedin-<id>.mp4" \
  -F "caption=<titulo curto> ｜ <URL>" > /tmp/tg_video.log 2>&1 &
sleep 15; head -c 500 /tmp/tg_video.log
```

- Audio: endpoint `sendAudio`, field `audio=@<arquivo.mp3>`
- Success: `"ok":true`. On `413 Request Entity Too Large`, fall back to WhatsApp/email.

## Cookies (login-required posts)

Public posts work without auth. Private/member-gated posts redirect to `signup/cold-join`:

```bash
# export cookies from a logged-in browser session once
yt-dlp --cookies /root/linkedin-cookies.txt "<URL>"
# persist for all yt-dlp runs
mkdir -p ~/.config/yt-dlp && printf '%s\n' '--cookies' '/root/linkedin-cookies.txt' > ~/.config/yt-dlp/config
```

Never commit `/root/linkedin-cookies.txt` or print its contents.

## Rules

- Always `--no-playlist` — LinkedIn URLs often carry feed context that can widen the match.
- Always quote URLs; they contain percent-encoded accented characters (`%C3%A7%C3%A3o`).
- Use `nohup + sleep + log check` for downloads and uploads; never block the turn.
- Verify `"ok":true` in the Telegram log before reporting success.
- Name files `linkedin-<post-id>.mp4` to stay unique and sortable.
- Run the probe (step 1) before downloading so the caption has the real title.
