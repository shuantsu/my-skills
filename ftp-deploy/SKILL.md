---
name: ftp-deploy
description: Use when deploying/sending project folders to FTP (init config, dry-run, deploy, fix 404/path issues). Use ONLY for FTP deploy tasks with ftp_deploy.py.
---

# FTP Deploy

Deploy project folders to FTP using `ftp_deploy.py` (stdlib-only, incremental via `.ftp_cache.json` with mtime + MD5).

## Environment

- Script: `/usr/local/lib/node_modules/@artickc/opencode-telegram-bot/ftp_deploy/ftp_deploy.py`
- Invoke: `python3 /usr/local/lib/node_modules/@artickc/opencode-telegram-bot/ftp_deploy/ftp_deploy.py <flags>` with `workdir` = project root
- Credenciais: `/root/ftp_credentials.txt` — `key=value`, campos `host`, `user`,
  `password`, `origin`, `ignore`, `site`. **Fora de qualquer repo** (gitignorado), chmod 600.
- Helper (leia os campos daqui; **nunca** os copie para SKILL.md, exemplos ou chat):

```bash
ftp_field() { grep -E "^$1=" /root/ftp_credentials.txt | head -1 | cut -d= -f2-; }
```

- FTP root login lands in `/www` (the web root) — confirme o campo `host` no arquivo.

## Workflow

### 1. Init config (once per project)

```bash
python3 /usr/local/lib/node_modules/@artickc/opencode-telegram-bot/ftp_deploy/ftp_deploy.py --init
```

Creates `.ftprules` in the project root. Edit it, preenchendo cada campo com o
helper (`ftp_field host`, `ftp_field user`, `ftp_field password`, `ftp_field origin`,
`ftp_field ignore`) — nenhum valor é citado aqui:

```ini
[remote]
<host>          # ftp_field host

[user]
<user>          # ftp_field user

[password]
<password>      # ftp_field password

[remote-folder]
www/new/<project>

[origin]
<origin>        # ftp_field origin

[ignore]
<ignore>        # ftp_field ignore
```

### 2. Deployable folder

- The deployable files must live in the `origin` folder (convention: `dist/` with `index.html`, assets).
- If the project files are at the project root, create `dist/` and move them there first.

### 3. Preview then deploy (always in this order)

```bash
python3 .../ftp_deploy.py --dry-run
python3 .../ftp_deploy.py
```

Other flags: `--force` (re-upload everything), `--open-config-folder` (edit template).

### 4. Verify

- Redeploy output ends with `Deploy completed to <host>/<remote-folder>!`
- Check via FTP listing and HTTP:

```bash
ftp_field() { grep -E "^$1=" /root/ftp_credentials.txt | head -1 | cut -d= -f2-; }
curl -s -o /dev/null -w '%{http_code}' --max-time 10 "$(ftp_field site)/new/<project>/"
```

Expect `200`.

## Rules

- `remote-folder` MUST start with `www/` (e.g. `www/new/<project>`). The script runs `cwd('/')` before creating dirs, so a bare `new/<project>` is created at filesystem `/new/<project>` — outside the web root — and the site returns 404 even though the upload "succeeds".
- If a deploy went to the wrong path (`/new/...` instead of `/www/new/...`), fix `.ftprules`, delete `.ftp_cache.json` in the project root (it only tracks local files, safe to remove), and redeploy with `--force` semantics (cache wipe forces full re-upload).
- Diagnose remote paths with a short `ftplib` listing before guessing:

```bash
python3 -c "
import ftplib
def f(k):
    for line in open('/root/ftp_credentials.txt'):
        line = line.strip()
        if line.startswith(k + '='):
            return line.split('=', 1)[1].strip()
ftp = ftplib.FTP(f('host'))
ftp.login(f('user'), f('password'))
ftp.cwd('/www/new/<project>')
print('PWD:', ftp.pwd())
ftp.retrlines('LIST')
ftp.quit()
"
```

- Each `workdir` run of bash is one-shot with a timeout; never run watchers/servers in the foreground (use `nohup ... &` + a single `curl` check for previews).
- Do NOT commit `.ftprules` or `.ftp_cache.json` (they contain a plaintext password / local state).
- Do NOT print the password into chat output; read it from `/root/ftp_credentials.txt` silently when needed.
- Do NOT copy any value from `/root/ftp_credentials.txt` (host, user, password, origin, site)
  into SKILL.md, examples, commit messages or chat — always call `ftp_field` at runtime.
- Never `cat` `/root/ftp_credentials.txt` into chat output.
