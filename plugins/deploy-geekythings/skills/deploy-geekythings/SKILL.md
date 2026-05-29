---
description: Deploy GeekyThings product catalogue to scc_contabo via CI/CD — bump version, push, and monitor the GitHub Actions workflow.
allowed-tools: Bash(git*) Bash(gh*) Bash(make*)
---

# Deploy GeekyThings to scc_contabo

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

## Check logs on scc_contabo
```bash
make logs
```

## Verify live
```bash
curl http://161.97.66.102:3002/
```

## Roll back to a previous version
```bash
ssh scc_contabo
cd ~/data/geekythings
VERSION=1.x.x docker compose -f docker-compose.prod.yml up -d
```

---

## Migrating to a new server

### 1. Dump the PostgreSQL database
```bash
# From old server (adjust container name as needed)
ssh -p 2223 root@192.168.1.200 "docker exec geekythings-db-1 pg_dump -U geekythings geekythings" > /tmp/geekythings-dump.sql

# Strip any Unraid SSH wrappers if coming from svr001
grep -v '\\restrict\|\\unrestrict' /tmp/geekythings-dump.sql > /tmp/geekythings-clean.sql
```

### 2. Copy the Products directory
```bash
# From Unraid svr001 (large — ~10GB)
rsync -az -e "ssh -p 2223" root@192.168.1.200:/mnt/user/data/geekythings/Products/ /tmp/geekythings-products/

# Then to new server
rsync -az /tmp/geekythings-products/ scc_contabo:/home/robbohomebot/data/geekythings/Products/
```

### 3. Create data directories and start Postgres on new server
```bash
ssh scc_contabo 'mkdir -p /opt/stacks/geekythings/Products /opt/stacks/geekythings/db'

# Temporarily start just the DB to restore into
ssh scc_contabo 'cd ~/data/geekythings && docker compose -f docker-compose.prod.yml up -d db'
sleep 5
```

### 4. Restore the database
```bash
ssh scc_contabo 'docker exec -i geekythings-db psql -U geekythings -d geekythings' < /tmp/geekythings-clean.sql
```

### 5. Set GitHub Secret for Postgres password
```bash
gh secret set POSTGRES_PASSWORD --repo robinsondan87/GeekyThingsProductCatalogue --body "geekythings"
```

### 6. Register a new self-hosted runner
See `skills/register-runner/SKILL.md` for full steps.
```bash
# Get registration token
gh api repos/robinsondan87/GeekyThingsProductCatalogue/actions/runners/registration-token --method POST --jq '.token'
# Runner name: scc_contabo-geekythings
# Directory: /opt/runners/geekythings/
```

### 7. Deploy via CI/CD
```bash
make bump-patch
git push && git push --tags
```

### 8. Cloudflare tunnel route
Add to tunnel ingress (via API or dashboard):
- Hostname: `geekythings.robbohome.com`
- Service: `http://<NEW_SERVER_IP>:3002`

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
| Internal URL | http://161.97.66.102:3002 |
| Port | 3002 (container port 8555) |
| GHCR image | `ghcr.io/robinsondan87/geekythings` |
| Data volumes | `/opt/stacks/geekythings/Products/` (~10GB), `/opt/stacks/geekythings/db/` |
| DB creds | geekythings / geekythings / geekythings |
| ZT App ID | `119c16e9-eeab-48af-ba7c-51e970ba1a34` |
| Token bypass ZT App | `45000b6f-89c1-4583-8e10-6c305815a4ac` (for /files-token/ — no longer needed) |
| Runner name | `scc_contabo-geekythings` |
| Runner dir | `/opt/runners/geekythings/` on scc_contabo |

## Notes
- Data volumes at `/opt/stacks/geekythings/` on scc_contabo — never touched by deploys
- Auth: username/password auth enabled via AUTH_USERNAME/AUTH_PASSWORD env vars
- See `skills/bambu-integration/SKILL.md` before attempting any Bambu Studio integration
