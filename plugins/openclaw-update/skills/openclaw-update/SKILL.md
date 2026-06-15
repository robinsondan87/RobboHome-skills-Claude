---
name: openclaw-update
description: openclaw-update skill for RobboHome automation.
---

# Skill: OpenClaw Update & Backup

Covers: checking for updates, installing, verifying, backing up, and restoring from catastrophic failure.

**IMPORTANT: Always take a backup before any config change or update.**

---

## Backup (run before ANY config change or update)

One parametrised script handles everything — same script that nightly cron uses, so manual and scheduled paths can never drift.

```bash
~/.openclaw/scripts/backup.sh openclaw
```

What it does in a single run:
1. **Full snapshot** → `~/backups/openclaw/$DATE/openclaw-full.tar.gz` (~450M)
   - Includes: workspace/, memory/, agents/ (sessions), credentials/, openclaw.json, cron, secrets.json
   - Skips: plugin-runtime-deps, browser, vendor, plugin-runtimes, tmp, logs, _migration_backup_*, *.bak*
   - Manifest with sha256 alongside the tarball
   - 7-day local retention
   - Syncthing fans the snapshot out to Unraid then svr003 automatically — no extra step
2. **Config-only subset** → `~/data/infrastructure/openclaw-backup/config/` then git commit + push
   - Single-file configs + agent identity .md/.json + workspace/named-workspace markdown + memory
   - Skips: session transcripts, vector DBs, logs, media, .bak files
   - Used as the change-history audit trail (diff between commits to see what changed when)

Other invocations:
```bash
~/.openclaw/scripts/backup.sh --list      # show available targets on this host
~/.openclaw/scripts/backup.sh all         # nightly cron firing — runs every target
```

Cron schedule: launchd `ai.openclaw.backup.plist` fires `backup.sh all` daily at 02:30 local. Logs go to `~/Library/Logs/backup.log`.

---

## Check current version and available updates

```bash
openclaw --version
openclaw status 2>&1 | grep -E "Update|Gateway self"
npm view openclaw dist-tags
```

---

## Update OpenClaw to latest stable

**Step 0 — backup first:** `~/.openclaw/scripts/backup.sh openclaw`

```bash
# 1. Install latest via npm (OpenClaw is a global npm package)
npm install -g openclaw@latest

# 2. Verify the binary updated
openclaw --version

# 3. Restart the gateway service to pick up the new version
openclaw gateway restart

# 4. Confirm gateway is running the new version
openclaw status 2>&1 | grep -E "Gateway self|Update"
```

---

## Health check after update

```bash
openclaw status
openclaw gateway status
openclaw doctor 2>&1 | head -40
```

---

## Restore from catastrophic failure

Three tiers of backup are available; pick the highest tier still alive:

| Tier | Source | Best when |
|---|---|---|
| **Mac local** | `~/backups/openclaw/<date>/openclaw-full.tar.gz` | Mac is alive, just rolling back a config change |
| **Onsite secondary** | Unraid `/mnt/user/data/syncthing/backups/mac-mini/openclaw/<date>/` | Mac mini hardware loss / OS reinstall |
| **Off-site** | svr003 `/mnt/backup/mac-mini/openclaw/<date>/` | Both Mac mini and Unraid lost (fire/flood/site-wide) |
| **Config-only history** | git repo `robinsondan87/robbohome-infrastructure` → `openclaw-backup/config/` | Want to inspect a past config or recover an older config without overwriting current data |

### Quick full restore (any tier)

```bash
# 1. Install prerequisites
brew install node
npm install -g openclaw@latest

# 2. Pull the most recent snapshot from whichever tier is alive
#    Example: from Unraid
TARBALL=$(ssh svr001 'ls -1t /mnt/user/data/syncthing/backups/mac-mini/openclaw/*/openclaw-full.tar.gz | head -1')
scp "svr001:$TARBALL" /tmp/openclaw-restore.tar.gz

# 3. Extract into ~/.openclaw
mkdir -p ~/.openclaw && tar -C ~/.openclaw -xzf /tmp/openclaw-restore.tar.gz
echo "verify manifest matches:"
ls /tmp/  # also pull manifest.txt from same dir if available

# 4. Start the gateway
openclaw gateway install
openclaw gateway start
openclaw status
openclaw doctor --fix
```

### Config-only restore (point-in-time inspection)

```bash
git clone git@github.com:robinsondan87/robbohome-infrastructure.git ~/data/infrastructure
git -C ~/data/infrastructure log -- openclaw-backup/config/   # find a known-good commit
git -C ~/data/infrastructure checkout <sha> -- openclaw-backup/config/
# then cherry-pick files into ~/.openclaw as needed
```

---

## Key reference

| Item | Value |
|------|-------|
| Install method | `npm install -g openclaw` |
| Config location | `~/.openclaw/openclaw.json` |
| Gateway service | macOS LaunchAgent, auto-restarts on login |
| Latest version | `npm view openclaw dist-tags.latest` |
| Local changelog | `/opt/homebrew/lib/node_modules/openclaw/CHANGELOG.md` |
| Docs | https://docs.openclaw.ai |
| Backup script | `~/.openclaw/scripts/backup.sh` (parametrised — `openclaw`/`all`/`--list`) |
| Backup schedule | launchd `~/Library/LaunchAgents/ai.openclaw.backup.plist` — daily 02:30 |
| Backup log | `~/Library/Logs/backup.log` |
| Local snapshots | `~/backups/openclaw/$DATE/` (7-day retention) |
| Onsite copy | Unraid `/mnt/user/data/syncthing/backups/mac-mini/` (Syncthing fanout) |
| Off-site copy | svr003 `/mnt/backup/mac-mini/` (Syncthing relay through Unraid) |
| Config-only history | `robinsondan87/robbohome-infrastructure` → `openclaw-backup/config/` |
| Infra repo (GitHub) | `robinsondan87/robbohome-infrastructure` (cloned to `~/data/infrastructure/`) |
