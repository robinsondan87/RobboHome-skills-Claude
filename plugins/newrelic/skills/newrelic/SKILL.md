---
description: newrelic — New Relic infrastructure agent on all hosts (Unraid, Ubuntu, Debian arm64, Mac) plus account/API key locations.
---

# Skill: New Relic

EU-region account. Infrastructure agents run on all four hosts: svr001, svr002, svr003, dans-macbook-pro.

## Account & keys
- **Region**: EU (license key prefix `eu01xx`, agent ships to EU endpoint automatically)
- **Account ID**: `4304361` (NerdGraph, NRQL, Grafana data source — all use this)
- **Keys** (svr002:`~/data/config/.secrets`, mode 600):
  - `NEW_RELIC_LICENSE_KEY` — Ingest license key, used by agents (`NRIA_LICENSE_KEY` env var)
  - `NEW_RELIC_USER_API_KEY` — User API key (`NRAK-…`) for REST/NerdGraph queries
- **License key vs User API key**: agents need the **License** key. `NRAK-…` keys won't authenticate the infra agent.

Fetch from any host:
```bash
ssh svr002 "grep ^NEW_RELIC_LICENSE_KEY= ~/data/config/.secrets | cut -d= -f2"
```

## Hosts

| Host | Role tag | Install method |
|---|---|---|
| svr001 (Unraid) | `unraid` | Docker container |
| svr002 (Ubuntu) | `home-server` | apt + systemd |
| svr003 (Debian Trixie arm64) | `backup-server` | apt + systemd, with `trusted=yes` workaround |
| Daniels-MBP.lan | `workstation` | Homebrew + launchd |

### svr001 — Unraid (Docker)
Slackware is not on New Relic's officially supported OS list, so use `newrelic/infrastructure-bundle` (bundle includes on-host integrations: docker, redis, postgres, etc.).

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

Expected (harmless) log warnings: `unable to initialize containerd client` (Unraid uses Docker, not containerd), `failed to connect to DBus` / `no systemd found` (Unraid is sysvinit). Host metrics + Docker container metrics + process samples all work normally despite these.

### svr002 — Ubuntu (apt)
Standard install via the New Relic apt repo:
```bash
ssh svr002 'sudo bash -c "
  curl -fsSL https://download.newrelic.com/infrastructure_agent/gpg/newrelic-infra.gpg | gpg --dearmor -o /etc/apt/trusted.gpg.d/newrelic-infra.gpg
  CODENAME=\$(lsb_release -cs)
  echo \"deb [signed-by=/etc/apt/trusted.gpg.d/newrelic-infra.gpg] https://download.newrelic.com/infrastructure_agent/linux/apt \$CODENAME main\" > /etc/apt/sources.list.d/newrelic-infra.list
  apt-get update -qq && apt-get install newrelic-infra -y
"'
```
Then write `/etc/newrelic-infra.yml`:
```yaml
license_key: <key>
display_name: svr002
custom_attributes:
  role: home-server
  environment: home
```
And `systemctl restart newrelic-infra`.

### svr003 — Debian Trixie arm64 (apt with trusted=yes)
Debian 13 (Trixie) apt rejects New Relic's GPG key because the binding signature uses SHA1 (deprecated in Debian since 2026-02-01). Workaround: use `[trusted=yes]` since the repo is HTTPS-served by a trusted vendor. Also use the `bookworm` codename since `trixie` isn't in NR's apt repo yet.

```bash
ssh svr003 'sudo bash -c "
  echo \"deb [trusted=yes] https://download.newrelic.com/infrastructure_agent/linux/apt bookworm main\" > /etc/apt/sources.list.d/newrelic-infra.list
  apt-get update -qq && apt-get install newrelic-infra -y
"'
```
Same `/etc/newrelic-infra.yml` pattern as svr002 (`display_name: svr003`, `role: backup-server`).

This works for arm64 — New Relic ships arm64 packages in the same apt repo.

### Mac — Homebrew
Use the `newrelic-infra-agent` formula (not `newrelic-cli`, that's a different tool):
```bash
brew install newrelic-infra-agent
mkdir -p /opt/homebrew/etc/newrelic-infra
cat > /opt/homebrew/etc/newrelic-infra/newrelic-infra.yml <<EOF
license_key: <key>
display_name: dans-macbook-pro
custom_attributes:
  role: workstation
  environment: home
EOF
brew services start newrelic-infra-agent
```
Runs as a user-scope launchd service (`~/Library/LaunchAgents/homebrew.mxcl.newrelic-infra-agent.plist`).

## Verify reporting (NerdGraph)
```bash
curl -s -X POST https://api.eu.newrelic.com/graphql \
  -H "API-Key: $(ssh svr002 grep ^NEW_RELIC_USER_API_KEY= ~/data/config/.secrets | cut -d= -f2)" \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ actor { account(id: 4304361) { nrql(query: \"SELECT latest(timestamp) FROM SystemSample SINCE 5 minutes ago FACET hostname\") { results } } } }"}'
```
A row per `hostname` with a recent `latest(timestamp)` confirms each host is shipping. Default harvest cycle is 5s (≈12 samples/min/host).

## UI
- one.eu.newrelic.com → **Infrastructure → Hosts**
- Filter by `role=unraid` / `home-server` / `backup-server` / `workstation`
- Filter by `environment=home`

## Related Skills
- `skills/svr002/SKILL.md` — primary home server, secrets live here
- `skills/svr003/SKILL.md` — backup server (Trixie arm64 — note the GPG workaround)
- `skills/docker-management/SKILL.md` — for Unraid container operations
- `skills/grafana/SKILL.md` — Grafana on Unraid uses NR as a data source
