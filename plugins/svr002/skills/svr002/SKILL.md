---
description: svr002 skill for RobboHome automation.
---

# Skill: svr002 Server Management

## Access
- SSH: `ssh svr002` or `ssh robbohome-server` (alias in ~/.ssh/config)
- Key: `~/.ssh/id_ed25519` (ed25519, comment: github@geekythings.co.uk)
- IP: 192.168.1.17, user: robbohomebot, passwordless sudo
- Local: http://192.168.1.17:9090 (Cockpit) / https://cockpit.robbohome.com

## SSH config entry (~/.ssh/config)
```
Host svr002 robbohome-server
  HostName 192.168.1.17
  User robbohomebot
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

## Key directories
- ~/data/hello-world/ — hello-world app deployment
- ~/data/ollama/ — Ollama + Open WebUI stack
- ~/actions-runner/ — GitHub Actions self-hosted runner

## Running services
| Service   | Start/stop command                        |
|-----------|-------------------------------------------|
| Docker    | sudo systemctl start/stop docker          |
| Cockpit   | sudo systemctl start/stop cockpit.socket  |
| Portainer | docker start/stop portainer               |
| Ollama    | cd ~/data/ollama && docker compose up/down|
| GH Runner | sudo systemctl start/stop actions.runner.robinsondan87-robbohome-hello-world.robbohome-server |

## Bootstrap script
~/data/infrastructure/server-bootstrap.sh — run with:
```bash
sudo bash server-bootstrap.sh RUNNER_TOKEN robinsondan87/robbohome-hello-world
```
Get fresh runner token: gh api repos/robinsondan87/robbohome-hello-world/actions/runners/registration-token --method POST --jq '.token'

## SSH hardening config
/etc/ssh/sshd_config.d/robbohome-hardening.conf:
- PasswordAuthentication yes (kept on for internal machines)
- PermitRootLogin no
- AllowUsers robbohomebot
- MaxAuthTries 3

## UFW rules
- SSH (22): 192.168.0.0/16 and 10.0.0.0/8 only
- 9090, 9000, 3000, 3001, 11434: open (protected by Cloudflare Zero Trust externally)

## Cockpit proxy config
/etc/cockpit/cockpit.conf — required for Cloudflare tunnel:
```ini
[WebService]
Origins = https://cockpit.robbohome.com wss://cockpit.robbohome.com
ProtocolHeader = X-Forwarded-Proto
AllowUnencrypted = true
```

## Boot mode
Headless by default (multi-user.target). To start GNOME:
```bash
sudo systemctl start graphical.target
```
