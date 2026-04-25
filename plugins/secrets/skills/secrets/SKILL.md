---
description: RobboHome shared secrets management — SOPS + age, multi-machine via robbohome-config repo.
---

# Skill: RobboHome Secrets Management

Single source of truth for shared credentials across every machine that runs RobboHome work (Mac, svr002, and up to ~5 hosts). Secrets live SOPS-encrypted in a git repo so they can be safely pushed to GitHub and pulled by any authorised machine.

## Architecture in one paragraph

The `robbohome-config` git repo holds three classes of SOPS-encrypted material: `~/data/config/.secrets.env` (dotenv, for shells/skills), `~/data/config/.secrets.openclaw.json` (JSON, mirrors OpenClaw's `~/.openclaw/secrets.json` shape), and `~/data/config/ssh/*.sops` (binary-mode encrypted SSH private keys, currently `id_ed25519.sops`). Each machine has its own age keypair; each machine's **public** key is listed as a recipient in `.sops.yaml`. All encrypted files are committed and pushed normally. Three consumers:
- Shell skills `source ~/data/config/load-secrets.sh` to get dotenv values into the current shell.
- OpenClaw is fed by `~/data/config/sync-openclaw.sh` which decrypts the JSON file and atomically rewrites `~/.openclaw/secrets.json`, then runs `openclaw secrets reload`.
- New / re-provisioned machines run `bash ~/data/config/install-ssh-keys.sh` to deploy the shared `~/.ssh/id_ed25519` after their first decrypt.

## Key reference

| Item | Value |
|------|-------|
| Config repo | `git@github.com:robinsondan87/robbohome-config.git` |
| Local clone path | `~/data/config` |
| Encrypted dotenv | `~/data/config/.secrets.env` (committed) |
| Encrypted OpenClaw JSON | `~/data/config/.secrets.openclaw.json` (committed) |
| Encrypted SSH keys | `~/data/config/ssh/<name>.sops` (committed; rule path_regex `^ssh/.+\.sops$`) |
| SSH public keys | `~/data/config/ssh/<name>.pub` (committed plain — public keys aren't secret) |
| Recipient list | `~/data/config/.sops.yaml` (committed) |
| Shell helper | `~/data/config/load-secrets.sh` (committed) |
| OpenClaw sync script | `~/data/config/sync-openclaw.sh` (committed) |
| SSH key install helper | `~/data/config/install-ssh-keys.sh` (committed) |
| OpenClaw plaintext (derived) | `~/.openclaw/secrets.json` — overwritten by sync, do not hand-edit |
| Deployed SSH keys (derived) | `~/.ssh/<name>` (mode 600), `~/.ssh/<name>.pub` (mode 644) — written by install-ssh-keys.sh |
| Age key (per machine) | `~/.config/sops/age/keys.txt` (mode 600, **never committed**) |
| Env var | `SOPS_AGE_KEY_FILE` — helper + sync set this if unset |
| Plaintext (legacy/backup) | `~/data/config/.secrets`, `.secrets.bak`, `~/.openclaw/secrets.json.pre-sops-bak`, anything matching `ssh/id_*` that isn't `.pub`/`.sops` — all gitignored |

## How shell skills use it

```bash
source ~/data/config/load-secrets.sh
# now $PORTAINER_PASS, $CLOUDFLARE_API_TOKEN, etc. are exported
```

Skills already on this pattern: `cicd`, `cloudflare`, `cloudflare-tunnel`, `docker-management`, `init-project`, `ollama`, `portainer`. Any new skill that needs a credential should `source` the helper rather than hardcoding or re-reading dotenv files.

## How OpenClaw uses it

OpenClaw doesn't read SOPS directly — it reads `~/.openclaw/secrets.json`. The sync script bridges the two. Edit/refresh flow:

```bash
cd ~/data/config && git pull
sops .secrets.openclaw.json     # opens decrypted in $EDITOR; save re-encrypts
~/data/config/sync-openclaw.sh  # decrypts → ~/.openclaw/secrets.json → openclaw secrets reload
git add .secrets.openclaw.json
git commit -m "feat: <what changed and why>"
git push
```

On any other machine that runs OpenClaw (currently just the Mac, but the encrypted file syncs everywhere): `git pull && ~/data/config/sync-openclaw.sh`.

**Important:** never hand-edit `~/.openclaw/secrets.json` — the next sync will overwrite it. The encrypted file is the source of truth.

## How shared SSH keys are distributed

The user's "fleet" SSH keypair (currently `id_ed25519`, used to ssh into svr001/svr002/svr003 and to talk to GitHub) lives in SOPS as `ssh/id_ed25519.sops` (binary-mode encrypted) plus the corresponding `ssh/id_ed25519.pub` (plain). After a new machine has its age key and has cloned the repo, run:

```bash
bash ~/data/config/install-ssh-keys.sh
```

This decrypts every `ssh/*.sops` file into `~/.ssh/<name>` (mode 600) and copies the matching `.pub` into `~/.ssh/<name>.pub` (mode 644). It overwrites in place — back up any existing identity first if it differs.

**To rotate or add another shared SSH key:**
```bash
cd ~/data/config && git pull
cp /path/to/new/private_key ssh/<name>.sops
chmod 600 ssh/<name>.sops
sops --encrypt --input-type binary --output-type binary --in-place ssh/<name>.sops
cp /path/to/new/public_key ssh/<name>.pub
git add ssh/<name>.sops ssh/<name>.pub
git commit -m "feat: add <name> ssh key"
git push
```

**Bootstrap caveat:** a brand-new machine doesn't yet have any SSH key authorised for GitHub, so the *first* clone of `robbohome-config` has to use HTTPS — typically `gh auth login` then `git clone https://github.com/...`. After step 6 (verify decrypt) the new machine runs `install-ssh-keys.sh`, then re-points the remote to SSH:
```bash
cd ~/data/config && git remote set-url origin git@github.com:robinsondan87/robbohome-config.git
```

---

## Onboarding a new machine

Two-party flow: the new machine self-declares as a candidate (steps 1–4), then **any** already-authed machine grants access (step 5). The Mac is not special after the initial bootstrap — any onboarded host can do step 5.

### Why two parties?
Editing `.sops.yaml` doesn't touch the encrypted file. The data key inside `.secrets.env` is wrapped separately for each existing recipient. To re-wrap it for the new larger recipient list, you need a private key that can already unwrap it — i.e. an existing recipient. So a stolen GitHub token can self-add to `.sops.yaml` but can't read secrets until an existing trusted machine runs `sops updatekeys`.

---

### On the NEW machine

#### 1. Install sops + age
```bash
# macOS
brew install sops age

# Debian/Ubuntu
sudo apt install age
# sops: download latest release binary from https://github.com/getsops/sops/releases
# (apt sops is usually outdated)
```

#### 2. Clone the config repo
A brand-new machine usually has no SSH key authorised for GitHub yet, so bootstrap-clone over HTTPS. Easiest path on macOS / Debian / Ubuntu:
```bash
brew install gh        # or: sudo apt install gh
gh auth login          # GitHub.com → HTTPS → web browser flow
mkdir -p ~/data && cd ~/data
git clone https://github.com/robinsondan87/robbohome-config.git config
cd config
```
After step 6 (verify decrypt) you'll run `install-ssh-keys.sh` to deploy the shared key, then re-point the remote to SSH (see the "Day-to-day workflow" section).

If the new machine already has a working SSH identity authorised on GitHub, you can clone via SSH directly (`git clone git@github.com:robinsondan87/robbohome-config.git config`) and skip the gh auth step.

#### 3. Generate this machine's age keypair
```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

`age-keygen` prints the **public key** (`age1...`). Note it down — you'll add it to `.sops.yaml` next.

#### 4. Self-declare as a candidate recipient
```bash
cd ~/data/config && git pull
$EDITOR .sops.yaml
```

Append the public key to the comma-separated list under `age:`. Existing format:
```yaml
creation_rules:
  - path_regex: \.secrets\.env$
    age: >-
      age1existing...,age1newhost...
```

Then push:
```bash
git add .sops.yaml
git commit -m "chore: propose <hostname> as sops recipient"
git push
```

At this point the new machine still **cannot** decrypt — it's only listed in `.sops.yaml`, not yet wrapped into the data key. Tell whoever is on an already-authed machine to run step 5.

---

### On ANY already-authed machine (grant)

#### 5. Re-wrap the data key for every encrypted file
```bash
cd ~/data/config && git pull
sops updatekeys .secrets.env
sops updatekeys .secrets.openclaw.json
for f in ssh/*.sops; do sops updatekeys "$f"; done
git commit -am "chore: grant <hostname> sops access"
git push
```

`sops updatekeys` reads `.sops.yaml`, unwraps the data key with this machine's existing age key, and re-wraps it for every recipient currently in the list. The encrypted values themselves don't change. **Every encrypted file has its own data key wrapping**, so you must run `updatekeys` on each one — a recipient added but skipped here can decrypt some files but not others.

---

### On the new machine (verify)

#### 6. Pull and confirm
```bash
cd ~/data/config && git pull
source ~/data/config/load-secrets.sh
echo "${PORTAINER_USERNAME:-MISSING}"   # should print, not "MISSING"
```

#### 7. Deploy the shared SSH key (if you bootstrap-cloned over HTTPS)
```bash
bash ~/data/config/install-ssh-keys.sh
cd ~/data/config && git remote set-url origin git@github.com:robinsondan87/robbohome-config.git
ssh -o StrictHostKeyChecking=accept-new -T git@github.com   # accept GitHub host key
git pull --ff-only                                            # confirm SSH works
```

---

## Day-to-day workflow

### Before reading secrets
```bash
cd ~/data/config && git pull   # ensure recipient list + encrypted file are current
```
In practice, skills can `source` the helper without pulling — values rarely change. Pull explicitly when adding/rotating.

### Add or change a shell/skill secret
```bash
cd ~/data/config && git pull
sops .secrets.env             # opens decrypted in $EDITOR; save closes & re-encrypts
git add .secrets.env
git commit -m "feat: add <KEY> for <reason>"
git push
```
Other machines pick it up with `git pull` next time they run the helper.

### Add or change an OpenClaw secret
```bash
cd ~/data/config && git pull
sops .secrets.openclaw.json   # edit the JSON path in $EDITOR; save re-encrypts
~/data/config/sync-openclaw.sh # apply locally + openclaw secrets reload
git add .secrets.openclaw.json
git commit -m "feat: <what changed and why>"
git push
```
On any other Mac that runs OpenClaw: `git pull && ~/data/config/sync-openclaw.sh`.

### Rotate a secret value
Same as add/change for whichever file holds it — open with `sops`, edit the value, save, commit, push (and run the OpenClaw sync if it was the JSON file).

### Remove a recipient (machine retired or key compromised)
```bash
cd ~/data/config && git pull
$EDITOR .sops.yaml            # remove the machine's age public key from EVERY rule
sops updatekeys .secrets.env
sops updatekeys .secrets.openclaw.json
for f in ssh/*.sops; do sops updatekeys "$f"; done
git add .sops.yaml .secrets.env .secrets.openclaw.json ssh/*.sops
git commit -m "chore: remove <hostname> as sops recipient"
git push
```
**Important:** also rotate every secret value (and the SSH keys themselves). The retired machine still has access to historical git versions of every encrypted file and could decrypt them with its old key. Concretely:
- `sops .secrets.env` and `sops .secrets.openclaw.json` — change every value, save.
- For each `ssh/*.sops`: regenerate the keypair (`ssh-keygen -t ed25519 -f /tmp/newkey`), re-encrypt, push, then re-deploy on every machine via `install-ssh-keys.sh`, then update `authorized_keys` on every remote that trusted the old public key (svr001/svr002/svr003, GitHub, etc.). SSH rotation is the heaviest of the three — plan it deliberately.

---

## Hygiene: auditing for stale plaintext

Use this flow when `openclaw secrets audit` reports `PLAINTEXT_FOUND`, or when you find scattered credential files anywhere on the system. Goal: decide per finding whether to **delete** (stale residue) or **migrate** (live value → SecretRef pointing at our SOPS-managed `secrets.json`).

### 1. Always back up first
```bash
bash ~/.openclaw/scripts/backup-to-github.sh ~/data/infrastructure
```
Pushes `~/.openclaw/` to `robbohome-infrastructure`. Reversible point.

### 2. Run the audit
```bash
openclaw secrets audit
```
- `PLAINTEXT_FOUND` in `~/.openclaw/openclaw.json` → typically **live** (provider/skill API keys in active config). Migrate.
- `PLAINTEXT_FOUND` in `~/.openclaw/agents/*/agent/auth-profiles.json` → **often stale** (old provider profiles left over from previous setups). Investigate before assuming.
- `LEGACY_RESIDUE` for OAuth profiles (e.g. `openai-codex:default`) → out of scope per OpenClaw; leave alone.

### 3. For per-agent findings: hash-compare across agents
Determine whether the same token is reused (one secret, many files) or distinct per agent — without printing values:
```bash
find ~/.openclaw/agents -path '*/agent/auth-profiles.json' -type f | while read f; do
  agent=$(echo "$f" | sed 's|.*/agents/||;s|/agent/auth-profiles.json||')
  hash=$(jq -r '.profiles."<PROVIDER>:<MODE>".token // empty' "$f" 2>/dev/null \
         | shasum -a 256 | awk '{print substr($1,1,12)}')
  [ -n "$hash" ] && [ "$hash" != "e3b0c44298fc" ] && echo "$hash  $agent"
done | sort
```
(`e3b0c44298fc` is the sha256 of the empty string — filters out empty-token files. Replace `<PROVIDER>:<MODE>`, e.g. `anthropic:manual`.)

Group by hash: same hash = one shared token; distinct = multiple.

### 4. Decide active vs stale
Active signals (→ migrate to SecretRef):
- Global `~/.openclaw/openclaw.json` `defaults.model.primary` references this provider
- Global `auth.profiles` registers this provider
- The profile entry is fully populated (`access`, `refresh`, `expires`, `accountId`, `managedBy`)
- `openclaw doctor` references this provider as configured/in-use

Stale signals (→ delete):
- Provider not in `defaults.model.primary`
- Provider not in global `auth.profiles`
- Profile is a bare stub: just `{type, provider, token}` with no oauth/expiry fields
- Only mentions outside auth-profiles are in `*.jsonl` / `*.trajectory.jsonl` (those are session logs, not config)

### 5a. Archive-then-delete (stale path)

**Always archive before deleting.** The OpenClaw backup script's targeted globs ignore `~/.openclaw-archive/`, so this directory is safe from being re-pushed to the infra repo. Archives are local-only quick-recovery copies; the authoritative pre-cleanup state lives in git history of `robbohome-infrastructure`.

```bash
PROVIDER="<PROVIDER>:<MODE>"     # e.g. anthropic:manual
TS=$(date -u +%Y-%m-%dT%H%M%SZ)
ARCHIVE="$HOME/.openclaw-archive/auth-profiles-${PROVIDER//:/_}-${TS}"
mkdir -p "$ARCHIVE"
chmod 700 "$HOME/.openclaw-archive" "$ARCHIVE"

find ~/.openclaw/agents -path '*/agent/auth-profiles.json' -type f | while IFS= read -r f; do
  if jq -e --arg p "$PROVIDER" '.profiles | has($p)' "$f" >/dev/null 2>&1; then
    agent=$(echo "$f" | sed 's|.*/agents/||;s|/agent/auth-profiles.json||')
    # Archive the removed entry only (with metadata)
    jq --arg ts "$TS" --arg agent "$agent" --arg key "$PROVIDER" \
       '{archived_at: $ts, source_agent: $agent, removed_profile_key: $key, profile: .profiles[$key]}' \
       "$f" > "$ARCHIVE/${agent}.json"
    chmod 600 "$ARCHIVE/${agent}.json"
    # Now delete from the live file
    tmp="$(mktemp "${f}.XXXXXX")"
    jq --arg p "$PROVIDER" 'del(.profiles[$p])' "$f" > "$tmp" \
      && mv "$tmp" "$f" \
      && echo "archived + cleaned: $agent" \
      || rm -f "$tmp"
  fi
done

echo "Archive: $ARCHIVE"
```

Then re-audit (`openclaw secrets audit`), run `openclaw doctor`, and re-run the backup script to push the cleaned HEAD.

### 5a.i. Recovering from archive (if cleanup needs to be reverted)

```bash
ARCHIVE="$HOME/.openclaw-archive/<archive-folder>"
for arc in "$ARCHIVE"/*.json; do
  agent=$(jq -r .source_agent "$arc")
  key=$(jq -r .removed_profile_key "$arc")
  live="$HOME/.openclaw/agents/$agent/agent/auth-profiles.json"
  [ -f "$live" ] || continue
  tmp="$(mktemp "${live}.XXXXXX")"
  jq --slurpfile arc "$arc" --arg key "$key" \
     '.profiles[$key] = $arc[0].profile' "$live" > "$tmp" \
     && mv "$tmp" "$live" \
     && echo "restored: $agent $key"
done
```

The authoritative pre-cleanup state is also in `git log` of `~/data/infrastructure/openclaw-backup/config/`, so `git show <commit>:openclaw-backup/config/agents/<agent>/auth-profiles.json` is always a fallback.

### 5b. Migrate (active path)
Use OpenClaw's native flow — don't roll your own:
```bash
openclaw secrets configure --agent <id> --plan-out /tmp/plan.json
# review plan, then:
openclaw secrets configure --agent <id> --apply
```
Centralise the value in `~/.openclaw/secrets.json` (which is already SOPS-managed via `~/data/config/.secrets.openclaw.json` + `sync-openclaw.sh`). Each agent file then references it via SecretRef instead of holding the literal.

### 6. After cleanup or migration
- **Revoke any deleted token at the provider's console.** Deletion clears it from disk but the value is still alive in pre-cleanup git history of `robbohome-infrastructure`. Revocation is the only way to neutralise it.
- Run `openclaw secrets audit` again; the relevant findings should be gone.
- `bash ~/.openclaw/scripts/backup-to-github.sh ~/data/infrastructure` to push the cleaned HEAD.

---

## Multi-machine coordination (5 hosts)

### Conflict avoidance
- The encrypted file uses a fresh IV per encryption, so any two parallel edits produce a textual diff that conflicts. Always `git pull` immediately before opening with `sops`.
- If a push is rejected: `git pull --rebase` first. If both sides edited the encrypted file, abort the rebase, redo the edit on the freshly-pulled file, then commit.

### Discovery on a new machine
If a machine has secrets in scattered `.env` files / `~/.bashrc` exports / docker-compose `environment:` blocks that aren't yet in the canonical store, audit and merge before standardising. The cross-machine merge prompts for this live with the user; ask if you need them.

### Consuming on Linux servers (svr002 etc.)
- Same helper, same path — `~/data/config/` clones identically.
- If `$EDITOR` isn't set: `EDITOR=nano sops .secrets.env`.
- For non-interactive scripts (systemd, cron), use `sops exec-env`:
  ```bash
  sops exec-env ~/data/config/.secrets.env 'your-script.sh'
  ```
  This runs the command with the env populated, without writing plaintext to disk.

### Consuming on Unraid (svr001) — RAM-based filesystem

Unraid's `/`, `/root`, `/etc`, and `/usr/local/bin` are all in RAM and reset on every boot. The flash drive at `/boot/config/` is the only writable persistent path that's always available; user shares under `/mnt/user/` are persistent but require the array to be online.

**Persistent layout (already in place on svr001):**

| Resource | Persistent location | Symlink target |
|----------|---------------------|----------------|
| sops, age, age-keygen binaries | `/boot/config/sops/{sops,age,age-keygen}` | `/usr/local/bin/{sops,age,age-keygen}` (re-created at boot) |
| Per-machine age key | `/boot/config/sops/keys.txt` | `/root/.config/sops/age/keys.txt` |
| Cloned config repo | `/mnt/user/appdata/robbohome-config/` | `/root/data/config` |

**Boot persistence** is provided by appending to `/boot/config/go` (the Unraid post-boot hook):
```bash
# RobboHome SOPS persistence
if [ -d /boot/config/sops ]; then
  install -m 0755 /boot/config/sops/sops       /usr/local/bin/sops
  install -m 0755 /boot/config/sops/age        /usr/local/bin/age
  install -m 0755 /boot/config/sops/age-keygen /usr/local/bin/age-keygen
  mkdir -p /root/.config/sops/age /root/data
  ln -sfn /boot/config/sops/keys.txt              /root/.config/sops/age/keys.txt
  ln -sfn /mnt/user/appdata/robbohome-config      /root/data/config
fi
```

**Initial install (idempotent — re-running just refreshes binaries):**
```bash
mkdir -p /boot/config/sops
# age + age-keygen — note --no-same-owner (tarball has uid/gid 1001)
AGE_VER=$(curl -sL https://api.github.com/repos/FiloSottile/age/releases/latest | grep tag_name | head -1 | cut -d'"' -f4)
curl -sL "https://github.com/FiloSottile/age/releases/download/${AGE_VER}/age-${AGE_VER}-linux-amd64.tar.gz" \
  | tar -xz --no-same-owner --strip-components=1 -C /boot/config/sops age/age age/age-keygen
chmod +x /boot/config/sops/{age,age-keygen}
# sops
SOPS_VER=$(curl -sL https://api.github.com/repos/getsops/sops/releases/latest | grep tag_name | head -1 | cut -d'"' -f4)
curl -sL -o /boot/config/sops/sops "https://github.com/getsops/sops/releases/download/${SOPS_VER}/sops-${SOPS_VER}.linux.amd64"
chmod +x /boot/config/sops/sops
# install for current session
install -m 0755 /boot/config/sops/{sops,age,age-keygen} /usr/local/bin/
```

**Gotchas on Unraid:**
- `tmpfs` doesn't always support atomic rename, so `ssh ... -T git@github.com` may print `hostfile_replace_entries: ... Operation not permitted`. The `known_hosts` file IS still written; the warning is benign.
- Don't put the age key in `/root/.config/sops/age/keys.txt` directly — it'll vanish on reboot. Always put it on `/boot/config/` and symlink.
- Don't clone the config repo into `/root/data/config` directly for the same reason — clone into `/mnt/user/appdata/` and symlink.

---

## Gotchas

- **Default age key path on macOS is NOT `~/.config/sops/age/keys.txt`** — sops looks at `~/Library/Application Support/sops/age/keys.txt` by default. The helper exports `SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt` to keep the path consistent across macOS and Linux.
- **The age private key never goes in git.** It's per-machine. Treat it like an SSH private key.
- **Don't commit plaintext.** `.secrets`, `.secrets.bak`, and any `.pre-sops-bak` files are gitignored. The only secrets files that should appear in `git status` are `.secrets.env` and `.secrets.openclaw.json`.
- **Don't hand-edit `~/.openclaw/secrets.json`.** It's regenerated from the encrypted file by `sync-openclaw.sh`. Edit `.secrets.openclaw.json` via `sops`, then run the sync.
- **`sops updatekeys` doesn't change values** — only re-wraps the data key for the current recipient list. Use it after editing `.sops.yaml`.
- **Editing `.sops.yaml` alone changes nothing on disk.** You must run `sops updatekeys` afterwards or new recipients can't decrypt.
- **Don't `cat .secrets.env`** expecting plaintext — values are AES-GCM encrypted; only keys are visible. Use `sops -d .secrets.env` to view.
- **No `bw`/Bitwarden coupling.** Vaultwarden is fine for human-facing passwords (browser, mobile) but is intentionally NOT used for skill secrets — this avoids a runtime network dependency.

## Related skills

- `cicd`, `cloudflare`, `cloudflare-tunnel`, `docker-management`, `init-project`, `ollama`, `portainer` — all consume secrets via the helper.
- `openclaw-update` — OpenClaw lifecycle; OpenClaw secrets are now sourced from this skill's encrypted JSON file via `sync-openclaw.sh`.
- `maintain-skills` — when adding a new skill that needs credentials, document the keys it requires and source the helper (or, for OpenClaw plugins, reference the JSON path).
