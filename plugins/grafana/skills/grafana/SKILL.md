---
description: grafana — Grafana on Unraid using New Relic NerdGraph as a data source via the Infinity plugin.
---

# Skill: Grafana

Self-hosted Grafana on svr001 (Unraid). Pulls metrics from New Relic via NerdGraph (no second metrics store). Public access via Cloudflare Tunnel + Cloudflare Access.

## URLs and creds
| | |
|---|---|
| Public URL | https://grafana.robbohome.com (Cloudflare Access gated by Google IDP) |
| LAN URL | http://192.168.1.200:3030 |
| Container port mapping | host 3030 → container 3000 |
| Admin user/pass | `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASS` (SOPS-encrypted in `~/data/config/.secrets.env`, see `skills/secrets/`) |
| Data dir | `/mnt/user/appdata/grafana` (container UID 472) |

## Deploy
```bash
ssh svr001 "mkdir -p /mnt/user/appdata/grafana && chown -R 472:472 /mnt/user/appdata/grafana
docker rm -f grafana 2>/dev/null
docker run -d --name=grafana --restart=unless-stopped \
  -p 3030:3000 \
  -v /mnt/user/appdata/grafana:/var/lib/grafana \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=changeme \
  -e GF_INSTALL_PLUGINS='yesoreyeram-infinity-datasource' \
  -e GF_SERVER_ROOT_URL='https://grafana.robbohome.com' \
  --label net.unraid.docker.managed=dockerman \
  grafana/grafana:latest"
```

**Don't install `grafana-newrelic-datasource`** — it's an Enterprise plugin that needs a paid Grafana license and won't load on free Grafana. Use `yesoreyeram-infinity-datasource` instead (free, queries any HTTP/JSON/GraphQL/CSV/XML, well-maintained).

## Cloudflare Tunnel + Access
The Cloudflare account/tunnel/zone IDs and API token live in the SOPS-encrypted `~/data/config/.secrets.env` (see `skills/secrets/`). Steps to expose any service publicly via the existing tunnel:

```bash
source ~/data/config/load-secrets.sh

# 1. Add tunnel ingress route (fetch current config, insert new route before catch-all, PUT back)
# 2. Create DNS CNAME record: grafana.robbohome.com -> $CLOUDFLARE_TUNNEL_ID.cfargotunnel.com (proxied)
# 3. Create Cloudflare Access app with email policy
```
See `skills/cloudflare/SKILL.md` for the exact API call patterns. Grafana's Access app id was `5169bf32-ce09-4123-a7b4-c5675ef0b636` (subject to change).

## NerdGraph data source (Infinity)
Provision via API. Source secrets via the SOPS helper (see `skills/secrets/`):
```bash
source ~/data/config/load-secrets.sh
# now $GRAFANA_ADMIN_PASS, $NEW_RELIC_USER_API_KEY etc. are exported

curl -s -u "admin:$GRAFANA_ADMIN_PASS" -X POST http://192.168.1.200:3030/api/datasources \
  -H 'Content-Type: application/json' \
  -d "{
    \"name\": \"New Relic (NerdGraph)\",
    \"type\": \"yesoreyeram-infinity-datasource\",
    \"access\": \"proxy\",
    \"url\": \"https://api.eu.newrelic.com/graphql\",
    \"isDefault\": true,
    \"jsonData\": {\"auth_method\": \"none\", \"httpHeaderName1\": \"API-Key\"},
    \"secureJsonData\": {\"httpHeaderValue1\": \"$NEW_RELIC_USER_API_KEY\"}
  }"
```

The `httpHeaderName1` / `secureJsonData.httpHeaderValue1` pair attaches `API-Key: <NRAK-…>` to every outbound request from this datasource. NerdGraph EU endpoint is `https://api.eu.newrelic.com/graphql`.

## Critical gotcha — Infinity body_type for GraphQL

When building a panel that POSTs a GraphQL query, **use `body_type: "graphql"` and `body_graphql_query: "<query>"`**. Don't use `body_type: "raw"` + `body: "..."` — Infinity v3.8 ignores the `body` field for raw mode and sends `Content-Length: 0`, which NerdGraph 400s.

### Wrong (silently sends empty body, 400 from NR)
```json
"url_options": {
  "method": "POST",
  "body_type": "raw",
  "body_content_type": "application/json",
  "body": "{\"query\":\"{ actor { ... } }\"}"
}
```

### Right
```json
"url_options": {
  "method": "POST",
  "body_type": "graphql",
  "body_content_type": "application/json",
  "body_graphql_query": "{ actor { account(id: 4304361) { nrql(query: \"SELECT count(*) FROM SystemSample\") { results } } } }"
}
```

The error `"error while performing the infinity query. unsuccessful HTTP response code\nstatus code : 400 Bad Request"` with no other detail is the classic symptom.

## Diagnose what Infinity actually sends
Point the datasource URL at `https://httpbin.org/anything` temporarily, set `root_selector: "headers"` in the panel, and httpbin echoes the exact request (headers + body) back. Restore the URL once done.

## Starter dashboard
Provisioned dashboard `Home Infrastructure (NR)` at uid `a500e9d5-d384-453d-8128-a8b55414dd89` shows the 5-host fleet (svr001/2/3 + 2 Macs):
- **Hosts** table — name, OS, role, current CPU%, Mem%, last seen
- **CPU % by host** timeseries
- **Memory % by host** timeseries
- **Disk % used (root)** timeseries
- **Network RX/TX** timeseries

Pattern for time-series panels:
- `format: "timeseries"`, `parser: "backend"`
- `root_selector: "data.actor.account.nrql.results"`
- columns: `facet` (host string) + `beginTimeSeconds` (timestamp_epoch_s) + the metric (number)
- NRQL query must include `TIMESERIES 1 minute FACET entityName`

## Common NRQL queries (paste into the `body_graphql_query` template)
- All hosts current state: `SELECT latest(timestamp), latest(role), latest(cpuPercent), latest(memoryUsedPercent) FROM SystemSample SINCE 10 minutes ago FACET entityName`
- Top 5 CPU consumers right now: `SELECT latest(cpuPercent) FROM SystemSample SINCE 5 minutes ago FACET entityName LIMIT 5`
- Disk usage by mount: `SELECT latest(diskUsedPercent) FROM StorageSample SINCE 10 minutes ago FACET entityName, mountPoint`
- Docker containers on Unraid: `SELECT latest(cpuPercent), latest(memoryResidentSizeBytes/1e6) AS mem_mb FROM ContainerSample WHERE hostname = 'svr001' SINCE 5 minutes ago FACET containerName LIMIT 30`
- Process samples (top by CPU): `SELECT average(cpuPercent) FROM ProcessSample SINCE 30 minutes ago FACET entityName, processDisplayName LIMIT 20`

Always wrap in the GraphQL envelope:
```graphql
{ actor { account(id: 4304361) { nrql(query: "<NRQL>") { results } } } }
```

## Related Skills
- `skills/newrelic/SKILL.md` — NR account, license keys, agent installs
- `skills/cloudflare/SKILL.md` — tunnel/DNS/Access API patterns
- `skills/cloudflare-tunnel/SKILL.md` — the tunnel-specific subset
- `skills/docker-management/SKILL.md` — Unraid container ops
