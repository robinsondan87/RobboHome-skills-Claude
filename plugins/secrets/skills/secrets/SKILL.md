---
description: RobboHome shared secrets management — SOPS + age, multi-machine via robbohome-config repo.
---

# Skill: RobboHome Secrets Management

Single source of truth for shared credentials across every machine that runs RobboHome work (Mac, svr002, and up to ~5 hosts). Secrets live SOPS-encrypted in a git repo so they can be safely pushed to GitHub and pulled by any authorised machine.

## Architecture in one paragraph

`~/data/config/.secrets.env` is a SOPS-encrypted dotenv file in the `robbohome-config` git repo. Each machine has its own age keypair; each machine's **public** key is listed as a recipient in `.sops.yaml`. The encrypted file is committed and pushed normally. A helper script `~/data/config/load-secrets.sh` decrypts on demand into the current shell. Skills source the helper instead of touching the encrypted file directly.

## Key reference

| Item | Value |
|------|-------|
| Config repo | `git@github.com:robinsondan87/robbohome-config.git` |
| Local clone path | `~/data/config` |
| Encrypted secrets | `~/data/config/.secrets.env` (committed) |
| Recipient list | `~/data/config/.sops.yaml` (committed) |
| Helper | `~/data/config/load-secrets.sh` (committed) |
| Age key (per machine) | `~/.config/sops/age/keys.txt` (mode 600, **never committed**) |
| Helper env var | `SOPS_AGE_KEY_FILE` — helper sets this if unset |
| Plaintext (if any) | `~/data/config/.secrets`, `.secrets.bak` — gitignored, do not commit |

## How skills use it

```bash
source ~/data/config/load-secrets.sh
# now $PORTAINER_PASS, $CLOUDFLARE_API_TOKEN, etc. are exported
```

Skills already updated to this pattern: `cicd`, `cloudflare`, `cloudflare-tunnel`, `docker-management`, `init-project`, `ollama`, `portainer`. Any new skill that needs a credential should `source` the helper rather than hardcoding or re-reading dotenv files.

---

## Onboarding a new machine

Run on the **new** machine first, then come back to a recipient machine to grant access.

### 1. Install sops + age
```bash
# macOS
brew install sops age

# Debian/Ubuntu
sudo apt install age
# sops: download latest release binary from https://github.com/getsops/sops/releases
# (apt sops is usually outdated)
```

### 2. Clone the config repo
```bash
mkdir -p ~/data && cd ~/data
git clone git@github.com:robinsondan87/robbohome-config.git config
cd config
```

### 3. Generate this machine's age keypair
```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

The command prints the **public key** (`age1...`). Capture it — that's what gets shared.

### 4. Add this machine as a recipient (run on a machine that's ALREADY a recipient)
On the existing recipient machine (e.g. the Mac):
```bash
cd ~/data/config
git pull
# Edit .sops.yaml — add the new machine's public key under `age:`
$EDITOR .sops.yaml
# Re-encrypt the data key for the updated recipient list
sops updatekeys .secrets.env
git add .sops.yaml .secrets.env
git commit -m "feat: add <hostname> as sops recipient"
git push
```

### 5. Verify on the new machine
```bash
cd ~/data/config && git pull
source ~/data/config/load-secrets.sh
# Should load without error. Confirm with: echo "${PORTAINER_USERNAME:-MISSING}"
```

---

## Day-to-day workflow

### Before reading secrets
```bash
cd ~/data/config && git pull   # ensure recipient list + encrypted file are current
```
In practice, skills can `source` the helper without pulling — values rarely change. Pull explicitly when adding/rotating.

### Add or change a secret
```bash
cd ~/data/config && git pull
sops .secrets.env             # opens decrypted in $EDITOR; save closes & re-encrypts
git add .secrets.env
git commit -m "feat: add <KEY> for <reason>"
git push
```
Other machines pick it up with `git pull` next time they run the helper.

### Rotate a secret value
Same as add/change — open with `sops`, edit the value, save, commit, push.

### Remove a recipient (machine retired or key compromised)
```bash
cd ~/data/config && git pull
$EDITOR .sops.yaml            # remove the machine's age public key
sops updatekeys .secrets.env
git add .sops.yaml .secrets.env
git commit -m "chore: remove <hostname> as sops recipient"
git push
```
**Important:** also rotate every secret value. The retired machine still has access to historical git versions of `.secrets.env` and could decrypt them with its old key.

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
- **Don't commit plaintext.** `.secrets` and `.secrets.bak` are in `.gitignore`. `.secrets.env` is the only secrets file that should ever appear in `git status`.
- **`sops updatekeys` doesn't change values** — only re-wraps the data key for the current recipient list. Use it after editing `.sops.yaml`.
- **Editing `.sops.yaml` alone changes nothing on disk.** You must run `sops updatekeys` afterwards or new recipients can't decrypt.
- **Don't `cat .secrets.env`** expecting plaintext — values are AES-GCM encrypted; only keys are visible. Use `sops -d .secrets.env` to view.
- **No `bw`/Bitwarden coupling.** Vaultwarden is fine for human-facing passwords (browser, mobile) but is intentionally NOT used for skill secrets — this avoids a runtime network dependency.

## Related skills

- `cicd`, `cloudflare`, `cloudflare-tunnel`, `docker-management`, `init-project`, `ollama`, `portainer` — all consume secrets via the helper.
- `maintain-skills` — when adding a new skill that needs credentials, document the keys it requires and source the helper.
