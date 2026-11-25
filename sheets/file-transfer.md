# File transfer cheat sheet

Full guides: [FTP vs SFTP vs FTPS](https://bigdomainhost.in/guide-ftp-vs-sftp-vs-ftps.html) · [SFTP and scp](https://bigdomainhost.in/guide-sftp-scp.html)

## Pick the protocol first, the port follows

| | FTP | FTPS (explicit) | FTPS (implicit) | SFTP |
|---|---|---|---|---|
| Encrypted | no | yes (TLS) | yes (TLS) | yes (SSH) |
| Port | 21 + data ports | 21 + data ports | 990 + data ports | 22 (or the SSH port) |
| Firewall | 21 + passive range | 21 + passive range, opened by hand | 990 + passive range | one port |
| Use when | closed network only | you only have FTP accounts | an old system requires it | you have SSH access |

SFTP is not "FTP over SSH". It is a separate protocol inside SSH, with no PASV and no port 21.

## sftp (interactive)

```bash
sftp deploy@203.0.113.10
sftp -P 2222 deploy@203.0.113.10          # custom port: capital -P
```

| Inside sftp | Does |
|---|---|
| `ls`, `cd`, `pwd` | remote side |
| `lls`, `lcd`, `lpwd` | local side |
| `put file.zip` / `get file.zip` | upload / download |
| `put -r site/` / `get -r logs/` | whole folder |
| `reput` / `reget` | resume an interrupted transfer |
| `mkdir`, `rm`, `rename`, `chmod 644 file` | remote file management |
| `bye` | quit |

Batch mode: `sftp -b commands.txt deploy@203.0.113.10` (needs key login).

## scp (one-off copies)

```bash
scp file.zip deploy@203.0.113.10:/var/www/          # upload
scp deploy@203.0.113.10:/var/log/nginx/error.log .  # download
scp -r site/ deploy@203.0.113.10:/var/www/site/     # folder
scp -P 2222 -i ~/.ssh/id_ed25519 file.zip deploy@203.0.113.10:~
scp -O file.zip old-host:~                          # legacy SCP protocol for old servers
```

Since OpenSSH 9.0, `scp` uses the SFTP protocol underneath by default; `-O` forces the old one when a server has no SFTP subsystem.

## rsync (repeatable syncs)

```bash
rsync -avz site/ deploy@203.0.113.10:/var/www/site/                 # copy contents of site/
rsync -avz site deploy@203.0.113.10:/var/www/                       # copy the folder itself
rsync -avz --delete --dry-run site/ deploy@203.0.113.10:/var/www/site/   # preview a mirror
rsync -avz --delete site/ deploy@203.0.113.10:/var/www/site/        # then run it
rsync -avzP -e "ssh -p 2222" big.tar.gz deploy@203.0.113.10:~       # custom port, progress, resumable
rsync -avz --exclude node_modules --exclude .git ./ web1:/srv/app/
```

- Trailing slash on the source = copy its contents; no slash = copy the directory.
- `--delete` removes files on the destination that are not in the source. Always `--dry-run` first.
- rsync must be installed on both ends.

## FTPS from a script

lftp, explicit FTPS, TLS required, upload a folder:

```bash
lftp -u ftpuser -e "set ftp:ssl-force true; set ftp:ssl-protect-data true; set ssl:verify-certificate yes; mirror -R --verbose ./site /public_html; quit" ftp.example.com
```

`mirror -R` uploads local to remote; plain `mirror` downloads. Add `--delete` only after a check run with `--dry-run`.

curl, explicit FTPS, refuse to continue without TLS:

```bash
curl --ssl-reqd -u ftpuser --list-only ftp://ftp.example.com/
curl --ssl-reqd -u ftpuser -T backup.zip ftp://ftp.example.com/backups/
```

curl asks for the password when `-u` has no `:password`, which keeps it out of your shell history.

## Common errors

| Error | Usually |
|---|---|
| Listing hangs after login (FTP/FTPS) | passive data ports not open on the server firewall |
| `530 Login incorrect` | wrong user/password, or the account is SFTP-only |
| `Permission denied (publickey)` (SFTP/scp/rsync) | key not on the server or wrong user |
| `Connection refused` on 22 | SSH not running on that port, or a firewall rejecting |
| `rsync: command not found` | rsync missing on the remote end |
| `subsystem request failed` (scp) | server lacks SFTP; retry with `scp -O` |
