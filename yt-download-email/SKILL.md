---
name: yt-download-email
description: Use when user asks to download video/audio from YouTube or other sites (yt-dlp) and send by email. Use ONLY for download-video/audio + email tasks.
---

# YT Download + Email

Download video/audio with yt-dlp standalone binary and send via himalaya v2.

## Environment

- yt-dlp binary: `/tmp/yt-dlp_linux` (v2026.08.19+, bundles Python, works on Python 3.9 systems)
  - If missing: `curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp_linux -o /tmp/yt-dlp_linux && chmod +x /tmp/yt-dlp_linux`
  - Check: `/tmp/yt-dlp_linux --version`
- ffmpeg static: `/tmp/ffmpeg-7.0.2-amd64-static/` (contains `ffmpeg` + `ffprobe`)
  - If missing: `curl -L https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz -o /tmp/ffmpeg.tar.xz && tar -xf /tmp/ffmpeg.tar.xz -C /tmp/`
  - Detect dir: `ls -d /tmp/ffmpeg-*-static/`
- Downloads dir: `/root/projetos/downloads/` (mkdir -p if missing)
- Email: `/root/.local/bin/himalaya` (see skill `email-send`). Conta/remetente em
  `/root/email_credentials.txt` (campo `email`, **fora do repo**) — nunca cite o valor na doc/chat.
- System python is 3.9 — DO NOT use `/root/projetos/yt-dlp` repo directly (requires 3.10+). Always use `/tmp/yt-dlp_linux`.

## Workflow

### 1. Download MP3 (audio)

```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
mkdir -p /root/projetos/downloads
/tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -x --audio-format mp3 --audio-quality 0 --embed-metadata --embed-thumbnail --convert-thumbnails jpg -o "/root/projetos/downloads/%(artist,uploader)s - %(title)s.%(ext)s" "<URL>" 2>&1 | tail -20
```

Music files MUST be named `artist - title` (`%(artist,uploader)s - %(title)s` falls back to uploader when artist is missing). `--embed-metadata` writes ID3 artist/title, `--embed-thumbnail` sets the cover art (requires ffmpeg).

### 1b. Force ID3v2.3 + verify cover (yt-dlp writes v2.4, less compatible)

```bash
export PATH="$(ls -d /tmp/ffmpeg-*-static/ | head -1):$PATH"
ffmpeg -y -i "<ARQUIVO.mp3>" -map 0:a -map 0:v -c copy -id3v2_version 3 -metadata artist="<ARTISTA>" -metadata title="<TITULO>" "/tmp/tagged.mp3" && mv /tmp/tagged.mp3 "<ARQUIVO.mp3>"
ffprobe -v error -show_entries format_tags -of default "<ARQUIVO.mp3>"
ffmpeg -y -i "<ARQUIVO.mp3>" -map 0:v -c copy /tmp/cover_check.jpg 2>&1 | tail -1; ls -lh /tmp/cover_check.jpg
```

Cover check must extract a valid JPG; if it fails, re-download thumbnail (`/tmp/yt-dlp_linux --write-thumbnail --skip-download --convert-thumbnails jpg -o /tmp/cover.%(ext)s "<URL>"`) and embed via `-i cover.jpg -map 0 -map 1 -c copy -c:v mjpeg -disposition:v attached_pic`.

For long downloads, use nohup:
```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
nohup /tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -x --audio-format mp3 --embed-metadata --embed-thumbnail --convert-thumbnails jpg -o "/root/projetos/downloads/%(artist,uploader)s - %(title)s.%(ext)s" "<URL>" > /tmp/ytdl.log 2>&1 &
sleep 10; cat /tmp/ytdl.log | head -40
```

### 2. Download MP4 (video, best <=1080p)

```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
/tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -f "bv*[height<=1080]+ba/b[height<=1080]/b" --merge-output-format mp4 -o "/root/projetos/downloads/%(title)s.%(ext)s" "<URL>" 2>&1 | tail -20
```

### 3. Send by email

```bash
email_field() { grep -E "^$1=" /root/email_credentials.txt | head -1 | cut -d= -f2-; }
/root/.local/bin/himalaya message compose --from "$(email_field email)" -t "<destino>" -s "Seu MP3/MP4 - <titulo>" --body "Segue em anexo: <URL>" --attach "<CAMINHO_ARQUIVO>" --send --no-save 2>&1
```

Quote paths with spaces. Verify: `ls -lh /root/projetos/downloads/`.

## Rules

- Always pass `--ffmpeg-location` (else MP3 conversion fails with `ffprobe and ffmpeg not found`).
- Always pass `--no-save` on himalaya send unless user asks to keep copy.
- Never run yt-dlp foreground with infinite wait; use nohup + short sleep + log check for large files.
- Prefer non-interactive flags; never hardcode passwords.
