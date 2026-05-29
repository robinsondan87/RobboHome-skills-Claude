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
Dan's MacBook Pro (FGG3TI4)  ─┐
Mac mini (36L7BA2)            ├──→  Unraid (JCN427H)  ─→  svr003 (KY6BY3S)
scc_contabo (RU4B6FL)              ─┘                              receiveonly
                                    sendreceive (relay)
Unraid (JCN427H) ───────────────────────────────────────→  svr003 (KY6BY3S)
                                                              receiveonly (svr001 backups direct)
```

Multiple senders → Unraid relays → svr003 receives. svr003 is purely backup — no writes propagate upstream.

### Folders on svr003
| Folder ID | Path | Source | Type |
|---|---|---|---|
| `pictures` / `documents` / `3dprinting` / `scc_2026` / `ssd_photos` | `/mnt/backup/<name>` | MacBook Pro | sendonly chain |
| `backups-mac-mini` | `/mnt/backup/mac-mini` | Mac mini | openclaw target |
| `backups-scc_contabo` | `/mnt/backup/scc_contabo` | scc_contabo | geekythings, gym-coach, loop-coach, brickswap, ollama, hello-world (plane decommissioned 2026-05-06) |
| `backups-svr001` | `/mnt/backup/svr001` | Unraid (svr001) | flash, appdata, docker-state, timescale |

## Recovery sequence (INFRA-2)

The off-site copy at svr003 is the last line of defence. If both the source host and Unraid are lost (fire/flood/site-wide), restore from svr003.

### Restore an openclaw snapshot to a fresh Mac mini
```bash
# 1. Copy the most recent tarball from svr003 to scratch
ssh svr003 'ls -1t /mnt/backup/mac-mini/openclaw/*/openclaw-full.tar.gz | head -1' \
  | xargs -I {} scp svr003:{} /tmp/openclaw-restore.tar.gz

# 2. Verify sha256 against the manifest
ssh svr003 'cat /mnt/backup/mac-mini/openclaw/$(ls /mnt/backup/mac-mini/openclaw | sort | tail -1)/manifest.txt'
shasum -a 256 /tmp/openclaw-restore.tar.gz

# 3. Extract into ~/.openclaw
mkdir -p ~/.openclaw
tar -C ~/.openclaw -xzf /tmp/openclaw-restore.tar.gz

# 4. Reinstall openclaw + start gateway
npm install -g openclaw@latest
openclaw gateway install && openclaw gateway start
openclaw doctor --fix
```

### Restore TimescaleDB on svr001 (or fresh Unraid)
```bash
# 1. Pull dump from svr003
ssh svr003 'ls -1t /mnt/backup/svr001/timescale/*/timescale-all.sql.gz | head -1' \
  | xargs -I {} scp svr003:{} /tmp/timescale-restore.sql.gz

# 2. Stop the timescaledb container, wipe + recreate volume (DESTRUCTIVE)
ssh svr001 'docker stop timescaledb'
# Remove the appdata dir contents OR mount a fresh one, then start
ssh svr001 'docker start timescaledb'

# 3. Restore
gunzip -c /tmp/timescale-restore.sql.gz \
  | ssh svr001 'docker exec -i -e PGPASSWORD=$POSTGRES_PASSWORD timescaledb psql -U metrics -d postgres'
```

### Restore a Postgres dump from scc_contabo (geekythings worked example — verified 2026-05-19)

This is the canonical drill for any pg_dump target on scc_contabo. The flow is the same for `geekythings.sql.gz` (live), or future targets like `loop-coach.sql.gz` if/when a pg_dump is added.

```bash
# 1. Pull from svr003 → Mac scratch (scc_contabo ↔ svr003 SSH is not configured; route via Mac)
DRILL=/tmp/restore-drill-$(date +%Y%m%d-%H%M%S); mkdir -p "$DRILL"
scp svr003:/mnt/backup/scc_contabo/geekythings/$(ssh svr003 'ls /mnt/backup/scc_contabo/geekythings/ | sort | tail -1')/geekythings.sql.gz "$DRILL/"

# 2. Verify integrity against the source on scc_contabo (skip if scc_contabo is the lost host)
SHA_OFFSITE=$(shasum -a 256 "$DRILL/geekythings.sql.gz" | awk '{print $1}')
SHA_SOURCE=$(ssh scc_contabo "sha256sum /home/robbohomebot/backups/geekythings/\$(ls /home/robbohomebot/backups/geekythings/ | grep '^2' | sort | tail -1)/geekythings.sql.gz | awk '{print \$1}'")
[ "$SHA_OFFSITE" = "$SHA_SOURCE" ] && echo "✓ offsite copy matches source" || echo "✗ drift — investigate"

# 3. Relay to scc_contabo (or wherever you're restoring to)
scp "$DRILL/geekythings.sql.gz" scc_contabo:/tmp/

# 4. Spin a throwaway Postgres container, restore, verify
ssh scc_contabo 'docker run -d --name restore-drill-pg \
    -e POSTGRES_PASSWORD=drill -e POSTGRES_DB=geekythings_restored -e POSTGRES_USER=postgres \
    -p 35432:5432 postgres:16-alpine
  for i in $(seq 1 30); do docker exec restore-drill-pg pg_isready -U postgres >/dev/null 2>&1 && break; sleep 1; done
  zcat /tmp/geekythings.sql.gz | docker exec -i restore-drill-pg psql -U postgres -d geekythings_restored

  # Row count compare against live source
  docker exec restore-drill-pg psql -U postgres -d geekythings_restored -t -A -F"|" -c \
    "SELECT relname, n_live_tup FROM pg_stat_user_tables WHERE n_live_tup > 0 ORDER BY n_live_tup DESC LIMIT 10;"
  PG_USER=$(docker exec geekythings-db sh -c "echo \$POSTGRES_USER")
  PG_DB=$(docker exec geekythings-db sh -c "echo \$POSTGRES_DB")
  docker exec geekythings-db psql -U "$PG_USER" -d "$PG_DB" -t -A -F"|" -c \
    "SELECT relname, n_live_tup FROM pg_stat_user_tables WHERE n_live_tup > 0 ORDER BY n_live_tup DESC LIMIT 10;"

  # Cleanup
  docker stop restore-drill-pg && docker rm restore-drill-pg
  rm /tmp/geekythings.sql.gz'
```

Real restores (not drills) replace step 4's throwaway container with the actual target container, the actual target DB name, and the source POSTGRES_USER/PASSWORD/DB — and you'd want to stop writes to that target during the replay.

### Restore Unraid `/boot/config` (full Unraid OS config — last-resort scenario)
```bash
# Copy from svr003
ssh svr003 'ls -1t /mnt/backup/svr001/flash/*/flash-config.tar.gz | head -1' \
  | xargs -I {} scp svr003:{} /tmp/flash-restore.tar.gz

# On a fresh Unraid USB (booted into the trial OS):
# 1. Mount the USB
# 2. tar -C /boot -xzf /tmp/flash-restore.tar.gz   # restores config/
# 3. Reboot
```

### Quarterly drill (INFRA-2 op habit, not code)
Pick one target per source host. Pull from svr003 to a scratch dir. Verify the manifest sha256 matches. Extract or open. Confirm content is intact. Don't restore over live data — drill should be non-destructive.

### Verified drills
| Date | Target | Result |
|---|---|---|
| 2026-05-19 | scc_contabo / geekythings pg_dump | sha256 match svr003 ↔ scc_contabo source ↔ Mac relay (`5b9a170e…3ea35`). Restore into throwaway `postgres:16-alpine` clean. Row counts identical: products=193, ukca_documents=16, product_pricing=13, events=3, supplies=2. |

## Related Skills
- `skills/scc_contabo/SKILL.md` — primary home server
- `skills/server-bootstrap/SKILL.md` — server setup reference
- `skills/docker-management/SKILL.md` — Unraid Syncthing container is `lscr.io/linuxserver/syncthing`, managed via dockerman
