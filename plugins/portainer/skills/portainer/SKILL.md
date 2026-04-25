---
description: portainer skill for RobboHome automation.
---

# Skill: Portainer Management

## Credentials
SOPS-encrypted in `~/data/config/.secrets.env`. Load with `source ~/data/config/load-secrets.sh`. Keys:
- PORTAINER_URL=http://svr002:9000
- PORTAINER_API_KEY=ptr_In1ifvRZX/CSAU5fg0iWi2BmciLEVhskwVPsI3z35f0=
- PORTAINER_USERNAME=admin

## Authenticate (get JWT for session)
```bash
curl -s -X POST http://svr002:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"PASSWORD"}'
```

## Common API calls (use X-API-Key header)
```bash
# List containers
curl -s http://svr002:9000/api/endpoints/3/docker/containers/json \
  -H "X-API-Key: {API_KEY}"

# List stacks
curl -s http://svr002:9000/api/stacks \
  -H "X-API-Key: {API_KEY}"

# Restart a container
curl -s -X POST http://svr002:9000/api/endpoints/3/docker/containers/{ID}/restart \
  -H "X-API-Key: {API_KEY}"

# Check endpoint health
curl -s http://svr002:9000/api/endpoints \
  -H "X-API-Key: {API_KEY}"
```

## Notes
- Endpoint ID for svr002 local Docker is 3
- Portainer bound to 0.0.0.0:9000 (required for Cloudflare tunnel from Unraid)
- External access: portainer.robbohome.com (Google auth required)
- Data volume: portainer_data
