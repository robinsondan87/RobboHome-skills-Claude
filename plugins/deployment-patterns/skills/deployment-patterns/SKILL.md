---
description: deployment-patterns skill for RobboHome automation.
---

# Skill: Deployment Patterns Overview

Two deployment patterns are used across robbohome projects. Pick the right one before starting a new project.

## Pattern A — Server App (Node.js / Docker → svr002)

Used for: APIs, web dashboards, background services.

```
Code change → git push → GitHub Actions → Docker build → ghcr.io → svr002 pull + restart
```

**Requirements:**
- Dockerfile + docker-compose.prod.yml
- `.github/workflows/deploy.yml`
- Self-hosted GitHub Actions runner on svr002
- `make deploy` or `/deploy-gym` skill triggers the pipeline

**Setup steps:**
1. Create GitHub repo
2. Register GitHub Actions runner on svr002 — see `skills/register-runner/SKILL.md`
3. Add `deploy.yml` workflow (use gym-coach as template: `.github/workflows/deploy.yml`)
4. Set GitHub repo secrets: `GHCR_TOKEN`, `SERVER_HOST`, `SERVER_USER`, `APP_PASSWORD`, etc.

**Examples:** gym-coach

---

## Pattern B — iOS App (Swift → iPhone via Fastlane)

Used for: native iPhone apps.

```
Code change → fastlane deploy → Xcode build → IPA → install direct to iPhone via USB/WiFi
```

**Requirements:**
- Xcode installed on Mac Mini
- `brew install fastlane`
- Free Apple Developer account (3dlabzuk@gmail.com)
- Fastlane `Fastfile` + `Appfile`
- `make deploy` builds and installs; `make resign` re-signs every 7 days

**No GitHub Actions runner needed — builds run locally on Mac Mini only.**

**Setup steps:**
1. Create GitHub repo (source control/backup only — no CI/CD)
2. Create Xcode project, add capabilities
3. Set up Fastlane — see `skills/ios-fastlane/SKILL.md`
4. First deploy — see `skills/ios-sideload/SKILL.md`

**Examples:** gym-coach-health-sync

---

## Quick Decision Guide

| Question | Answer |
|---|---|
| Does it run on a server? | Pattern A |
| Does it run on an iPhone? | Pattern B |
| Does it need a GitHub Actions runner? | Pattern A only |
| Does it need Docker? | Pattern A only |
| Does it expire every 7 days? | Pattern B (free account) |
| Does it use Cloudflare tunnel for public access? | Pattern A only |

## Related Skills
- Pattern A setup: `skills/register-runner/SKILL.md`, `skills/server-bootstrap/SKILL.md`
- Pattern B setup: `skills/xcode-setup/SKILL.md`, `skills/ios-sideload/SKILL.md`, `skills/ios-fastlane/SKILL.md`
