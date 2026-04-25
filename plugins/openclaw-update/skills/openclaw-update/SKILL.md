---
description: openclaw-update skill for RobboHome automation.
---

# Skill: OpenClaw Update & Backup

Covers: checking for updates, installing, verifying, backing up config to
GitHub, and restoring from catastrophic failure.

**IMPORTANT: Always take a local backup before any config change or update.**

---

## Local quick-backup (run before ANY config change or update)

Creates a timestamped snapshot of `~/.openclaw` in `~/backups/openclaw/`.
Fast, local, no network required. Do this first — every time.

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
DEST=~/backups/openclaw/$STAMP
mkdir -p "$DEST"
cp ~/.openclaw/openclaw.json "$DEST/" 2>/dev/null || true
cp ~/.openclaw/secrets.json  "$DEST/" 2>/dev/null || true
cp ~/.openclaw/cron/jobs.json "$DEST/cron-jobs.json" 2>/dev/null || true
echo "Local backup saved to $DEST"
```

To keep only the 10 most recent local backups:
```bash
ls -dt ~/backups/openclaw/*/ | tail -n +11 | xargs rm -rf
```

---

## Check current version and available updates

```bash
openclaw --version
openclaw status 2>&1 | grep -E "Update|Gateway self"
npm view openclaw dist-tags
```

---

## Update OpenClaw to latest stable

**Step 0 — local backup first (see above)**

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

## Backup config to GitHub

Backs up all essential config, agent identities, workspaces, and credentials
to the private `robbohome-infrastructure` repo for catastrophic recovery.

**Run after every update, or any time you change agent configs.**
**Always do the local quick-backup above first — the GitHub backup is the long-term record.**

```bash
bash ~/data/infrastructure/openclaw-backup/backup.sh
```

What gets backed up:
- `openclaw.json` — main config (all agent definitions, model routing, ACP, etc.)
- `secrets.json` — API keys and credentials
- `cron/jobs.json` — all scheduled jobs
- `agents/*/agent/*.md` and `*.json` — agent identity/config files per agent
- `workspace/agents/*/` — internal workspace agent markdown + memory
- `workspaces/*/` — named workspace markdown, memory, and skills

What is skipped (large, not needed for restore):
- Session transcripts
- Memory vector databases / sqlite
- Logs, media, images
- `.bak` files

---

## Restore from catastrophic failure

Full step-by-step is in:
`~/data/infrastructure/openclaw-backup/RESTORE.md`

### Quick summary

```bash
# 1. Install prerequisites
brew install node
npm install -g openclaw@latest

# 2. Clone infrastructure repo
mkdir -p ~/data
git clone git@github.com:robinsondan87/robbohome-infrastructure.git ~/data/infrastructure

# 3. Restore config
BACKUP="$HOME/data/infrastructure/openclaw-backup/config"
DEST="$HOME/.openclaw"
mkdir -p "$DEST"

cp "$BACKUP/openclaw.json"    "$DEST/openclaw.json"
cp "$BACKUP/secrets.json"     "$DEST/secrets.json"
mkdir -p "$DEST/cron"
cp "$BACKUP/cron-jobs.json"   "$DEST/cron/jobs.json"

# 4. Restore agent identity files
for agent_dir in "$BACKUP/agents"/*/; do
  agent_id=$(basename "$agent_dir")
  mkdir -p "$DEST/agents/$agent_id/agent"
  cp "$agent_dir"*.md   "$DEST/agents/$agent_id/agent/" 2>/dev/null || true
  cp "$agent_dir"*.json "$DEST/agents/$agent_id/agent/" 2>/dev/null || true
done

# 5. Restore internal workspace agents
for agent_dir in "$BACKUP/workspace-agents"/*/; do
  agent_id=$(basename "$agent_dir")
  mkdir -p "$DEST/workspace/agents/$agent_id"
  cp "$agent_dir"*.md "$DEST/workspace/agents/$agent_id/" 2>/dev/null || true
  [[ -d "$agent_dir/memory" ]] && mkdir -p "$DEST/workspace/agents/$agent_id/memory" && \
    cp "$agent_dir/memory/"*.md "$DEST/workspace/agents/$agent_id/memory/" 2>/dev/null || true
done

# 6. Restore named workspaces
for ws_dir in "$BACKUP/workspaces"/*/; do
  ws_id=$(basename "$ws_dir")
  mkdir -p "$DEST/workspaces/$ws_id"
  cp "$ws_dir"*.md "$DEST/workspaces/$ws_id/" 2>/dev/null || true
  [[ -d "$ws_dir/memory" ]] && mkdir -p "$DEST/workspaces/$ws_id/memory" && \
    cp "$ws_dir/memory/"*.md "$DEST/workspaces/$ws_id/memory/" 2>/dev/null || true
  [[ -d "$ws_dir/skills" ]] && cp -r "$ws_dir/skills" "$DEST/workspaces/$ws_id/skills" || true
done

# 7. Start the gateway
openclaw gateway install
openclaw gateway start
openclaw status

# 8. Run doctor to fix any issues
openclaw doctor --fix
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
| Backup script | `~/data/infrastructure/openclaw-backup/backup.sh` |
| Restore guide | `~/data/infrastructure/openclaw-backup/RESTORE.md` |
| Infra repo (GitHub) | `robinsondan87/robbohome-infrastructure` (cloned to `~/data/infrastructure/`) |
