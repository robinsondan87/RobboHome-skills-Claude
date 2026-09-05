---
name: deploy-geekythings
description: Deploy GeekyThings Product Manager to svr001 via tagged CI/CD releases, with image builds on the Contabo runner and deployment on the dedicated local runner.
allowed-tools: Bash(git*) Bash(gh*) Bash(make*)
---

# Deploy GeekyThings to svr001

## IMPORTANT: CI/CD only triggers on version tags

The GitHub Actions workflow triggers on `v*` tag pushes only — **not on branch pushes**.
Always use `make bump-patch` / `make bump-minor` which creates the tag automatically.
If VERSION was bumped manually without a tag, create and push the tag explicitly:

```bash
git tag v$(cat VERSION) && git push --tags
```

Without a tag, the code is committed but the site will NOT redeploy.

## Patch release (bug fix)
```bash
cd /Users/robbohomebot/Projects/GeekyThingsProductCatalogue
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
gh run watch --repo robinsondan87/GeekyThingsProductCatalogue
```

## Check logs on svr001
```bash
ssh svr001 'docker logs geekythings-app --tail 100'
```

## Verify live
```bash
ssh svr001 'curl -fsS http://127.0.0.1:3002/api/session'
curl -I https://geekythings.robbohome.com/
```

## Roll back to a previous version
```bash
ssh svr001
cd /mnt/user/appdata/geekythings-prod
VERSION=1.x.x docker compose -f docker-compose.prod.yml up -d
```

---

## Current production topology

- Production stack: `/mnt/user/appdata/geekythings-prod` on `svr001`.
- Product files: `/mnt/user/appdata/geekythings-prod/data/Products`.
- Database: `/mnt/user/appdata/geekythings-prod/data/db`.
- Local app origin: `http://127.0.0.1:3002`.
- Cloudflare Tunnel route: `geekythings.robbohome.com` to `http://127.0.0.1:3002`.
- CI build runner: `vps-scc-geekythings` on Contabo, label `robbohome`.
- CI deploy runner: `svr001-geekythings`, label `geekythings-local`.
- Local runner directory: `/mnt/user/appdata/actions-runner-geekythings`; boot launcher: `/boot/config/custom/start-geekythings-runner.sh`.
- The stopped Contabo stack under `/opt/stacks/geekythings` is an archive/rollback copy. Do not restart it during normal deployment.
- The app joins `metamcp_metamcp-network`; MetaMCP reaches it privately as `http://geekythings-app:8555/api/agent`.
- The `geekythings` MetaMCP endpoint exposes the Product Manager tools. Etsy remains unconfigured until the pending personal app is approved.

---

## Cloudflare Zero Trust access

App ID: `119c16e9-eeab-48af-ba7c-51e970ba1a34`

Current allowed emails:
- 3dlabzuk@gmail.com
- robinsondan87@gmail.com

To add a user:
```bash
source /opt/stacks/config/.secrets
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/access/apps/119c16e9-eeab-48af-ba7c-51e970ba1a34/policies/<POLICY_ID>" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"name":"RobboHome Admins","decision":"allow","precedence":1,"include":[{"email":{"email":"existing@example.com"}},{"email":{"email":"new@example.com"}}]}'
```

---

## Key reference

| Item | Value |
|------|-------|
| Project path | `/Users/robbohomebot/Projects/GeekyThingsProductCatalogue` |
| Repo | `robinsondan87/GeekyThingsProductCatalogue` |
| Public URL | https://geekythings.robbohome.com |
| Internal URL | http://127.0.0.1:3002 on svr001 |
| Port | 3002 (container port 8555) |
| GHCR image | `ghcr.io/robinsondan87/geekythings` |
| Data volumes | `/mnt/user/appdata/geekythings-prod/data/Products/` (~11GB), `/mnt/user/appdata/geekythings-prod/data/db/` |
| DB creds | geekythings / geekythings / geekythings |
| ZT App ID | `119c16e9-eeab-48af-ba7c-51e970ba1a34` |
| Token bypass ZT App | `45000b6f-89c1-4583-8e10-6c305815a4ac` (for /files-token/ — no longer needed) |
| Runner name | `svr001-geekythings` for deploy; `vps-scc-geekythings` for build |
| Runner dir | `/mnt/user/appdata/actions-runner-geekythings/` on svr001 |

## Notes
- Data volumes under `/mnt/user/appdata/geekythings-prod/data/` are never replaced by deploys.
- Auth: username/password auth enabled via AUTH_USERNAME/AUTH_PASSWORD env vars
- See `skills/bambu-integration/SKILL.md` before attempting any Bambu Studio integration
