---
description: cicd skill for RobboHome automation.
---

# Skill: CI/CD Pipeline

## Overview
Push to main → GitHub Actions on svr002 runner → builds Docker image → pushes to ghcr.io → deploys on svr002

## Repos
- App code: robbohome-hello-world (template for all projects)
- Runner: robbohome-server (labels: robbohome, homeserver), Default group

## Deploy a new version
```bash
cd ~/data/projects/PROJECT_NAME
make bump-patch     # or bump-minor / bump-major
git push && git push --tags
```

## Trigger manual deploy (no version bump)
```bash
gh workflow run deploy.yml --repo robinsondan87/REPO_NAME
```

## Watch a running workflow
```bash
gh run watch RUN_ID --repo robinsondan87/REPO_NAME
```

## Check runner status
```bash
gh api repos/robinsondan87/robbohome-hello-world/actions/runners --jq '.runners[]'
```

## Get fresh runner registration token
```bash
gh api repos/robinsondan87/robbohome-hello-world/actions/runners/registration-token --method POST --jq '.token'
```

## GitHub Secrets (set on each new repo)
```bash
source ~/data/config/.secrets
gh secret set CLOUDFLARE_API_TOKEN --repo robinsondan87/REPO --body "$CLOUDFLARE_API_TOKEN"
gh secret set CLOUDFLARE_ACCOUNT_ID --repo robinsondan87/REPO --body "$CLOUDFLARE_ACCOUNT_ID"
gh secret set CLOUDFLARE_TUNNEL_ID --repo robinsondan87/REPO --body "$CLOUDFLARE_TUNNEL_ID"
```

## New project boilerplate
Template at ~/data/infrastructure/templates/node-app/
Placeholders: {{PROJECT_NAME}}, {{GITHUB_USERNAME}}, {{PORT}}

## Known issues / fixes applied
- npm ci → use `npm install --omit=dev` (no package-lock.json)
- Node.js 20 actions deprecated June 2026 — update actions to v5+ when available
- docker-compose.prod.yml must bind to 0.0.0.0 (not 127.0.0.1) for Cloudflare tunnel access
