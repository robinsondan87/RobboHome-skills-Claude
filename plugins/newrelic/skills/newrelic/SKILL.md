---
description: newrelic — New Relic infrastructure agent on Unraid plus account/API key locations.
---

# Skill: New Relic

EU-region account. Infrastructure agent runs as a Docker container on svr001 (Unraid). Slackware is not on New Relic's officially supported OS list, so the host package install isn't an option — `newrelic/infrastructure-bundle` Docker image is the path.

## Account & keys
- **Region**: EU (license key prefix `eu01xx`, agent ships to EU endpoint automatically)
- **Keys** (svr002:`~/data/config/.secrets`, mode 600):
  - `NEW_RELIC_LICENSE_KEY` — Ingest license key, used by agents (`NRIA_LICENSE_KEY` env var)
  - `NEW_RELIC_USER_API_KEY` — User API key (`NRAK-…`) for REST/NerdGraph queries
- **License key vs User API key**: agents need the **License** key. `NRAK-…` keys won't authenticate the infra agent.

Fetch from any host:
```bash
ssh svr002 "grep ^NEW_RELIC_LICENSE_KEY= ~/data/config/.secrets | cut -d= -f2"
```

## Infrastructure agent on Unraid (svr001)
Container: `newrelic/infrastructure-bundle:latest` — bundle includes the on-host integrations (docker, redis, postgres, etc.) so they auto-discover Unraid's containers.

```bash
ssh svr001 "docker run -d --name newrelic-infra \
  --restart unless-stopped \
  --network=host --pid=host --privileged --cap-add=SYS_PTRACE \
  -v /:/host:ro \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e NRIA_LICENSE_KEY=\"\$(ssh svr002 grep ^NEW_RELIC_LICENSE_KEY= ~/data/config/.secrets | cut -d= -f2)\" \
  -e NRIA_DISPLAY_NAME='svr001' \
  -e NRIA_CUSTOM_ATTRIBUTES='{\"role\":\"unraid\",\"environment\":\"home\"}' \
  --label net.unraid.docker.managed=dockerman \
  newrelic/infrastructure-bundle:latest"
```

### Expected (harmless) log warnings on Unraid
- `unable to initialize containerd client` — Unraid uses Docker, not containerd
- `failed to connect to DBus` / `no systemd found` — Unraid is sysvinit-based

These do not affect host metrics, Docker container metrics, or process samples — those all work normally.

### Verify reporting
```bash
ssh svr001 "docker logs newrelic-infra 2>&1 | grep 'connect got id'"
```
A line containing `connect got id agent-guid=… agent-id=…` means New Relic has accepted the license and assigned a GUID. Data appears in the UI within ~1–2 min: **one.eu.newrelic.com → Infrastructure → Hosts** (filter by `role=unraid`).

## Adding more hosts
For svr002 / svr003 / Mac, the same image works — change `NRIA_DISPLAY_NAME` and add appropriate `role=` tag. svr002/svr003 run a real systemd, so the DBus warning won't appear there.

For non-Docker installs (e.g. svr002 native, Mac), the standard package install is fine since both are on supported OSes (Ubuntu/Debian and macOS).

## Related Skills
- `skills/svr002/SKILL.md` — primary home server, secrets live here
- `skills/svr003/SKILL.md` — backup server, would benefit from monitoring when remote
- `skills/docker-management/SKILL.md` — for Unraid container operations
