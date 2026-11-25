# Ubuntu VPS first-boot checklist

Full guide: [Ubuntu 24.04 VPS initial setup](https://bigdomainhost.in/guide-ubuntu-vps-initial-setup.html) · Script version: [bigdomainhost/vps-first-boot](https://github.com/bigdomainhost/vps-first-boot)

Written for Ubuntu 24.04 LTS (most steps also apply to 22.04 and Debian 12). About 15 minutes.

> **Do not lock yourself out.** Keep your first root session open. After each SSH or firewall change, log in from a **second terminal** before you continue. If you do get locked out, use your provider's web/out-of-band console.

## Checklist

- [ ] 1. Log in with your key, update
- [ ] 2. Create a sudo user, copy the key, test it
- [ ] 3. Key-only SSH, root login off, test it
- [ ] 4. ufw: allow SSH, then enable
- [ ] 5. Confirm automatic security updates
- [ ] 6. Timezone and hostname
- [ ] 7. Swap (1–2 GB RAM plans)
- [ ] 8. fail2ban (optional)
- [ ] 9. Check listening ports
- [ ] 10. Reboot if needed, snapshot, off-server backup

## 1. Update (as root)

```bash
ssh root@203.0.113.10
apt update && apt upgrade -y
ls /var/run/reboot-required 2>/dev/null && echo "reboot needed at the end"
```

## 2. Sudo user (as root)

```bash
adduser deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

Test from a second terminal: `ssh deploy@203.0.113.10` then `sudo whoami` → `root`.

## 3. Key-only SSH (as deploy, with sudo)

```bash
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sudo sshd -t && sudo systemctl restart ssh
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication)'   # all "no"
```

Test: `ssh deploy@…` works; `ssh root@…` gives `Permission denied (publickey)`.
Changing `Port`/`ListenAddress` on 24.04 also needs `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`.

## 4. Firewall

```bash
sudo ufw allow OpenSSH          # or: sudo ufw allow 2222/tcp for a custom SSH port
sudo ufw enable
sudo ufw status verbose
# later, only when a web server runs:
sudo ufw allow 80,443/tcp
```

A provider firewall in front of the VM needs the same ports opened.

## 5. Automatic security updates

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades           # both APT::Periodic lines = "1"
systemctl status unattended-upgrades --no-pager
# if missing:
sudo apt install unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades
```

Automatic reboots are off by default; check `/var/run/reboot-required` now and then.

## 6. Timezone and hostname

```bash
sudo timedatectl set-timezone Asia/Kolkata        # or your zone: timedatectl list-timezones
sudo hostnamectl set-hostname web1
sudoedit /etc/hosts                               # put web1 on the 127.0.1.1 line
```

If the hostname reverts after reboot, cloud-init manages it: add `preserve_hostname: true` to `/etc/cloud/cloud.cfg.d/99-hostname.cfg`.

## 7. Swap (skip if `swapon --show` lists one)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # run once only
sudo findmnt --verify
free -h
```

## 8. fail2ban (optional)

```bash
sudo apt install fail2ban
sudo fail2ban-client status sshd
```

Own settings go in `/etc/fail2ban/jail.local`. Add a fixed office IP to `ignoreip`.

## 9. What is listening

```bash
sudo ss -tlnp
```

Fresh 24.04: port 22 (systemd + sshd, socket activation) and the resolver on 127.0.0.53/127.0.0.54. Anything on `0.0.0.0` or `[::]` is public unless firewalled.

## 10. Reboot, snapshot, backup

```bash
[ -f /var/run/reboot-required ] && sudo reboot
```

Then take a snapshot in your provider's panel. A snapshot lives on the same platform as the server, so keep an off-server backup as well and test a restore once.

## Next

- Point a domain: [guide](https://bigdomainhost.in/guide-point-domain-to-vps.html)
- Deploy a Python app behind Nginx: [guide](https://bigdomainhost.in/guide-deploy-python-app-vps.html)
- Size RAM: [guide](https://bigdomainhost.in/guide-how-much-ram-does-a-vps-need.html)
