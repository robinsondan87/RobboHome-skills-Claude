---
description: Deploy gym-coach to svr002 via CI/CD — bump version, push, and monitor the GitHub Actions workflow.
allowed-tools: Bash(git*) Bash(gh*) Bash(make*)
---

# Deploy Gym Coach to svr002

## Patch release (bug fix)
```bash
cd /Users/robbohomebot/gym-coach
make bump-patch
git push && git push --tags
```

## Minor release (new feature)
```bash
make bump-minor
git push && git push --tags
```

## Watch the deployment
```bash
gh run watch --repo robinsondan87/gym-coach
```

## Check logs on svr002
```bash
make logs
```

## Verify live
```bash
curl https://gymcoach.robbohome.com/api/health
```

## Roll back to a previous version
```bash
ssh robbohome-server
cd ~/data/gym-coach
VERSION=1.x.x docker compose -f docker-compose.prod.yml up -d
```

---

## Nightly DB backup

Backups run at 2am daily via cron on svr002, pushed to `robinsondan87/gym-coach-backup` (private).

**Check last backup:**
```bash
ssh robbohome-server 'tail -5 ~/data/gym-coach/backup.log'
```

**Run a manual backup:**
```bash
ssh robbohome-server 'bash ~/data/gym-coach/backup.sh'
```

**Restore from backup:**
```bash
ssh robbohome-server '
docker stop gym-coach
cp ~/data/gym-coach-backup/gym-latest.db ~/data/gym-coach/data/gym.db
chmod 666 ~/data/gym-coach/data/gym.db
docker start gym-coach
'
```

### Re-installing backup on a new server
1. Clone backup repo: `git clone https://github.com/robinsondan87/gym-coach-backup.git ~/data/gym-coach-backup`
2. Install sqlite3: `sudo apt-get install -y sqlite3`
3. Copy backup script from gym-coach repo (`backup.sh`) into `~/data/gym-coach/`
4. Configure git credentials: `git config --global credential.helper store` + add token to `~/.git-credentials`
5. Set up cron: `crontab -e` → add `0 2 * * * /bin/bash /home/robbohomebot/data/gym-coach/backup.sh >> /home/robbohomebot/data/gym-coach/backup.log 2>&1`
6. Fix DB file permissions: `chmod 666 ~/data/gym-coach/data/gym.db*`

### backup.sh content
```bash
#!/bin/bash
set -e
DB_SRC="/home/robbohomebot/data/gym-coach/data/gym.db"
BACKUP_REPO="/home/robbohomebot/data/gym-coach-backup"
DATE=$(date +%Y-%m-%d)
sqlite3 "$DB_SRC" "PRAGMA wal_checkpoint(TRUNCATE);"
cp "$DB_SRC" "$BACKUP_REPO/gym-${DATE}.db"
cp "$DB_SRC" "$BACKUP_REPO/gym-latest.db"
cd "$BACKUP_REPO"
ls -t gym-20*.db 2>/dev/null | tail -n +31 | xargs -r rm
git add -A
git diff --cached --quiet && echo "No changes to backup" && exit 0
git commit -m "backup: ${DATE}"
git push origin main
echo "Backup complete: ${DATE}"
```

---

## Key reference

| Item | Value |
|------|-------|
| Project path | `/Users/robbohomebot/gym-coach` |
| Repo | `robinsondan87/gym-coach` |
| Public URL | https://gymcoach.robbohome.com |
| Internal URL | http://192.168.1.17:3847 |
| Port | 3847 |
| GHCR image | `ghcr.io/robinsondan87/gym-coach` |
| Data volume | `~/data/gym-coach/data/gym.db` on svr002 |
| Backup repo | `robinsondan87/gym-coach-backup` (private, 30 days) |
| Auth | Cloudflare Zero Trust (Google auth) — no APP_PASSWORD |
| Key agent endpoint | `GET /api/coach/session-brief?type=push` |

## Notes
- Data volume at `~/data/gym-coach/data/` on svr002 — never touched by deploys
- DB file permissions must be 666 (container runs as node uid 1000, host user is uid 1001)
- AUTH_SECRET and Cloudflare secrets stored as GitHub Secrets on the repo
