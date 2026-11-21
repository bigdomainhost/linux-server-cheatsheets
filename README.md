# linux-server-cheatsheets

Short, dense reference sheets for the jobs that come up again and again on a Linux server: SSH, ports, moving files, npm, HTTP errors and the first boot of a new VPS. Each sheet is one page of commands you can copy, with a link at the top to a longer guide that explains the why.

Commands are written for Ubuntu 24.04 LTS and Debian 12 unless a line says otherwise. Most work unchanged on other distributions; where the service or package name differs (for example `ssh` vs `sshd`), the sheet says so.

## Sheets

| Sheet | Covers |
|---|---|
| [ssh.md](sheets/ssh.md) | Keys, ssh-agent, `~/.ssh/config`, jump hosts, local/remote/dynamic tunnels, server hardening checks |
| [ports.md](sheets/ports.md) | Common hosting ports (web, SSH, mail, FTP, databases, cPanel/WHM/Webmail) and how to test each one |
| [file-transfer.md](sheets/file-transfer.md) | FTP vs SFTP vs FTPS, `sftp`, `scp`, `rsync`, `lftp` and `curl` one-liners |
| [npm.md](sheets/npm.md) | npm cache, lockfiles, `npm ci`, npm vs pnpm command map |
| [http-errors.md](sheets/http-errors.md) | 403, 500, 502, 503, 504 and Chrome `ERR_` codes, each with a one-line first check |
| [first-boot.md](sheets/first-boot.md) | Ubuntu VPS first-boot checklist: sudo user, key-only SSH, ufw, updates, swap, snapshot |

## Conventions

- `203.0.113.10` and `example.com` are placeholders (reserved for documentation). Replace them.
- `deploy` is the example non-root user.
- Lines starting with `#` are comments; `$` prompts are left out so you can paste.
- Anything that can lock you out of a server says so. Keep one session open while you test changes from a second terminal.

## Contributing

Corrections are welcome. Open an issue or a pull request with the command, the distro and version you tested on, and a link to the upstream documentation if the behaviour changed. Please keep sheets short: if something needs a paragraph, it belongs in a guide rather than a cheat sheet.

## Licence

MIT. See [LICENSE](LICENSE).

---

Maintained by Bigdomainhost, an Indian VPS and hosting provider — full guides at https://bigdomainhost.com/docs.html

Written with AI assistance; commands checked against the upstream documentation linked in each guide.
