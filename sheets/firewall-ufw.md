# UFW firewall cheat sheet

Full guide: [UFW on Ubuntu without locking yourself out](https://bigdomainhost.in/guide-ufw-firewall-ubuntu.html)

Written for Ubuntu 24.04 (ufw 0.36.2) and 22.04 (ufw 0.36.1). UFW is installed but **inactive** by default; once enabled it denies all incoming traffic and starts at every boot.

> **Lockout risk.** Allow SSH *before* `ufw enable`. Keep your current session open and test a new login from a **second terminal**. If you lose every session, use your provider's web/out-of-band console.

## Safe order

```bash
sudo ufw status verbose          # where you start
sudo ufw allow OpenSSH           # SSH first (22/tcp, IPv4 + IPv6)
sudo ufw --dry-run enable        # preview only
sudo ufw enable                  # answer y to the SSH warning
sudo ufw status verbose
ssh -v deploy@203.0.113.10       # from a SECOND terminal
```

Scripts: `sudo ufw --force enable` skips the prompt (only after the SSH rule exists).

## Which port is SSH really on?

```bash
sudo ss -tlnp | grep -E 'sshd|systemd'
sudo sshd -T | grep -i '^port'
sudo ufw app list                # OpenSSH profile comes from openssh-server
sudo ufw app info OpenSSH        # covers 22/tcp only
```

## Defaults (already the install defaults)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Routed is deny. Denying outgoing breaks apt, DNS and time sync until you add rules.

## SSH rule variants

| Situation | Command |
|---|---|
| Standard port 22 | `sudo ufw allow OpenSSH` |
| Same, by number | `sudo ufw allow 22/tcp` |
| Custom port | `sudo ufw allow 2222/tcp` |
| One source IP only (must be static) | `sudo ufw allow proto tcp from 203.0.113.7 to any port 22` |
| Rate-limit (deny an IP with 6+ connections in 30 s) | `sudo ufw limit OpenSSH` |

`ufw allow ssh` works too but always means 22. Moving SSH: allow the new port, switch sshd, log in on it, *then* `sudo ufw delete allow OpenSSH`.

## Common rules

```bash
sudo ufw allow 80,443/tcp                                   # web (numeric list, no spaces, max 15 ports)
sudo ufw allow 'Nginx Full'                                 # same, via the Nginx profile
sudo ufw allow proto tcp from 10.10.0.5 to any port 3306    # DB from one app server only
sudo ufw allow 8443/tcp comment 'admin panel'               # label a rule
sudo ufw insert 1 deny from 198.51.100.23                   # block an IP above the allows
```

Never open 3306/5432 to everyone. Do not open app ports (3000, 8000) that sit behind Nginx.

## Read and inspect

```bash
sudo ufw status                  # Status: active/inactive + rules
sudo ufw status verbose          # + defaults and logging level
sudo ufw status numbered         # rule numbers (first match wins, top down)
sudo ufw show listening          # listening ports and the rules that hit them
sudo ufw show raw                # complete ruleset, incl. before.rules
```

`Anywhere` = `0.0.0.0/0` / `::/0`; `(v6)` rows are the IPv6 copies; `LIMIT` = rate-limited rule.

## Delete

```bash
sudo ufw delete allow 80/tcp     # by rule text: removes IPv4 + IPv6 copies
sudo ufw status numbered
sudo ufw delete 3                # by number: one line only; numbers shift after each delete
```

Check another rule still allows SSH before deleting anything on port 22.

## Symptoms

| Symptom | Usually means | Fix |
|---|---|---|
| SSH times out after `ufw enable` | no rule for the real SSH port | console: `sudo ufw allow 22/tcp` (or `ufw disable`) |
| `Connection refused` straight away | nothing listening, or a REJECT rule | `sudo ss -tlnp`, `systemctl status ssh` |
| `Could not find a profile matching 'OpenSSH'` | openssh-server missing / typo | `sudo ufw allow 22/tcp` |
| `Skipping adding existing rule` | rule already there | nothing to do |
| `Could not delete non-existent rule` | text does not match exactly | delete by number |
| ALLOW shown, port still unreachable | provider firewall in front | open it there too |
| Docker port reachable despite UFW | Docker NAT runs before UFW | publish on `127.0.0.1` |
| `Status: inactive` after reboot | never enabled, or `ufw disable` | `sudo ufw enable` |

## Docker gap

```bash
docker run -d -p 127.0.0.1:8080:80 nginx     # not -p 8080:80
```

Expose it through a reverse proxy on 80/443, which UFW does control. Keep Docker Engine current (28.0+ closed same-segment access to localhost-published ports).

## Locked out (from the provider console)

```bash
sudo ufw status numbered
sudo ufw allow 22/tcp            # or your custom port
sudo ufw disable                 # temporary; also stops it at boot, re-enable later
```

## Avoid

- `sudo ufw reset`: disables UFW and wipes rules, SSH rule included.
- `ufw logging high|full`: can fill a disk. `low` (default) logs to the kernel log / `/var/log/ufw.log`.
- Blocking ping: makes diagnosis harder and does not hide open ports.
- `allow 1:65535/tcp`: makes the firewall pointless.

Full guide: https://bigdomainhost.in/guide-ufw-firewall-ubuntu.html
