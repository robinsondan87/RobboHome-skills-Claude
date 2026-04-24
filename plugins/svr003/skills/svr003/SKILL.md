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
- Size: 1.9TB total
- Active Syncthing folders: `Pictures/`, `3dPrinting/`, `Documents/`, `SCC_2026/`, `SSD_Photos/`
- Legacy: `backup/` (pre-Syncthing archive)

## Services
| Service | Status |
|---|---|
| Tailscale | Running (100.77.187.4) |
| SSH | Port 2223 only |
| Syncthing | Running — receive-only from Unraid |

## Syncthing
Off-site backup destination. All folders receive-only from Unraid (which receives from Mac).

- Device ID: `KY6BY3S-FIQAUYQ-73G2J7T-CGDWX3E-FETZOGQ-G4T2FQA-2K5MAZI-2SDJ3AV`
- GUI binds to `127.0.0.1:8384` only — access via SSH tunnel: `ssh -L 8384:127.0.0.1:8384 svr003`
- Config: `~/.local/state/syncthing/config.xml`
- API key: in `config.xml` (`<apikey>…</apikey>`)
- Peers pinned with Tailscale IP — continues working when svr003 moves to remote location (Tailscale is a mesh VPN + Syncthing global discovery as fallback)

### Folder layout
| Folder ID | Type | Path |
|---|---|---|
| `pictures` | receiveonly | `/mnt/backup/Pictures` |
| `3dprinting` | receiveonly | `/mnt/backup/3dPrinting` |
| `documents` | receiveonly | `/mnt/backup/Documents` |
| `scc_2026` | receiveonly | `/mnt/backup/SCC_2026` |
| `ssd_photos` | receiveonly | `/mnt/backup/SSD_Photos` |

### Common commands
```bash
# Service control
ssh svr003 'systemctl --user status syncthing'
ssh svr003 'journalctl --user -u syncthing -n 50'

# API via SSH (GUI is localhost-only)
ssh svr003 "curl -s -H 'X-API-Key: <key>' http://127.0.0.1:8384/rest/system/status"

# Folder status
ssh svr003 "curl -s -H 'X-API-Key: <key>' http://127.0.0.1:8384/rest/db/status?folder=pictures"
```

## Cluster topology
```
Mac (FGG3TI4)      →      Unraid (JCN427H)      →      svr003 (KY6BY3S)
sendonly                 sendreceive (relay)           receiveonly
```

Mac is the source of truth. Unraid receives and forwards to svr003. svr003 is purely a backup — no writes propagate upstream.

## Related Skills
- `skills/svr002/SKILL.md` — primary home server
- `skills/server-bootstrap/SKILL.md` — server setup reference
- `skills/docker-management/SKILL.md` — Unraid Syncthing container is `lscr.io/linuxserver/syncthing`, managed via dockerman
