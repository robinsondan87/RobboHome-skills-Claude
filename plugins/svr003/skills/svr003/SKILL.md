---
description: svr003 — remote Raspberry Pi backup server management.
---

# Skill: svr003 Server Management

## About
Remote backup server — Raspberry Pi (aarch64, Debian). Used for off-site/remote backups of RobboHome infrastructure.

## Access
- SSH: `ssh svr003`
- Key: `~/.ssh/svr003_remote` (ed25519, comment: svr003-remote)
- IP: 192.168.20.91, Port: 22, User: robbohome

## SSH config entry (~/.ssh/config)
```
Host svr003
  HostName 192.168.20.91
  User robbohome
  Port 22
  IdentityFile ~/.ssh/svr003_remote
  IdentitiesOnly yes
```

## Hardware
| | |
|---|---|
| OS | Debian Linux (aarch64 / Raspberry Pi) |
| Kernel | 6.12.47+rpt-rpi-v8 |
| Storage | 28GB SD card (20GB free) |
| Docker | Not installed |

## Status
Fresh setup — backup configuration in progress.

## Related Skills
- `skills/svr002/SKILL.md` — primary home server
- `skills/server-bootstrap/SKILL.md` — server setup reference
