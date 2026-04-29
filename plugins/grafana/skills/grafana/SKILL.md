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

## Templating — host and container variables

Multi-select dropdowns at the top of the dashboard. The custom-variable pattern with `${var:singlequote}` is the easiest way to filter NRQL `IN (…)` clauses:

```json
"templating": {
  "list": [
    {
      "name": "host", "type": "custom",
      "query": "dans-macbook-pro,robbo-mac-mini,svr001,svr002,svr003",
      "includeAll": true, "multi": true
    },
    {
      "name": "container", "type": "query",
      "datasource": {"type": "yesoreyeram-infinity-datasource", "uid": "<ds-uid>"},
      "query": {"queryType": "infinity", "infinityQuery": {
        "type": "json", "source": "url", "format": "table", "parser": "backend",
        "root_selector": "data.actor.account.nrql.results",
        "url_options": {"method":"POST","body_type":"graphql","body_content_type":"application/json",
          "body_graphql_query": "{ actor { account(id: <id>) { nrql(query: \"SELECT uniqueCount(name) FROM ContainerSample SINCE 1 hour ago FACET name LIMIT 100\") { results } } } }"},
        "columns": [{"selector":"facet","text":"name","type":"string"}]
      }},
      "includeAll": true, "multi": true, "refresh": 1
    }
  ]
}
```

Use in NRQL:
```sql
WHERE entityName IN (${host:singlequote})
WHERE hostname IN (${host:singlequote}) AND name IN (${container:singlequote})
```

Don't set `allValue` — without it, `${var:singlequote}` expands "All" to every option quoted, which `IN (…)` accepts directly. With `allValue` set to a sentinel, you have to special-case it.

`SystemSample`/`StorageSample`/`NetworkSample`/`ProcessSample`/`Log` use `entityName`. `ContainerSample` uses `hostname`. NRQL is case-sensitive on field names.

## Dashboard time picker integration

Use Grafana's `${__from}` / `${__to}` (epoch milliseconds) instead of fixed `SINCE 1 hour ago`:
```sql
SELECT … FROM SystemSample WHERE entityName IN (${host:singlequote})
SINCE ${__from} UNTIL ${__to} TIMESERIES AUTO FACET entityName
```
Then the dashboard's top-right time picker drives every panel. `TIMESERIES AUTO` lets NR pick the bucket size.

Keep account-wide queries on fixed ranges (`SINCE this month`, `SINCE 24 hours ago`) — the time picker on those is meaningless.

## Inserting a WHERE clause into NRQL via string-replace — gotcha

Naïve `q.replace("WHERE ", "...")` will hit the **first** `WHERE` in the query, which may be inside `percentage(count(*), WHERE result = 'SUCCESS')` or another nested filter — it will silently destroy the inner filter.

Always anchor against `FROM <EventType>` instead:
```python
q = q.replace("FROM SyntheticCheck", "FROM SyntheticCheck WHERE monitorName LIKE 'RobboHome%'", 1)
```

## Critical gotcha — multi-series timeseries (split by host)

Infinity's backend parser returns one frame containing **all rows mixed together** even when there's a string column like `host`. Grafana's timeseries panel renders that as a single zigzag line jumping between hosts (looks like wild 0→60% oscillation).

Fix: add a Grafana transformation `partitionByValues` on the string column you want to split by:
```json
"transformations": [
  {"id": "partitionByValues", "options": {"fields": ["host"], "keepFields": false, "naming": {"asLabels": false}}}
]
```
Now each unique value of `host` becomes its own series.

For multi-FACET queries (e.g. `FACET entityName, mountPoint`), the `facet` column NR returns is a string like `"svr001,/mnt/cache"` — partitioning on that splits one series per host+mount combo.

## Critical gotcha — dotted field names

NRQL functions return field names like `average.cpuPercent` (literal dot in the key). Infinity's selector treats `.` as JSONPath nesting and silently returns null. **Always alias** in NRQL with `AS`:

```sql
-- bad: selector "average.cpuPercent" tries to read obj.average.cpuPercent → null
SELECT average(cpuPercent) FROM SystemSample TIMESERIES FACET entityName

-- good: selector "cpu" reads the flat field
SELECT average(cpuPercent) AS cpu FROM SystemSample TIMESERIES FACET entityName
```

## Critical gotcha — mount points vary by host

`StorageSample WHERE mountPoint = '/'` excludes Unraid entirely. Unraid (Slackware) has no `/` mount — only `/var/lib/docker`, `/mnt/cache`, `/mnt/disk1`, `/mnt/user`, `/etc/libvirt`. macOS reports both `/` and `/System/Volumes/Data` (same APFS volume, identical numbers).

Use a curated mount allowlist + facet by both:
```sql
SELECT average(diskUsedPercent) AS disk
FROM StorageSample
WHERE mountPoint IN ('/', '/mnt/cache', '/mnt/disk1', '/mnt/user', '/var/lib/docker')
SINCE 1 hour ago TIMESERIES 1 minute
FACET entityName, mountPoint LIMIT 50
```

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

## Dashboards

| UID | Title | Default range | Refresh | Datasource(s) |
|---|---|---|---|---|
| `a500e9d5-d384-453d-8128-a8b55414dd89` | Home Infrastructure (NR) | now-1h | 30s | NerdGraph (Infinity) |
| `cloudflare` | Cloudflare | now-3h | 1m | Timescale (Postgres) |
| `home-assistant` | Home Assistant | now-24h | 1m | Timescale (Postgres) |
| `unifi` | UniFi | now-3h | 1m | Timescale (Postgres) |
| `media` | Media stack | now-24h | 1m | Timescale (Postgres) |
| `unraid` | Unraid | now-24h | 1m | Timescale (Postgres) |

## Postgres datasource — Grafana 11+ gotcha

Grafana 11.x changed how the Postgres datasource reads its database name. The legacy `database` at the top level of the datasource JSON is no longer enough — it must **also** appear inside `jsonData`:

```json
{
  "name": "Timescale (metrics)",
  "type": "grafana-postgresql-datasource",
  "url": "192.168.1.200:5432",
  "user": "metrics",
  "database": "metrics",
  "jsonData": {
    "sslmode": "disable",
    "postgresVersion": 1600,
    "timescaledb": true,
    "database": "metrics"
  },
  "secureJsonData": { "password": "<from SOPS>" }
}
```

Symptom when the second `database` is missing: panel shows red triangle + "You do not currently have a default database configured for this data source. Postgres requires a default database with which to connect."

## Auth / session — token rotation on mobile

If panels show "No data" with a warning triangle and Grafana logs show `[session.token.rotate] token needs to be rotated`, the browser session expired but the cached page is still showing. The fix is to **sign out + back in** OR run Grafana with longer session lifetimes:

```bash
-e GF_AUTH_LOGIN_MAXIMUM_INACTIVE_LIFETIME_DURATION=30d \
-e GF_AUTH_LOGIN_MAXIMUM_LIFETIME_DURATION=90d \
-e GF_AUTH_TOKEN_ROTATION_INTERVAL_MINUTES=1440 \
-e GF_SECURITY_COOKIE_SAMESITE=lax
```

## Stat panel time-picker independence

A stat panel with `format: time_series` is filtered by Grafana to rows inside the dashboard time range — so "Total today" using `WHERE time >= date_trunc('day', NOW())` returns 0 if the picker is set to "Last 5 min" before midnight. Use `format: table` and an explicit window:

```json
{ "format": "table", "rawQuery": true,
  "rawSql": "SELECT SUM(requests) AS value FROM cf_zone_stats WHERE time >= date_trunc('day', NOW())" }
```

## Postgres / Timescale SQL patterns

Comprehensive set lives in `skills/timescale/SKILL.md`. Key ones:

- **Octopus monotonic counters** — `MAX(value) - MIN(value)` per bucket; never `last - first` (out-of-order inserts will flip sign).
- **Cache hit ratio** — `100.0 * SUM(cached_requests)::float / NULLIF(SUM(requests), 0)`.
- **Status-class stacked area** — `CONCAT((status_code/100)::text, 'xx') AS metric, SUM(requests) AS value` grouped by minute + class.
- **Heatmap (hour-of-day)** — `time_bucket('1 hour', time)` plus heatmap panel `calculate: true`.
- **Top-N delta vs yesterday** — `LAG(value)` window or two-window subquery + percent-change column.

## Related Skills
- `skills/newrelic/SKILL.md` — NR account, license keys, agent installs
- `skills/cloudflare/SKILL.md` — tunnel/DNS/Access API patterns
- `skills/cloudflare-tunnel/SKILL.md` — the tunnel-specific subset
- `skills/docker-management/SKILL.md` — Unraid container ops
