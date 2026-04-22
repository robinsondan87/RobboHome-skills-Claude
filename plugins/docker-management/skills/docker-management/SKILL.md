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
Key: `~/.ssh/id_ed25519` — configured in `~/.ssh/config` as `svr002 / robbohome-server`.

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
- Credentials: `admin` / see `~/data/config/.secrets` → `PORTAINER_PASS`
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

## Related Skills
- Setting up a new runner: `skills/register-runner/SKILL.md`
- Exposing publicly: `skills/cloudflare-tunnel/SKILL.md`
- Full CI/CD pattern: `skills/deployment-patterns/SKILL.md`
