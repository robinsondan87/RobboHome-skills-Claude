---
description: svr003 — remote Raspberry Pi backup server management.
---

# Skill: svr003 Server Management

## About
Remote backup server — Raspberry Pi (aarch64, Debian). Used for off-site/remote backups of RobboHome infrastructure.

## Access
- SSH: `ssh svr003`
- Key: `~/.ssh/svr003_remote` (ed25519, comment: svr003-remote)
- IP: 192.168.20.91 (LAN) / 100.77.187.4 (Tailscale), Port: 2223, User: robbohome

## SSH config entry (~/.ssh/config)
```
Host svr003
  HostName 192.168.20.91
  User robbohome
  Port 2223
  IdentityFile ~/.ssh/svr003_remote
  IdentitiesOnly yes
```

## Hardware
| | |
|---|---|
| OS | Debian Linux (aarch64 / Raspberry Pi) |
| Kernel | 6.12.47+rpt-rpi-v8 |
| Storage | 28GB SD card |
| Backup disk | 2TB exFAT, mounted at `/mnt/backup` (Mac/Windows/Linux compatible) |
| Docker | Not installed |

## Tailscale
- Installed and authenticated as `svr003`
- Tailscale IP: `100.77.187.4`

## Tailscale network
| Device | Tailscale IP |
|---|---|
| svr003 (this) | 100.77.187.4 |
| svr001 / Unraid | 100.119.202.44 |
| Dan's MacBook Pro | 100.78.90.123 |
| Robbo's Mac Mini | 100.126.7.105 |
| vmi3091030 (VPS) | 100.80.48.12 |

## Backup disk
- Device: `/dev/sda1`
- Mount: `/mnt/backup` (persistent via `/etc/fstab`)
- Format: exFAT (UUID: 6192-651E)
- Size: 1.9TB total, ~1.6TB free
- Existing folders: `3dPrinting/`, `backup/`, `Pictures/`

## Services
| Service | Status |
|---|---|
| Tailscale | Running |
| SSH | Port 2223 only |
| Syncthing | Planned — receive-only from Mac/Unraid |

## Related Skills
- `skills/svr002/SKILL.md` — primary home server
- `skills/server-bootstrap/SKILL.md` — server setup reference
