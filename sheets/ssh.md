# SSH cheat sheet

Full guides: [SSH keys](https://bigdomainhost.com/guide-ssh-keys.html) · [SSH tunnels](https://bigdomainhost.com/guide-ssh-tunnel.html)

## Keys

```bash
ssh-keygen -t ed25519 -C "laptop-2026"            # new key; set a passphrase
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_work   # second key, different file (never overwrite)
ssh-keygen -t rsa -b 4096 -C "old-device"         # only for systems without Ed25519 support
ssh-keygen -p -f ~/.ssh/id_ed25519                # change the passphrase
ssh-keygen -lf ~/.ssh/id_ed25519.pub              # show a key's fingerprint
ssh-keygen -y -f ~/.ssh/id_ed25519                # print the public key from a private key
ssh-keygen -R 203.0.113.10                        # forget a host key after a server rebuild
```

| File | What | Share? |
|---|---|---|
| `~/.ssh/id_ed25519` | private key | never |
| `~/.ssh/id_ed25519.pub` | public key | yes |
| `~/.ssh/authorized_keys` (on the server) | public keys allowed to log in | n/a |
| `~/.ssh/known_hosts` | server host keys you have accepted | n/a |

## ssh-agent

```bash
eval "$(ssh-agent -s)"            # Linux: start an agent
ssh-add ~/.ssh/id_ed25519         # load the key (asks for the passphrase once)
ssh-add -l                        # list loaded keys
ssh-add -D                        # unload all keys
```

Windows (PowerShell as Administrator, once):

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

Then in a normal window: `ssh-add $env:USERPROFILE\.ssh\id_ed25519`.

## Copy a key to a server

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@203.0.113.10
```

Windows (no `ssh-copy-id`):

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh deploy@203.0.113.10 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Permissions sshd insists on:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh     # if you created them as root
```

## ~/.ssh/config

```
Host web1
    HostName 203.0.113.10
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3

Host db1
    HostName 10.0.0.5
    User deploy
    ProxyJump web1            # reach a private host through web1
```

Then `ssh web1`, `ssh db1`, `scp file web1:/tmp/`.
One-off jump without config: `ssh -J deploy@203.0.113.10 deploy@10.0.0.5`.

## Tunnels

```bash
# Local: laptop 127.0.0.1:13306 -> MySQL listening only on the server
ssh -N -L 13306:127.0.0.1:3306 deploy@203.0.113.10

# Same, in the background, failing loudly if the port is taken
ssh -f -N -o ExitOnForwardFailure=yes -L 13306:127.0.0.1:3306 deploy@203.0.113.10
pgrep -af "ssh -f -N"     # find it; stop it with kill <pid>

# Remote: server 127.0.0.1:8080 -> app on your laptop port 3000
ssh -N -R 8080:127.0.0.1:3000 deploy@203.0.113.10

# Dynamic: SOCKS5 proxy on laptop port 1080
ssh -N -D 1080 deploy@203.0.113.10
curl --socks5-hostname 127.0.0.1:1080 https://example.com
```

- Connect the MySQL client to `127.0.0.1` (not `localhost`, which means the Unix socket): `mysql -h 127.0.0.1 -P 13306 -u app -p`.
- Forwarded ports bind to loopback by default. Leave `-g` and `GatewayPorts` off unless you mean to share them.
- In config: `LocalForward 13306 127.0.0.1:3306`, `RemoteForward 8080 127.0.0.1:3000`, `DynamicForward 1080`.
- Auto-restart: `autossh -M 0 -N db-tunnel` (relies on the `ServerAlive*` settings).

## Server side checks

```bash
sudo sshd -t                                   # syntax check; run before every restart
sudo sshd -T | grep -Ei '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowtcpforwarding|gatewayports)'
sudo systemctl restart ssh                     # Ubuntu/Debian; the service is sshd on RHEL-family
systemctl is-active ssh.socket                 # Ubuntu 22.10+: socket activation
sudo journalctl -u ssh -n 50 --no-pager        # recent sshd log (Ubuntu/Debian)
```

Key-only drop-in, `/etc/ssh/sshd_config.d/00-hardening.conf` (first value read wins, so `00-` beats `50-cloud-init.conf`):

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
```

On Ubuntu 24.04, changing `Port` or `ListenAddress` also needs `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`. Open the new port in every firewall first.

## Debugging a login

```bash
ssh -v deploy@203.0.113.10      # -vv / -vvv for more
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no deploy@203.0.113.10
nc -zv 203.0.113.10 22          # is the port reachable at all?
```

| Message | Usually means |
|---|---|
| `Connection refused` | nothing listening on that port, or a firewall rejecting |
| `Connection timed out` | packets dropped: firewall, wrong IP, server down |
| `Permission denied (publickey)` | key not in `authorized_keys`, wrong user, or bad permissions |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | server rebuilt or a different host on that IP; check, then `ssh-keygen -R` |

Stuck session: press Enter, then `~.` to close it from the client side.

Test changes from a second terminal and keep your current session open until the new login works.
