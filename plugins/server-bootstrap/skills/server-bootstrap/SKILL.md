---
description: server-bootstrap skill for RobboHome automation.
---

# Skill: Server Bootstrap (svr002)

One-time setup script to configure a fresh Ubuntu server as svr002.

## Run the bootstrap
```bash
# On the new server (or via SSH)
curl -fsSL https://raw.githubusercontent.com/robinsondan87/robbohome-infrastructure/main/server-bootstrap.sh | bash
```

Or clone and run locally:
```bash
git clone git@github.com:robinsondan87/robbohome-infrastructure.git
cd robbohome-infrastructure
bash server-bootstrap.sh
```

## What it installs
- Docker + Docker Compose
- NVIDIA Container Toolkit (GPU support for AI workloads)
- Portainer (container management UI on port 9000)
- Cockpit (server admin UI on port 9090)
- UFW firewall (allows SSH, 3847, 9000, 9090, 18789)
- Creates `robbohomebot` user with sudo
- Sets up standard data directory structure at `~/data/`

## Post-bootstrap checklist
- [ ] Verify Docker: `docker ps`
- [ ] Verify GPU: `nvidia-smi` (if NVIDIA card present)
- [ ] Access Portainer: http://192.168.1.17:9000 — set admin password on first visit
- [ ] Access Cockpit: http://192.168.1.17:9090
- [ ] Register GitHub Actions runners for each app — see `skills/register-runner/SKILL.md`
- [ ] Copy app data volumes from old server if migrating
- [ ] Update Cloudflare Tunnel to point to new server — see `skills/cloudflare-tunnel/SKILL.md`

## Current server
| | |
|---|---|
| Hostname | svr002 |
| IP | 192.168.1.17 |
| User | robbohomebot |
| SSH | `ssh svr002` (alias) or `ssh robbohome-server` |
| Key | `~/.ssh/svr002_remote` (ed25519, comment: svr002-remote) |
| OS | Ubuntu |

## SSH config entry (~/.ssh/config)
```
Host svr002 robbohome-server
  HostName 192.168.1.17
  User robbohomebot
  IdentityFile ~/.ssh/svr002_remote
  IdentitiesOnly yes
```

## svr001 / Unraid NAS
| | |
|---|---|
| Hostname | svr001 / unraid |
| IP | 192.168.1.200 |
| User | root |
| Port | 2223 |
| SSH | `ssh unraid` or `ssh svr001` |
| Key | `~/.ssh/codex_remote` |

```
Host unraid svr001
  HostName 192.168.1.200
  User root
  Port 2223
  IdentityFile ~/.ssh/codex_remote
  IdentitiesOnly yes
```

## Related Skills
- `skills/register-runner/SKILL.md` — register GitHub Actions runner after bootstrap
- `skills/docker-management/SKILL.md` — day-to-day Docker operations
- `skills/cloudflare-tunnel/SKILL.md` — expose apps publicly after bootstrap
