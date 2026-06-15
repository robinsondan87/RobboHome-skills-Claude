---
name: fleet-onboard
description: Onboard a new host into the RobboHome fleet — SSH, SOPS, New Relic, backup target, periodic config pull. The cross-cutting "what fleet membership means" runbook, separate from OS-specific bootstrap.
---

# Skill: Fleet Onboard

Adds an existing host (Linux server, Mac, or Unraid box that already has an OS + working SSH) into the RobboHome fleet. Bare-metal OS install is out of scope — see `skills/server-bootstrap/SKILL.md` for the scc_contabo-style Ubuntu start.

The contract once onboarding is complete:
- Reachable as a named SSH alias from every other fleet host that needs it
- Holds a current decrypted copy of `~/data/config` (SOPS secrets refreshed every 15 min)
- Reports to New Relic with trimmed ingest (no process metrics, 30s sample rate)
- Its writable data is captured by a nightly `backup.sh` → Syncthing → Unraid → svr003 fanout
- Listed in the svr003 SKILL.md topology table so a future restore knows the host exists

Skip individual sections only when the host's role makes them irrelevant (e.g. an SCC-only VPS that needs no fleet secrets — skip SOPS + backup).

## Layer 1 — Tailscale

If the host isn't already on the tailnet, enrol it:
```bash
# Linux
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname=<short-name>

# Unraid: Tailscale plugin via Community Apps
# Mac: brew install --cask tailscale, then sign in via menu bar
```

Pick a tag matching the host's role (e.g. `tag:home-server`, `tag:backup`, `tag:scc-prod`). Set in the Tailscale admin policy so ACLs apply automatically.

Record the Tailscale IP and add it to `skills/svr003/SKILL.md` → "Tailscale network" table.

## Layer 2 — SSH

### 2a. Add a Host block to the shared SSH config

Edit `/opt/stacks/config/ssh/config` (the canonical shared file) on the Mac, add a new block:

```
Host <short-name>
  HostName <tailscale-ip-or-LAN-ip>
  User <linux-user-or-root>
  Port 22                          # 2223 on svr001 + svr003
  IdentityFile ~/.ssh/id_ed25519
```

Commit + push to `robbohome-config`. The fleet's 15-min puller (INFRA-16) propagates the new block to every other host automatically — no manual fan-out.

Each host already does `Include /opt/stacks/config/ssh/config` from its `~/.ssh/config` (INFRA-18). If a brand-new host doesn't yet, add that line first.

### 2b. Install the shared SSH key

On the new host, after `~/data/config` is in place (Layer 3):

```bash
bash /opt/stacks/config/install-ssh-keys.sh
```

Idempotent. Decrypts the SOPS-encrypted private keys into `~/.ssh/`, copies pubkeys alongside, and appends the shared pubkey to `~/.ssh/authorized_keys` only if not already present (compares by key body, not by comment).

After this, every other fleet host that holds the shared private key can SSH in by name.

## Layer 3 — SOPS + age

### 3a. Install sops + age binaries

```bash
# Linux (apt)
sudo apt-get install -y age
curl -L https://github.com/getsops/sops/releases/latest/download/sops-v3.10.2.linux.amd64 -o /tmp/sops
sudo install -m 0755 /tmp/sops /usr/local/bin/sops

# Linux arm64 — substitute .linux.arm64 in the URL above

# Mac
brew install sops age

# Unraid — the binaries are checked into /boot/config/sops/ already.
# /boot/config/go installs them to /usr/local/bin/ at boot.
```

### 3b. Place the age key

The age secret key (only known to the household) goes at `~/.config/sops/age/keys.txt` with mode 600. For a brand-new host, copy it from another fleet host:

```bash
mkdir -p ~/.config/sops/age
scp <existing-host>:~/.config/sops/age/keys.txt ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

(On Unraid the canonical key lives in `/boot/config/sops/keys.txt` and is symlinked into `/root/.config/sops/age/keys.txt` by `/boot/config/go`.)

### 3c. Clone `robbohome-config`

```bash
mkdir -p ~/data
git clone git@github.com:robinsondan87/robbohome-config.git ~/data/config
source /opt/stacks/config/load-secrets.sh
echo "${JIRA_BASE_URL:-MISSING}"   # smoke test — should print the Jira URL
```

If the clone fails with permission errors, the shared SSH key hasn't been installed yet — back up to Layer 2b.

### 3d. Periodic config pull (INFRA-16 pattern)

Keeps SOPS secrets fresh as Dan pushes updates from the Mac.

**Linux (systemd-user):**
```bash
sudo loginctl enable-linger $(whoami)
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/sops-config-pull.service <<'UNIT'
[Unit]
Description=Pull latest robbohome-config (SOPS-encrypted secrets) — INFRA-16
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=%h/data/config
ExecStart=/usr/bin/git fetch --quiet origin main
ExecStart=/usr/bin/git pull --ff-only origin main
StandardOutput=journal
StandardError=journal
UNIT

cat > ~/.config/systemd/user/sops-config-pull.timer <<'TIMER'
[Unit]
Description=Run sops-config-pull every 15 minutes — INFRA-16

[Timer]
OnBootSec=2min
OnUnitActiveSec=15min
Persistent=true

[Install]
WantedBy=timers.target
TIMER

systemctl --user daemon-reload
systemctl --user enable --now sops-config-pull.timer
systemctl --user list-timers sops-config-pull.timer
```

**Unraid:** write the puller to `/boot/config/custom/sops-config-pull.sh`, add a `*/15 * * * * /bin/bash /boot/config/custom/sops-config-pull.sh` line to root's crontab, and add an idempotent re-installer stanza to `/boot/config/go` so it survives reboot. `/boot` is FAT — never rely on the exec bit; always invoke via `/bin/bash`.

**Mac:** intentionally NOT done — the Mac is the write side. A puller there would race against in-flight commits.

## Layer 4 — New Relic infra agent

Install per the OS, then drop the trimmed config (no process metrics, 30s sample rates — see `reference_newrelic_ingest_trim` memory).

```bash
# Linux apt — see skills/newrelic/SKILL.md for the exact key/install sequence
# Debian Trixie arm64 needs the [trusted=yes] workaround.

source /opt/stacks/config/load-secrets.sh

sudo tee /etc/newrelic-infra.yml >/dev/null <<EOF
license_key: $NEW_RELIC_LICENSE_KEY
display_name: <short-name>
custom_attributes:
  role: <role-tag>
  environment: home

# Ingest reduction (fleet-wide pattern, 2026-05-17)
enable_process_metrics: false
metrics_system_sample_rate: 30
metrics_network_sample_rate: 30
metrics_storage_sample_rate: 30
EOF

sudo systemctl restart newrelic-infra
sudo systemctl is-active newrelic-infra
```

For Unraid, use the Docker bundle with `NRIA_ENABLE_PROCESS_METRICS=false` + the three `NRIA_METRICS_*_SAMPLE_RATE=30` env vars at container start.

For Mac, the equivalent YAML lives at `/opt/homebrew/etc/newrelic-infra/newrelic-infra.yml`.

After ~5 minutes verify the host appears:
```bash
source /opt/stacks/config/load-secrets.sh
curl -s -X POST https://api.eu.newrelic.com/graphql \
  -H "API-Key: $NEW_RELIC_USER_API_KEY" -H 'Content-Type: application/json' \
  -d '{"query":"{actor{account(id:4304361){nrql(query:\"SELECT latest(timestamp) FROM SystemSample SINCE 10 minutes ago FACET hostname\"){results}}}}"}' \
  | jq '.data.actor.account.nrql.results'
```

## Layer 5 — Backup target

Only if the host has writable data worth fanning out (scc_contabo, the Mac, Unraid itself — not single-purpose VPSes).

### 5a. Install Syncthing

```bash
# Linux (Docker)
docker run -d --name syncthing --restart unless-stopped \
  --hostname=<short-name>-syncthing \
  --memory 512m --cpus 2.0 \
  -e PUID=$(id -u) -e PGID=$(id -g) \
  -p 8384:8384 -p 22000:22000/tcp -p 22000:22000/udp -p 21027:21027/udp \
  -v $HOME/.config/syncthing:/var/syncthing/config \
  -v $HOME/backups:/var/syncthing/backups \
  lscr.io/linuxserver/syncthing:latest

# Mac
brew install syncthing
brew services start syncthing
```

Pair the device against Unraid via the Syncthing GUI (LAN URL `http://192.168.1.200:3030` for Unraid's instance — wait, that's Grafana; Syncthing is on `http://192.168.1.200:8384`). Pin the device's Tailscale IP in the connection so it survives the host moving off-LAN.

### 5b. Wire the backup folders

| Side | Folder ID | Path | Type |
|---|---|---|---|
| New host | `backups-<short-name>` | `~/backups/` | sendonly |
| Unraid | `backups-<short-name>` | `/data-backups/<short-name>/` | sendreceive |
| svr003 | `backups-<short-name>` | `/mnt/backup/<short-name>/` | receiveonly |

Use the same folder ID across all three so the syncs link automatically.

### 5c. Write `backup.sh` + schedule

Copy `/opt/stacks/scripts/backup.sh` from scc_contabo (or `~/.openclaw/scripts/backup.sh` from the Mac) as a starting template. The contract per target:
- Output `~/backups/<target>/<DATE>/`
- 7-day retention via `prune_target`
- Wrap target invocation in a subshell: `if ( "$fn" ); then ...` — so a failed target's `die`/`exit` doesn't kill the orchestrator (see "Known gotcha" in [[reference_backup_architecture]])

Schedule nightly:
- **Mac**: LaunchAgent `~/Library/LaunchAgents/ai.openclaw.backup.plist`, 02:30 local
- **Linux**: crontab `15 2 * * * /bin/bash /opt/stacks/scripts/backup.sh all`
- **Unraid**: root crontab + persistence in `/boot/config/go`

### 5d. Verify fanout end-to-end

```bash
/opt/stacks/scripts/backup.sh all              # manual fire
ls ~/backups/                             # confirm dated dirs created
# Wait ~2 min for Syncthing
ssh svr001 'ls /mnt/user/backups/<short-name>/'
ssh svr003 'ls /mnt/backup/<short-name>/'
```

Then run a restore drill on one target — the canonical pattern is in `skills/svr003/SKILL.md` → "Restore a Postgres dump from scc_contabo (geekythings worked example)".

## Layer 6 — Update fleet documentation

After everything above lands:

1. **`skills/svr003/SKILL.md`** — add the host to:
   - the "Tailscale network" table (host + Tailscale IP)
   - the "Folders on svr003" table (`backups-<short-name>`)
   - the "Cluster topology" diagram if it changes the shape
2. **`skills/newrelic/SKILL.md`** — add a row to the Hosts table (host + role tag + install method)
3. **`MEMORY.md`** — only if there's a gotcha worth remembering (Trixie GPG issue, FAT no-exec-bit, etc.). Don't pollute with routine notes.

## Decommissioning (mirror of the onboard)

If a host is being retired, walk the layers in reverse: stop backup.sh + delete LaunchAgent/timer; disable Syncthing folders; remove host block from `/opt/stacks/config/ssh/config`; stop NR agent; remove the SSH Host block from `/opt/stacks/config/ssh/config`; remove from `skills/svr003/SKILL.md` tables. Last: `tailscale logout` on the host, then remove from Tailscale admin.

## Lessons baked into this skill

- **Don't trust the orchestrator's "ok" exit code** without checking per-target output. scc_contabo's backup ran "successfully" for 13 days while only producing failed `plane` snapshots (2026-05-19 — see [[reference_backup_architecture]]).
- **15 min is the right sync cadence** for SOPS secrets — long enough that hot-pushes don't thrash, short enough that a same-day push reaches every host before EOD.
- **Tailscale ACLs gate fleet→VPS SSH** by default. Set up an ACL rule explicitly if you need svr→VPS without per-call browser check-in.
- **Always invoke shell scripts via `bash <path>` on Unraid** — `/boot` is FAT and won't preserve the exec bit, so direct `/path/to/script.sh` execution fails.
- **Process metrics are 51% of New Relic's free-tier bill** for a Docker-heavy fleet. Default to off, opt in per-host if you actually need them.

## Related skills

- `skills/newrelic/SKILL.md` — agent install per OS, keys, EU endpoint
- `skills/secrets/SKILL.md` — SOPS + age workflow, key rotation
- `skills/svr003/SKILL.md` — offsite topology + canonical restore pattern
- `skills/server-bootstrap/SKILL.md` — Ubuntu bare-metal start (separate concern)
- `skills/docker-management/SKILL.md` — Syncthing, NR bundle on Unraid

## Related memories

- `reference_backup_architecture` — current state of the 3-tier fanout
- `reference_newrelic_ingest_trim` — why the agent ships with process metrics off
- `reference_linkedin_health_check` — pattern for soft-failure watchdogs (not host setup, but same fleet hygiene)
