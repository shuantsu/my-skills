---
name: yt-download-whatsapp
description: Use when user asks to download YouTube/other video/audio (yt-dlp) and send via WhatsApp (neonize media). Use ONLY for download-media + WhatsApp tasks.
---

# YT Download + WhatsApp

Download media with `/tmp/yt-dlp_linux`, ensure WhatsApp-compatible codec, send with `/root/whatsapp/envio/send_media.py` (any file type) or `send_video.py` (video only).

## Environment

- yt-dlp: `/tmp/yt-dlp_linux` (if missing: `curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp_linux -o /tmp/yt-dlp_linux && chmod +x /tmp/yt-dlp_linux`)
- ffmpeg static: `ls -d /tmp/ffmpeg-*-static/` → export `PATH="/tmp/ffmpeg-X:$PATH"` (media ops fail without it)
- Sender (any file): `/root/whatsapp/envio/send_media.py <target> <file> [caption] [--ptt]` — auto-routes: video→send_video, audio→send_audio (+caption as separate text), image→send_image, other→send_document. Use this by default.
- Sender (video only): `/root/whatsapp/envio/send_video.py <target> <file.mp4> [caption]`
- Contacts (OFFLINE — never connect while daemon is alive):
  `wa_recent.py` / `wa_names.json` (see `whatsapp-send` skill). The legacy
  `listagem/list_contacts_groups.py` CONNECTS and kicks the daemon — use only if daemon is dead.
- Aliases locais: `/root/whatsapp/aliases.json` (consulte/persista com
  `daemon/wa_alias.py get|set`). Passe o **nome** ao `wa_enqueue.py` / `send_media.py`
  — eles expandem o JID sozinhos. Nunca escreva telefone/JID real aqui.
- Downloads: `/root/projetos/downloads/`

## Daemon guard (MANDATORY)

- Sends go through the OUTBOX QUEUE (offline, never steals the session):

```bash
export PATH="$(ls -d /tmp/ffmpeg-*-static/ | head -1):$PATH"
/root/whatsapp/.venv/bin/python -u /root/whatsapp/daemon/wa_enqueue.py "<alias_ou_jid>" --file "<arquivo>" --caption "<legenda + URL>" 2>&1
```

- Verify: query `outbox` table for `status='done'` after ~15s.
- Direct `send_media.py` / `send_video.py` CONNECT and kick the systemd daemon —
  use ONLY if `systemctl is-active wa_daemon` != active.

## Workflow

### 1. Download audio MP3 (music: ALWAYS `artist - title`)

```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
mkdir -p /root/projetos/downloads
/tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -x --audio-format mp3 --audio-quality 0 --embed-metadata --embed-thumbnail --convert-thumbnails jpg -o "/root/projetos/downloads/%(artist,uploader)s - %(title)s.%(ext)s" "<URL>" 2>&1 | tail -5
```

Music files MUST be named `artist - title` (`%(artist,uploader)s - %(title)s` falls back to uploader when artist is missing). `--embed-metadata` writes ID3 artist/title, `--embed-thumbnail` sets the cover art (requires ffmpeg).

### 1b. Force ID3v2.3 + verify cover (yt-dlp writes v2.4, less compatible)

```bash
export PATH="$(ls -d /tmp/ffmpeg-*-static/ | head -1):$PATH"
ffmpeg -y -i "<ARQUIVO.mp3>" -map 0:a -map 0:v -c copy -id3v2_version 3 -metadata artist="<ARTISTA>" -metadata title="<TITULO>" "/tmp/tagged.mp3" && mv /tmp/tagged.mp3 "<ARQUIVO.mp3>"
ffprobe -v error -show_entries format_tags -of default "<ARQUIVO.mp3>"
ffmpeg -y -i "<ARQUIVO.mp3>" -map 0:v -c copy /tmp/cover_check.jpg 2>&1 | tail -1; ls -lh /tmp/cover_check.jpg
```

### 2. Download video (prefer low-res + compatible)

```bash
FFDIR=$(ls -d /tmp/ffmpeg-*-static/ | head -1)
mkdir -p /root/projetos/downloads
/tmp/yt-dlp_linux --ffmpeg-location "$FFDIR" -f "bv*[height<=240]+ba/b[height<=240]/b" --merge-output-format mp4 -o "/root/projetos/downloads/%(title)s-240p.%(ext)s" "<URL>" 2>&1 | tail -5
```

If `ERROR: This video is not available`, retry with android client (SABR experiment):

```bash
/tmp/yt-dlp_linux --extractor-args "youtube:player_client=android" -F "<URL>" 2>&1 | head -20
/tmp/yt-dlp_linux --extractor-args "youtube:player_client=android" --ffmpeg-location "$FFDIR" -f 18 -o "/root/projetos/downloads/%(title)s-360p.%(ext)s" "<URL>" 2>&1 | tail -5
```

Long downloads: wrap with `nohup ... > /tmp/ytdl.log 2>&1 &` + `sleep 10; cat /tmp/ytdl.log`.

### 2. Ensure WhatsApp plays it (H.264 + AAC)

Check:

```bash
export PATH="/tmp/ffmpeg-7.0.2-amd64-static:$PATH"
ffprobe -v error -show_entries stream=codec_name,width,height -of csv "<file.mp4>"
```

If video is NOT h264 (e.g. av1/vp9) or audio is NOT aac (e.g. opus), transcode:

```bash
ffmpeg -y -i "<in.mp4>" -c:v libx264 -profile:v baseline -level 3.0 -pix_fmt yuv420p -c:a aac -b:a 128k -movflags +faststart "<out-wa.mp4>" 2>&1 | tail -3
```

Format 18 (avc1+mp4a) usually needs no transcode.

### 3. Send (any file type via send_media.py)

```bash
export PATH="/tmp/ffmpeg-7.0.2-amd64-static:$PATH"
timeout 120 /root/whatsapp/.venv/bin/python -u /root/whatsapp/envio/send_media.py "<alias_ou_jid>" "<file>" "<caption + URL>" 2>&1 | tail -5
```

Success: `Midia enviada com sucesso para <alias_ou_jid>!`. On `write: connection reset by peer` during upload, retry once. For voice-note style audio add `--ptt`.

## Rules

- Always quote paths with spaces; always set ffmpeg on PATH before send.
- Never run downloads/uploads in blocking foreground loops; use nohup/timeout + log tail.
- Do NOT edit `send_video.py`; call it. See `whatsapp-send` skill for text-only fallback.
