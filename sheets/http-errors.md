# HTTP and browser errors: one-line first checks

Full guides: [502](https://bigdomainhost.in/guide-502-bad-gateway.html) · [503](https://bigdomainhost.in/guide-503-service-unavailable.html) · [504](https://bigdomainhost.in/guide-504-gateway-timeout.html) · [403](https://bigdomainhost.in/guide-403-forbidden.html) · [500](https://bigdomainhost.in/guide-500-internal-server-error.html) · [This site can't be reached](https://bigdomainhost.in/guide-this-site-cant-be-reached.html)

Rule one: find out **which layer sent the error** (CDN, web server, app) and read **that layer's log** before changing anything.

```bash
curl -sI https://example.com | head -n 20            # status line + Server / cf-ray headers
sudo tail -n 50 /var/log/nginx/error.log             # Nginx
sudo tail -n 50 /var/log/apache2/error.log           # Apache (Debian/Ubuntu); /var/log/httpd/error_log on RHEL-family
sudo journalctl -u php8.3-fpm -n 50 --no-pager       # PHP-FPM (Ubuntu 24.04 package name)
```

## HTTP status codes (the server answered)

| Code | Means | First check | Guide |
|---|---|---|---|
| 403 Forbidden | a rule refused the request on purpose | `namei -l /var/www/site/index.html` (every directory needs `x` for the web server user) | [403](https://bigdomainhost.in/guide-403-forbidden.html) |
| 500 Internal Server Error | something crashed or a config line is invalid | the error log line at that timestamp; for Apache, comment out new `.htaccess` lines | [500](https://bigdomainhost.in/guide-500-internal-server-error.html) |
| 502 Bad Gateway | the proxy got no usable answer from the backend | `systemctl status php8.3-fpm` (or your app's unit), then match the socket/port in the proxy config: `sudo ss -xlp \| grep php` | [502](https://bigdomainhost.in/guide-502-bad-gateway.html) |
| 503 Service Unavailable | overloaded, at a limit, in maintenance, or no backend available | WordPress: `ls -la /path/to/site/.maintenance`; FPM: `grep -i "max_children" /var/log/php8.3-fpm.log` | [503](https://bigdomainhost.in/guide-503-service-unavailable.html) |
| 504 Gateway Timeout | the backend answered too slowly | `curl -o /dev/null -s -w '%{time_starttransfer}s\n' https://example.com/slow-page` and the slow query log | [504](https://bigdomainhost.in/guide-504-gateway-timeout.html) |

Notes:
- Nginx's `proxy_read_timeout` / `fastcgi_read_timeout` default to 60 s. Raise them only for genuinely long tasks; queue the rest.
- Behind Cloudflare: a Cloudflare-branded 502/504 page came from your origin; 520–526 are Cloudflare reporting on its connection to your origin.
- Typical 502 log line: `connect() to unix:/run/php/php8.3-fpm.sock failed (2: No such file or directory)` means a wrong socket path or FPM is stopped.

## Chrome / Chromium `ERR_` codes (no usable answer)

Same codes in Chrome, Edge, Brave and Opera. First test for everyone: open the site on **mobile data with Wi-Fi off**. Loads there = local device, router or DNS problem. Fails everywhere = the site's side.

| Code | Means | One-line check (site owner) | Guide |
|---|---|---|---|
| `DNS_PROBE_FINISHED_NXDOMAIN` | DNS says the name does not exist | `dig +short example.com A` and `dig NS example.com +short` | [NXDOMAIN](https://bigdomainhost.in/guide-dns-probe-finished-nxdomain.html) |
| `ERR_NAME_NOT_RESOLVED` | the lookup failed | `dig example.com @1.1.1.1` vs `dig example.com @8.8.8.8` | [name not resolved](https://bigdomainhost.in/guide-err-name-not-resolved.html) |
| `ERR_CONNECTION_REFUSED` | host reached, nothing listening on that port | `sudo ss -tlnp \| grep -E ':(80\|443) '` | [refused](https://bigdomainhost.in/guide-err-connection-refused.html) |
| `ERR_CONNECTION_TIMED_OUT` | no reply at all: firewall drop, wrong IP, host down | `nc -zv -w 5 203.0.113.10 443` then `mtr -rwc 20 203.0.113.10` | [timed out](https://bigdomainhost.in/guide-err-connection-timed-out.html) |
| `ERR_CONNECTION_RESET` | connection opened, then cut | `curl -v https://example.com` from two networks; check WAF / proxy logs | [reset](https://bigdomainhost.in/guide-connection-reset.html) |
| `ERR_CONNECTION_CLOSED` | server closed it without sending data | `curl -v https://example.com`; check the TLS vhost exists for that name | [closed](https://bigdomainhost.in/guide-err-connection-closed.html) |
| `ERR_EMPTY_RESPONSE` | connected, got zero bytes back | error log for a crashed worker at that timestamp | [empty response](https://bigdomainhost.in/guide-err-empty-response.html) |
| `ERR_SSL_PROTOCOL_ERROR` | TLS handshake failed | `openssl s_client -connect example.com:443 -servername example.com </dev/null` | [SSL protocol](https://bigdomainhost.in/guide-err-ssl-protocol-error.html) |
| `NET::ERR_CERT_AUTHORITY_INVALID` | chain not trusted (self-signed, missing intermediate) | `openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null` | [authority invalid](https://bigdomainhost.in/guide-err-cert-authority-invalid.html) |
| `NET::ERR_CERT_DATE_INVALID` | certificate expired, or the visitor's clock is wrong | `echo \| openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \| openssl x509 -noout -dates` | [date invalid](https://bigdomainhost.in/guide-net-err-cert-date-invalid.html) |
| `ERR_TOO_MANY_REDIRECTS` | redirect loop (HTTP↔HTTPS or www↔non-www) | `curl -sSIL http://example.com \| grep -iE '^(HTTP\|location)'` | [redirects](https://bigdomainhost.in/guide-err-too-many-redirects.html) |
| `ERR_HTTP2_PROTOCOL_ERROR` | the HTTP/2 stream broke mid-response | `curl -v --http2 https://example.com -o /dev/null`, then `--http1.1` to compare | [HTTP/2](https://bigdomainhost.in/guide-err-http2-protocol-error.html) |
| `ERR_NETWORK_CHANGED` | the visitor's own network switched mid-load | nothing on the server; check the visitor's Wi-Fi / VPN | [network changed](https://bigdomainhost.in/guide-err-network-changed.html) |

Redirect loop behind Cloudflare: set SSL/TLS mode to **Full (strict)**, not Flexible, and keep one HTTPS redirect rule.

Visitor-side quick fixes (in order): check the URL, try another network, turn off VPN/proxy, flush DNS (`ipconfig /flushdns` on Windows, `resolvectl flush-caches` on systemd-resolved Linux), check the system clock.
