# Ports cheat sheet (hosting)

Full guides: [Email port numbers](https://bigdomainhost.com/guide-email-port-numbers.html) · [cPanel port numbers](https://bigdomainhost.com/guide-cpanel-port-numbers.html)

## Web, shell and files

| Port | Proto | Service | Notes |
|---|---|---|---|
| 22 | TCP | SSH, SFTP, scp, rsync over SSH | one port for all of them |
| 80 | TCP | HTTP | keep open for redirects and Let's Encrypt HTTP-01 |
| 443 | TCP | HTTPS (HTTP/1.1, HTTP/2) | |
| 443 | UDP | HTTP/3 (QUIC) | only if your server enables it |
| 21 | TCP | FTP control / explicit FTPS | plus a passive data port range |
| 20 | TCP | FTP active-mode data | rarely needed today |
| 990 | TCP | implicit FTPS | legacy |
| 53 | TCP+UDP | DNS | only on servers that answer DNS |

## Mail

| Port | Service | Use it? |
|---|---|---|
| 587 | SMTP submission, STARTTLS | yes, for sending from mail apps |
| 465 | SMTP submission, implicit TLS | yes, alternative to 587 |
| 993 | IMAP over TLS | yes, for receiving |
| 995 | POP3 over TLS | only if you must use POP3 |
| 25 | SMTP server-to-server | not for mail apps; blocked outbound on many networks and VPS providers |
| 143 | IMAP, plain or STARTTLS | avoid unencrypted |
| 110 | POP3, plain or STARTTLS | avoid unencrypted |
| 26 | alternative SMTP on some cPanel servers | only if your host says so |

## cPanel / WHM / Webmail

| Port (SSL) | Port (no SSL) | Interface | Also reachable at |
|---|---|---|---|
| 2083 | 2082 | cPanel | `https://cpanel.example.com` (443) |
| 2087 | 2086 | WHM | `https://whm.example.com` (443) |
| 2096 | 2095 | Webmail | `https://webmail.example.com` (443) |
| 2089 (outbound) | | cPanel licence check | must be allowed out |

Use the SSL ports. If an office or hotel network blocks 2083, the service subdomains work over 443.

## Databases and app ports

| Port | Service | Open to the internet? |
|---|---|---|
| 3306 | MySQL / MariaDB | only if something off the server connects; prefer an SSH tunnel |
| 5432 | PostgreSQL | same |
| 6379 | Redis | no; bind to 127.0.0.1 |
| 27017 | MongoDB | no; bind to 127.0.0.1 |
| 3000, 5000, 8000, 8080 | typical app dev ports | no; put Nginx or Caddy in front |
| 3389 | RDP (Windows) | restrict to known IPs |
| 8443 / 8880 | Plesk (HTTPS / HTTP) | only if you run Plesk |

## What is listening here?

```bash
sudo ss -tlnp                  # TCP listeners with process names
sudo ss -ulnp                  # UDP listeners
sudo ss -tlnp 'sport = :443'   # just one port
```

`127.0.0.1` / `[::1]` = local only. `0.0.0.0` / `[::]` / `*` = all interfaces, reachable unless a firewall blocks it.

## Can I reach it from outside?

```bash
nc -zv 203.0.113.10 443                         # TCP connect test
nc -zv -w 5 203.0.113.10 2083                   # with a 5 s timeout
curl -v telnet://203.0.113.10:587               # when nc is not installed (Ctrl+C to exit)
openssl s_client -connect mail.example.com:993 -servername mail.example.com </dev/null
openssl s_client -starttls smtp -connect mail.example.com:587 </dev/null
```

`succeeded` / `Connected` = open. `refused` = host reached, nothing listening (or REJECT rule). Hangs then times out = dropped by a firewall, wrong IP or host down.

## Open a port (ufw)

```bash
sudo ufw allow OpenSSH                     # do this before `ufw enable`
sudo ufw allow 80,443/tcp
sudo ufw allow from 198.51.100.7 to any port 3306 proto tcp   # one IP only
sudo ufw status numbered
sudo ufw delete 3                          # remove rule number 3
```

Cloud servers often have a second, provider-level firewall in front of ufw. A port has to be open in both.
