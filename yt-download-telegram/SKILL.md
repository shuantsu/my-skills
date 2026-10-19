---
name: yt-download-telegram
description: Use when user asks to download YouTube/other video/audio (yt-dlp) and attach/send via Telegram (Bot API) to the owner chat. Use ONLY for download-media + Telegram tasks.
---

# YT Download + Telegram

Download media with `/tmp/yt-dlp_linux` and send to the owner's Telegram chat via Bot API (see skill `telegram-send`).

## Environment

- yt-dlp: `/tmp/yt-dlp_linux` (if missing: `curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp_linux -o /tmp/yt-dlp_linux && chmod +x /tmp/yt-dlp_linux`)
- ffmpeg static: `ls -d /tmp/ffmpeg-*-static/` → pass `--ffmpeg-location` to yt-dlp AND export `PATH` for sends
- Credentials: `/root/.env` (`TELEGRAM_BOT_TOKEN`, `ALLOWED_USERS`); load with `set -a; source /root/.env; set +a` (never print token)
- Default chat: `${ALLOWED_USERS%%,*}`
- Bot API limit: 50 MB per file — larger files go via WhatsApp/email instead
- Downloads: `/root/projetos/downloads/`
- System python is 3.9 — always use `/tmp/yt-dlp_linux`, never the repo in `/root/projetos/yt-dlp`

## Workflow

### 1. Download MP3 (music: ALWAYS `artist - title` + ID3 + cover)

```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
mkdir -p /root/projetos/downloads
/tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -x --audio-format mp3 --audio-quality 0 --embed-metadata --embed-thumbnail --convert-thumbnails jpg -o "/root/projetos/downloads/%(artist,uploader)s - %(title)s.%(ext)s" "<URL>" 2>&1 | tail -5
```

`--embed-metadata` writes ID3 artist/title, `--embed-thumbnail` sets cover art.

### 1b. Force ID3v2.3 + verify cover (yt-dlp writes v2.4, less compatible)

```bash
export PATH="$(ls -d /tmp/ffmpeg-*-static/ | head -1):$PATH"
ffmpeg -y -i "<ARQUIVO.mp3>" -map 0:a -map 0:v -c copy -id3v2_version 3 -metadata artist="<ARTISTA>" -metadata title="<TITULO>" "/tmp/tagged.mp3" && mv /tmp/tagged.mp3 "<ARQUIVO.mp3>"
ffprobe -v error -show_entries format_tags -of default "<ARQUIVO.mp3>"
ffmpeg -y -i "<ARQUIVO.mp3>" -map 0:v -c copy /tmp/cover_check.jpg 2>&1 | tail -1; ls -lh /tmp/cover_check.jpg
```

### 2. Download video (low-res default, WhatsApp/Telegram-friendly)

```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
/tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -f "bv*[height<=480]+ba/b[height<=480]/b" --merge-output-format mp4 -o "/root/projetos/downloads/%(title)s-%(height)s.%(ext)s" "<URL>" 2>&1 | tail -5
```

If `ERROR: This video is not available`, retry with android client (SABR experiment):

```bash
/tmp/yt-dlp_linux --extractor-args "youtube:player_client=android" -F "<URL>" 2>&1 | head -20
/tmp/yt-dlp_linux --extractor-args "youtube:player_client=android" --ffmpeg-location "$FFDIR" -f 18 -o "/root/projetos/downloads/%(title)s-360p.%(ext)s" "<URL>" 2>&1 | tail -5
```

If video codec is not h264 (av1/vp9), Telegram still plays most MP4s — no transcode needed (unlike WhatsApp).

Long downloads: `nohup ... > /tmp/ytdl.log 2>&1 &` + `sleep 10; tail -c 1000 /tmp/ytdl.log`.

### 3. Send to Telegram (audio/video/document)

```bash
set -a; source /root/.env; set +a
nohup curl -s -m 120 -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendAudio" -F chat_id="${ALLOWED_USERS%%,*}" -F "audio=@<ARQUIVO.mp3>" -F "caption=<artista - titulo + URL>" > /tmp/tg_audio.log 2>&1 &
sleep 15; head -c 500 /tmp/tg_audio.log
```

- Video: endpoint `sendVideo`, field `video=@<ARQUIVO.mp4>`
- Other files (.txt, .zip): endpoint `sendDocument`, field `document=@<ARQUIVO>`
- Success: `{"ok":true,"result":{"message_id":...`

## Rules

- NEVER print `TELEGRAM_BOT_TOKEN`; refer to it as `<TOKEN>`.
- Always quote paths (spaces, `｜`, accents common in titles).
- Use `nohup + sleep + log check` for downloads and uploads; never block the turn.
- Verify `"ok":true`; on `413 Request Entity Too Large` fall back to WhatsApp/email.
- MP3 music: always `artist - title` filename + `--embed-metadata --embed-thumbnail`.
