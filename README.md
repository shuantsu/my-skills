# my-skills

Skills do [opencode](https://opencode.ai) para automação de mensagens, e-mail, mídia e deploy.

Cada pasta contém um `SKILL.md` com frontmatter (`name` + `description`) e o procedimento
completo que o agente segue. As skills assumem um servidor Linux com os binários e scripts
auxiliares já instalados.

## Instalação

```bash
git clone https://github.com/shuantsu/my-skills.git ~/.config/opencode/skills
```

Ou copie apenas as pastas desejadas para `~/.config/opencode/skills/`:

```bash
cp -r my-skills/email-send ~/.config/opencode/skills/
```

Reinicie o opencode para que as skills sejam carregadas.

## Skills

| Skill | O que faz | Depende de |
|---|---|---|
| `email-send` | Envia e consulta e-mails via himalaya v2 | `himalaya`, `~/.config/himalaya/config.toml` |
| `email-schedule` | Agenda e-mail para uma hora futura | `himalaya`, `at`/`atd` |
| `ftp-deploy` | Deploy incremental de pastas via FTP | `ftp_deploy.py` |
| `telegram-send` | Envia mensagem/áudio/vídeo/documento no Telegram | Bot API + `TELEGRAM_BOT_TOKEN` |
| `whatsapp-send` | Envia mensagens, mídia e lista contatos/grupos | `neonize` + sessão autenticada |
| `whatsapp-received` | Lê mensagens recebidas do SQLite local (offline) | `wa_messages.db` + daemon |
| `yt-download-email` | Baixa mídia (yt-dlp) e envia por e-mail | `yt-dlp`, `himalaya` |
| `yt-download-telegram` | Baixa mídia (yt-dlp) e envia no Telegram | `yt-dlp`, Bot API |
| `yt-download-whatsapp` | Baixa mídia (yt-dlp) e envia no WhatsApp | `yt-dlp`, `neonize` |
| `linkedin-download` | Baixa vídeo do LinkedIn e envia no chat | `yt-dlp` |

## Credenciais

Nenhuma credencial fica neste repositório. As skills leem segredos de arquivos
**fora** da árvore do repo, com permissão `600`:

| Arquivo | Campos |
|---|---|
| `/root/email_credentials.txt` | `account`, `email`, `display_name`, `imap_server`, `smtp_server` |
| `/root/ftp_credentials.txt` | `host`, `user`, `password`, `origin`, `ignore`, `site` |
| `/root/.env` | `TELEGRAM_BOT_TOKEN`, `ALLOWED_USERS` |
| `/root/.config/himalaya/config.toml` | config do himalaya (contém senha) |
| `/root/whatsapp/aliases.json` | apelidos → JID (gitignorado) |

Formato dos arquivos `*_credentials.txt`: uma linha por campo, `chave=valor`, ex.:

```ini
host=ftp.exemplo.com
user=usuario
password=***
site=https://exemplo.com
```

Para ler um campo em shell:

```bash
ftp_field() { grep -E "^$1=" /root/ftp_credentials.txt | head -1 | cut -d= -f2-; }
email_field() { grep -E "^$1=" /root/email_credentials.txt | head -1 | cut -d= -f2-; }
```

> **Nunca** faça commit desses arquivos, da sessão do WhatsApp (`whatsapp_session.db`),
> do banco de mensagens (`wa_messages.db`) ou de `.ftprules`. O `.gitignore` cobre os
> casos mais comuns, mas mantenha os segredos fora do diretório do repo.

## Ajustando para outra máquina

Os caminhos absolutos (`/root/...`, `/usr/local/lib/node_modules/@artickc/...`) refletem
a instalação de referência. Em outro ambiente, edite os caminhos no `SKILL.md` da skill
correspondente, mantendo a mesma estrutura de arquivos de credenciais.

## Licença

Uso pessoal. Sem garantias.
