---
description: docker-management skill for RobboHome automation.
---

# Skill: Docker Management on svr002

Managing Docker containers on svr002 (192.168.1.17).

## Access
```bash
ssh svr002
# or: ssh robbohome-server
```
Key: `~/.ssh/svr002_remote` — configured in `~/.ssh/config` as `svr002 / robbohome-server`.

## Common commands

### View all running containers
```bash
ssh svr002 'docker ps'
```

### View logs
```bash
# Live tail
ssh svr002 'docker logs <container> -f --tail 50'

# Last 100 lines
ssh svr002 'docker logs <container> --tail 100'
```

### Restart a container
```bash
ssh svr002 'docker restart <container>'
```

### Pull latest image and restart (manual deploy)
```bash
ssh svr002 '
  docker pull ghcr.io/robinsondan87/<app>:latest
  cd ~/data/<app>
  docker compose -f docker-compose.prod.yml up -d
'
```

### Disk usage
```bash
ssh svr002 'docker system df'
# Clean up unused images/volumes
ssh svr002 'docker system prune -f'
```

## Portainer (web UI)
- URL: https://portainer.robbohome.com (behind Cloudflare Access)
- Credentials: `admin` / `PORTAINER_PASS` from `source ~/data/config/load-secrets.sh`
- Useful for: viewing container stats, logs, volumes without SSH

## Data volumes on svr002
All app data is stored under `~/data/` on svr002:
```
/home/robbohomebot/data/
├── gym-coach/
│   └── data/gym.db          ← SQLite database
├── geekythings/
│   ├── Products/             ← 3MF files (~10GB)
│   └── db/                  ← PostgreSQL data
└── <other-apps>/
```

## Running apps
| Container | Port | Compose file | Public URL |
|-----------|------|--------------|------------|
| gym-coach | 3847 | ~/data/gym-coach/docker-compose.prod.yml | gymcoach.robbohome.com |
| geekythings | 3002 | ~/data/geekythings/docker-compose.prod.yml | geekythings.robbohome.com |
| plane (12 containers under `plane-app-*`) | 18080 (proxy) | ~/data/plane-selfhost/plane-app/docker-compose.yaml | plane.robbohome.com |

## Plane (project management)
Self-hosted [makeplane/plane](https://github.com/makeplane/plane) — Linear/Jira hybrid for tracking infra TODOs as issues. Stack lives at `~/data/plane-selfhost/` on svr002, started via `./setup.sh` (option 1 = install, 2 = start, 3 = stop).

Stack composition: proxy (`plane-app-proxy-1` listening 18080→80), web/admin/space/live frontends, api + workers + beat-worker + migrator (Django), plane-db (Postgres 15.7), plane-redis, plane-mq (RabbitMQ), plane-minio (S3 for attachments).

**plane.env gotcha**: `DATABASE_URL` defaults to `postgresql://plane:plane@plane-db/plane` (hardcoded creds in `docker-compose.yaml`'s `${DATABASE_URL:-...}` fallback). If you change `POSTGRES_PASSWORD` in `plane.env` you MUST also set `DATABASE_URL` to match — otherwise the API can connect to Postgres but Django uses the stale URL. Easiest: leave `POSTGRES_PASSWORD=plane` (DB is internal-network only, never exposed on the host).

Setup script `>> Plane Server failed to start ❌` is a flaky check that fires even when all 12 containers are healthy — verify with `curl -fsS http://localhost:18080/api/instances/` returning 200, not the script's exit message.

## Related Skills
- Setting up a new runner: `skills/register-runner/SKILL.md`
- Exposing publicly: `skills/cloudflare-tunnel/SKILL.md`
- Full CI/CD pattern: `skills/deployment-patterns/SKILL.md`
