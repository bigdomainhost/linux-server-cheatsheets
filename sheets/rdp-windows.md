# RDP troubleshooting cheat sheet (Windows Server)

Full guide: [Remote Desktop can't connect: fix RDP on a Windows VPS](https://bigdomainhost.in/guide-rdp-cant-connect.html)

Server commands are for Windows Server 2019/2022, PowerShell as Administrator. Client commands for Windows PowerShell and Ubuntu 24.04/22.04. RDP default port: **3389** TCP and UDP (IANA name `ms-wbt-server`).

> **Lockout risk.** Open your provider's out-of-band console in a second window *before* changing firewall rules, ports or security settings. It works without RDP.

## 1. Port test (from the client that fails)

```powershell
Test-NetConnection -ComputerName 203.0.113.10 -Port 3389     # look for TcpTestSucceeded : True
psping -accepteula 203.0.113.10:3389                         # Sysinternals alternative
```

```bash
sudo apt install netcat-openbsd
nc -vz -w 5 203.0.113.10 3389          # succeeded / Connection refused / no answer
```

| Result | Means | Go to |
|---|---|---|
| Timeout / `False` | dropped: firewall, wrong IP, server off | section 2 |
| Refused | up, nothing listening on that port | section 3 |
| Opens, mstsc still fails | path fine; RDP settings / cert / CredSSP | sections 4-5 |

Test from two networks (e.g. a phone hotspot). Works on only one = that network blocks outbound RDP.

## 2. Timeout checklist

- Server running? (control panel status)
- Right IP? (copy it again; rebuilds can change it)
- Provider firewall: inbound TCP 3389 (or your port) from your IP
- Windows Firewall: section 3
- Your network: hotspot test above

## 3. Inside the server (console, PowerShell as Admin)

```powershell
# RDP enabled? 0x0 = yes, 0x1 = no
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections

# enable RDP + built-in firewall rules
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# value flips back to 1? find the Group Policy
gpresult /H C:\gpresult.html

# services
Get-Service TermService, UmRdpService
Start-Service TermService

# listener and port
qwinsta                                              # rdp-tcp ... Listen
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name PortNumber
cmd /c 'netstat -ano | find "3389"'                  # LISTENING line
cmd /c 'tasklist /svc | find "<pid>"'                # who holds the port

# firewall rules and allowed remote addresses
Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select-Object DisplayName, Enabled, Profile
Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Get-NetFirewallAddressFilter
```

No `rdp-tcp` listener at all: back up the RDP-Tcp key with `reg export`, import the key from a healthy server on the same Windows version, restart TermService.

## 4. "An internal error has occurred"

On the server console: `mstsc` → connect to `localhost`.
- Works: the server is fine; the fault is on the network path.
- Fails: negotiation or the RDP self-signed certificate.

Temporary negotiation test (turns NLA off; console only, restore the same day):

```powershell
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v SecurityLayer /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v UserAuthentication /t REG_DWORD /d 0 /f
# restore the recommended values
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v SecurityLayer /t REG_DWORD /d 2 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v UserAuthentication /t REG_DWORD /d 1 /f
```

Re-create the certificate: `certlm.msc` → Remote Desktop → Certificates → delete the self-signed cert → restart Remote Desktop Services.

```powershell
Get-Service CertPropSvc, KeyIso, CryptSvc, SessionEnv    # must run to create the cert
```

- `C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys`: Administrators full control, Everyone read + write.
- RDP-Tcp key: `SSLCertificateSHA1HashValue` not matching the current thumbprint → delete it, restart (not on a Connection Broker); `fAllowSecProtocolNegotiation` = 1; `MinEncryptionLevel` = 3.
- SSL Cipher Suite Order policy → Not configured.
- Hangs ~1 min, errors, then works on retry: check the Windows Audio service.

## 5. CredSSP encryption oracle remediation (CVE-2018-0886)

| `AllowEncryptionOracle` | Setting | Effect |
|---|---|---|
| 0 | Force updated clients | server refuses unpatched clients |
| 1 | Mitigated (default since May 2018) | no fallback; server still accepts unpatched clients |
| 2 | Vulnerable | fallback allowed; exposes servers |

Fix: Windows Update on **both** sides (the server via the console), reboot.
Temporary client-side bridge only (reboot needed; delete it or set it back to 1 after patching):

```powershell
REG ADD HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters /v AllowEncryptionOracle /t REG_DWORD /d 2
```

## 6. Custom port (firewall rule first, then port, then restart)

```powershell
$port = 50389
New-NetFirewallRule -DisplayName 'RDP-Custom-TCP-In' -Direction Inbound -Action Allow -Protocol TCP -LocalPort $port
New-NetFirewallRule -DisplayName 'RDP-Custom-UDP-In' -Direction Inbound -Action Allow -Protocol UDP -LocalPort $port
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name PortNumber -Value $port
Restart-Computer
```

Connect as `203.0.113.10:50389`; open the port in the provider firewall too. A port change cuts scanner noise; restricting RDP to your own IPs is the real control.

## 7. Connect from Ubuntu

```bash
sudo apt install freerdp3-x11                                     # 24.04
xfreerdp3 /v:203.0.113.10 /u:Administrator /cert:tofu
xfreerdp3 /v:203.0.113.10:50389 /u:Administrator /cert:tofu /log-level:DEBUG
sudo apt install freerdp2-x11                                     # 22.04
xfreerdp /v:203.0.113.10 /u:Administrator /cert:tofu
```

`/cert:tofu` trusts on first use and refuses a changed cert later (safer than `/cert:ignore`). `DEBUG` shows the failing stage: TCP, TLS or NLA.

Full guide: https://bigdomainhost.in/guide-rdp-cant-connect.html
