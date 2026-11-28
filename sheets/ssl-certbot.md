# Let's Encrypt + Certbot on Nginx cheat sheet

Full guide: [Let's Encrypt SSL on Nginx with Certbot](https://bigdomainhost.in/guide-lets-encrypt-certbot-nginx.html)

Written for Ubuntu 24.04 and 22.04 with Nginx from the Ubuntu archive. HTTP-01 challenge (the default): Let's Encrypt fetches a file from your server on **port 80**, and the ACME standard does not let you pick another port.

## Before you run Certbot

```bash
curl -4 https://ifconfig.co                 # server's public IPv4
curl -6 https://ifconfig.co                 # server's public IPv6 (if any)
dig +short A example.com
dig +short A www.example.com
dig +short AAAA example.com                 # must point HERE or not exist: IPv6 is tried first
dig +short CAA example.com                  # empty, or includes letsencrypt.org
sudo nginx -T | grep server_name            # every -d name must be listed
sudo ufw allow 'Nginx Full'                 # 80 + 443; open them in the provider firewall too
```

Windows: `nslookup -type=A example.com 1.1.1.1` and `nslookup -type=AAAA example.com 1.1.1.1`.

From **another** machine: `curl -I http://example.com/` (any status line, even 404 = port 80 reachable; hang = firewall). PowerShell: `Test-NetConnection example.com -Port 80`.

## Minimal server block

`/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.html;
}
```

```bash
sudo mkdir -p /var/www/example.com
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## Install (snap, recommended by the Certbot project)

```bash
sudo apt remove certbot                     # remove any apt copy first
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
certbot --version
```

apt alternative: `sudo apt install certbot python3-certbot-nginx` (24.04 ships 2.9.0, frozen). Use one method, never both.

## Issue

```bash
sudo certbot --nginx -d example.com -d www.example.com              # cert + edits Nginx + HTTP->HTTPS redirect
sudo certbot --nginx --no-redirect -d example.com -d www.example.com  # keep plain HTTP working
sudo certbot certonly --nginx -d example.com -d www.example.com     # cert only, edit Nginx yourself
sudo certbot --nginx --test-cert -d example.com                     # staging cert while debugging
sudo certbot --nginx rollback                                       # undo Certbot's config edits
```

Always put the bare name and `www` in the same certificate.

| File in `/etc/letsencrypt/live/example.com/` | Use |
|---|---|
| `fullchain.pem` | `ssl_certificate` |
| `privkey.pem` | `ssl_certificate_key` (secret) |
| `cert.pem`, `chain.pem` | not needed for Nginx |

Point Nginx at these paths; never copy the files (copies do not update on renewal).

## Renewal

```bash
sudo certbot renew --dry-run                # full renewal against staging; run it now
systemctl list-timers | grep -i certbot     # the timer the package installed
sudo certbot certificates                   # names, expiry, paths
sudo ls /var/log/letsencrypt/               # renewal logs
```

- Default lifetime 90 days today; 64 days from 10 Feb 2027, 45 days from 16 Feb 2028 (announced by Let's Encrypt).
- Certbot 4.0+ renews when a third of the lifetime remains; older versions (incl. 2.9.0) at 30 days before expiry.
- No expiry emails since 4 Jun 2025. Monitor expiry yourself.
- Keep port 80 open after issuance: renewal uses it again.

Reload Nginx after each renewal (needed with `certonly`):

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null <<'EOT'
#!/bin/sh
systemctl reload nginx
EOT
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

## Errors

| Error contains | Usually means | First fix |
|---|---|---|
| `Timeout during connect (likely firewall problem)` | port 80 dropped | ufw + provider firewall; check AAAA |
| `Connection refused` | nothing listening on 80 at that address | start Nginx; add `listen [::]:80;` |
| `Invalid response from ... 404` | another server/block answered | old A/AAAA, CDN proxy, `deny` on `/.well-known/` |
| `DNS problem: NXDOMAIN looking up A` | name not in DNS | create the record; check `-d` spelling |
| `CAA record for ... prevents issuance` | CAA excludes Let's Encrypt | add `0 issue "letsencrypt.org"` |
| `too many certificates` / `too many failed authorizations` | rate limit | wait; test with `--dry-run` |
| `Could not automatically find a matching server block` | name missing from `server_name` | add it, `nginx -t`, reload |
| `nginx: [emerg]` after Certbot | config error | `sudo nginx -t`, fix the line it names |

Is the challenge path reaching this block?

```bash
sudo mkdir -p /var/www/example.com/.well-known/acme-challenge
echo ok | sudo tee /var/www/example.com/.well-known/acme-challenge/test
curl -iL http://example.com/.well-known/acme-challenge/test     # expect "ok"
sudo rm /var/www/example.com/.well-known/acme-challenge/test
sudo ss -tlnp | grep ':80'                                      # who listens on 80, v4 and v6
```

Rate limits: 5 certs per exact name set per 7 days; 50 per registered domain per 7 days; 5 failed validations per name per account per hour.

## Wildcards or port 80 blocked: DNS-01

```bash
sudo snap set certbot trust-plugin-with-root=ok
sudo snap install certbot-dns-cloudflare     # pick your DNS provider's plugin
```

Needs a DNS API credential on the server. DNS-01 is the only way to get `*.example.com` from Let's Encrypt.

Behind Cloudflare: issue with the record on **DNS only**, then proxy on with SSL mode **Full (strict)**.

Full guide: https://bigdomainhost.in/guide-lets-encrypt-certbot-nginx.html
