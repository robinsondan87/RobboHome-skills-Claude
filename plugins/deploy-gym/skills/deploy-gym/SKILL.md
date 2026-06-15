---
name: deploy-gym
description: Deploy Gym Coach AI (gymcoachai monorepo) to the Contabo VPS via push-to-main CI/CD, and monitor the GitHub Actions workflow.
allowed-tools: Bash(git*) Bash(gh*) Bash(ssh*) Bash(curl*) Bash(pnpm*)
---

# Deploy Gym Coach AI

The canonical app is the **`gymcoachai`** monorepo (`robinsondan87/gymcoachai`, local `~/Projects/gymcoachai`), which **replaced** the old single-user `gym-coach` (svr002 — decommissioned 2026-06-07). Deploy is **push-to-`main`**: a self-hosted runner on the **Contabo VPS** rsyncs the source and rebuilds the compose stack. There is no version bump/tag step and no `make` targets — pushing to `main` is the deploy.

> **Deliberate naming asymmetry (don't "fix"):** the GitHub repo + brand + domain are `gymcoachai`, but the **on-box** stack stays `gymcoachshared` — deploy path `/opt/stacks/gymcoachshared`, container `gymcoachshared-app` (port 3011), CI `DEPLOY_PATH=/opt/stacks/gymcoachshared`. Renaming risks the data dir + restic snapshot paths.

## Deploy a change
```bash
cd ~/Projects/gymcoachai
# (optional hygiene) bump the VERSION file: patch for fixes, minor for features
printf '0.2.1\n' > VERSION

# de-risk first — CI runs `next build`, stricter than tsc:
AUTH_SECRET=build-test pnpm --filter gym-coach-app build   # expect exit 0

git add -A && git commit -m "…"   # end with the Co-Authored-By trailer
git push origin main              # ← this triggers the deploy
```
If you add a dependency, commit the updated `pnpm-lock.yaml` (CI installs with `--no-frozen-lockfile`, so a slight drift won't fail the build, but keep it clean).

## Watch the deployment
```bash
RID=$(gh run list --repo robinsondan87/gymcoachai --limit 1 --json databaseId -q '.[0].databaseId')
gh run watch "$RID" --repo robinsondan87/gymcoachai --exit-status
```
A full build+deploy takes ~6 min. Workflow: `.github/workflows/deploy.yml` (`on: push: branches:[main]` + `workflow_dispatch`).

## Verify live
```bash
curl -s -o /dev/null -w '%{http_code}\n' https://gymcoachai.robbohome.com/   # 307 = normal auth redirect
ssh scc_contabo 'cat /opt/stacks/gymcoachshared/VERSION; docker ps --filter name=gymcoachshared-app --format "{{.Status}}"'
```

## How the CI deploy works (deploy.yml)
1. `actions/checkout`.
2. **rsync** the repo into `$DEPLOY_PATH` with plain `--delete` (NOT `--delete-excluded`). `.env` (compose secrets: Google OAuth, OpenRouter, AUTH_SECRET) and `data/` (auth.db + every per-user `gym.db`) are `--exclude`'d, so they are never sent and **never deleted** — they survive every deploy.
3. `docker compose build && docker compose up -d --remove-orphans && docker image prune -f`.

**CI gotcha (already fixed, keep it):** never switch the rsync to `--delete-excluded` — it wiped the excluded `data/` dir (all user DBs) on every deploy.

## Roll back
```bash
ssh scc_contabo 'cd /opt/stacks/gymcoachshared && git -C ~/Projects/gymcoachai log --oneline -5'  # find a good SHA
# Re-deploy a known-good commit by pushing a revert, or check out the SHA on the box and rebuild:
ssh scc_contabo 'cd /opt/stacks/gymcoachshared && docker compose up -d --build'
```
(Source on the box is rsync'd, not a git checkout, so the clean rollback is `git revert <sha> && git push` from the Mac.)

## Key reference

| Item | Value |
|------|-------|
| Local path | `~/Projects/gymcoachai` |
| GitHub repo | `robinsondan87/gymcoachai` (private, self-hosted runner CI) |
| iOS repo | `robinsondan87/gymcoachai-ios` (`~/Projects/gymcoachai-ios`) |
| Deploy trigger | push to `main` |
| Runner / host | self-hosted on **Contabo VPS** (ssh alias `scc_contabo`) |
| On-box path | `/opt/stacks/gymcoachshared` (asymmetry — see above) |
| Container | `gymcoachshared-app`, port **3011** |
| Public URL | https://gymcoachai.robbohome.com (Cloudflare proxied; HTTP origin) |
| App brand | "Gym Coach AI" |
| Per-user data | `/opt/stacks/gymcoachshared/data/users/<userId>/gym.db` (+ `memory.db`); central `auth.db` (accounts only) |
| Build (local de-risk) | `pnpm --filter gym-coach-app build` |
| Secrets | GitHub Secrets on the repo + `.env` on the box (never in git) |

## Notes
- **Never use `confirm()` / `alert()`** in the web app — the iOS WKWebView shell silently swallows them. Use the in-app `ConfirmModal` + inline banners.
- Data dirs + `.env` on the box are never touched by deploys (rsync-excluded).
- Old `gym-coach` / `gym-coach-ai` / svr002 are decommissioned; their GitHub repos are archived (read-only), not deleted. Old domains (`gymcoach.robbohome.com`, `gymcoachshared.robbohome.com`) are removed — only `gymcoachai.robbohome.com` remains.
- iOS deploy is separate: `cd ~/Projects/gymcoachai-ios && make deploy` (fastlane → connected iPhone). `DEVELOPMENT_TEAM` must be `9SF4DS367B` (Dan's paid team — required for HealthKit).
