---
description: RobboHome shared secrets management — SOPS + age, multi-machine via robbohome-config repo.
---

# Skill: RobboHome Secrets Management

Single source of truth for shared credentials across every machine that runs RobboHome work (Mac, svr002, and up to ~5 hosts). Secrets live SOPS-encrypted in a git repo so they can be safely pushed to GitHub and pulled by any authorised machine.

## Architecture in one paragraph

The `robbohome-config` git repo holds two SOPS-encrypted secret stores: `~/data/config/.secrets.env` (dotenv, for shells/skills) and `~/data/config/.secrets.openclaw.json` (JSON, mirrors OpenClaw's `~/.openclaw/secrets.json` shape). Each machine has its own age keypair; each machine's **public** key is listed as a recipient in `.sops.yaml`. Both encrypted files are committed and pushed normally. Two consumers:
- Shell skills `source ~/data/config/load-secrets.sh` to get dotenv values into the current shell.
- OpenClaw is fed by `~/data/config/sync-openclaw.sh` which decrypts the JSON file and atomically rewrites `~/.openclaw/secrets.json`, then runs `openclaw secrets reload`.

## Key reference

| Item | Value |
|------|-------|
| Config repo | `git@github.com:robinsondan87/robbohome-config.git` |
| Local clone path | `~/data/config` |
| Encrypted dotenv | `~/data/config/.secrets.env` (committed) |
| Encrypted OpenClaw JSON | `~/data/config/.secrets.openclaw.json` (committed) |
| Recipient list | `~/data/config/.sops.yaml` (committed) |
| Shell helper | `~/data/config/load-secrets.sh` (committed) |
| OpenClaw sync script | `~/data/config/sync-openclaw.sh` (committed) |
| OpenClaw plaintext (derived) | `~/.openclaw/secrets.json` — overwritten by sync, do not hand-edit |
| Age key (per machine) | `~/.config/sops/age/keys.txt` (mode 600, **never committed**) |
| Env var | `SOPS_AGE_KEY_FILE` — helper + sync set this if unset |
| Plaintext (legacy/backup) | `~/data/config/.secrets`, `.secrets.bak`, `~/.openclaw/secrets.json.pre-sops-bak` — gitignored, do not commit |

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
```bash
mkdir -p ~/data && cd ~/data
git clone git@github.com:robinsondan87/robbohome-config.git config
cd config
```

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

#### 5. Re-wrap the data key
```bash
cd ~/data/config && git pull
sops updatekeys .secrets.env
git commit -am "chore: grant <hostname> sops access"
git push
```

`sops updatekeys` reads `.sops.yaml`, unwraps the data key with this machine's existing age key, and re-wraps it for every recipient currently in the list. The encrypted values themselves don't change.

---

### On the new machine (verify)

#### 6. Pull and confirm
```bash
cd ~/data/config && git pull
source ~/data/config/load-secrets.sh
echo "${PORTAINER_USERNAME:-MISSING}"   # should print, not "MISSING"
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
$EDITOR .sops.yaml            # remove the machine's age public key
sops updatekeys .secrets.env
sops updatekeys .secrets.openclaw.json
git add .sops.yaml .secrets.env .secrets.openclaw.json
git commit -m "chore: remove <hostname> as sops recipient"
git push
```
**Important:** also rotate every secret value in both files. The retired machine still has access to historical git versions and could decrypt them with its old key.

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
