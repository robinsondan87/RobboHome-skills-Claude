---
description: TimescaleDB on Unraid (svr001:5432) — the home metrics store, fed by per-minute pollers (cf-poll, ha-poll) and queried directly by Grafana via the Postgres datasource.
---

# Skill: Timescale (home metrics)

A single Postgres + TimescaleDB extension instance on Unraid, used as the source of truth for **anything we want time-series-shaped without paying NR ingest** — Cloudflare zone analytics, Home Assistant state history, and future custom feeds. Same Grafana dashboards read from this and from NR side-by-side.

## Connection

| | |
|---|---|
| Host | `svr001` (Unraid) |
| Port | `5432` |
| Container | `timescaledb` (image `timescale/timescaledb:latest-pg16`) |
| Database | `metrics` (user: `metrics`) |
| Data dir on host | `/mnt/user/appdata/timescaledb/data` |
| Creds | SOPS keys `TIMESCALE_HOST` / `TIMESCALE_PORT` / `TIMESCALE_USER` / `TIMESCALE_PASS` / `TIMESCALE_DB` |

Quick psql shell:
```bash
source ~/data/config/load-secrets.sh
ssh svr001 "docker exec -it -e PGPASSWORD='$TIMESCALE_PASS' timescaledb psql -U $TIMESCALE_USER -d $TIMESCALE_DB"
```

## Active hypertables

| Table | Granularity | Retention | Compression | Source |
|---|---|---|---|---|
| `cf_zone_stats` | 1 min | 1 year | after 7d | `cf-poll.sh` (totals per zone) |
| `cf_country_stats` | 1 min | 1 year | after 7d | `cf-poll.sh` (per-country) |
| `cf_status_stats` | 1 min | 1 year | after 7d | `cf-poll.sh` (per-status-code) |
| `cf_host_stats` | 1 min | 1 year | after 7d | `cf-poll.sh` (per-subdomain) |
| `ha_state_history` | per state-change | 1 year | after 7d | `ha-poll.sh` (HA `/api/states`) |
| `unifi_device_stats` | 1 min | 1 year | after 7d | `unifi-poll.sh` (`/stat/device`) |
| `unifi_client_stats` | 1 min | 90 days | after 3d | `unifi-poll.sh` (`/stat/sta`) |
| `unifi_site_stats` | 1 min | 1 year | after 7d | `unifi-poll.sh` (`/stat/health`) |
| `unifi_speedtest` | per test (~1/day) | 2 years | none | `unifi-poll.sh` (`archive.speedtest`) |
| `media_app_stats` | 1 min | 1 year | after 7d | `media-poll.sh` (Sonarr/Radarr/Prowlarr/Jellyfin/Jellyseerr/ABS) |
| `unraid_array` | 1 min | 2 years | after 7d | `unraid-poll.sh` (GraphQL `array.parityCheckStatus`) |
| `unraid_disk` | 1 min | 90 days | after 7d | `unraid-poll.sh` (GraphQL `array.disks` + `parities` + `caches`) |
| `unraid_notifications` | event | none | none | `unraid-poll.sh` (GraphQL `notifications.list`, idempotent on `notif_id`) |

All hypertables: `chunk_time_interval => '1 day'`, `compress_segmentby` set to the most-faceted text column.

## Pollers

Both live on Unraid at `/mnt/user/appdata/scripts/`. Each is invoked from root's crontab — see `/boot/config/crontab.robbohome` (persisted across reboots via `/boot/config/go`):

```cron
# CF analytics — per-minute via httpRequestsAdaptiveGroups
* * * * * CLOUDFLARE_API_TOKEN='…' TIMESCALE_PASS='…' /mnt/user/appdata/scripts/cf-poll.sh >> /var/log/cf-poll.log 2>&1
# HA states — per-minute snapshot of /api/states
* * * * * HOMEASSISTANT_URL='…' HOMEASSISTANT_TOKEN='…' TIMESCALE_PASS='…' /mnt/user/appdata/scripts/ha-poll.sh >> /var/log/ha-poll.log 2>&1
```

Unraid uses **vixie crond**, which **doesn't read `/etc/cron.d/`** — must be in `/var/spool/cron/crontabs/root`. The `/boot/config/go` snippet rebuilds it on boot.

### `cf-poll.sh`
Hits Cloudflare's GraphQL Analytics endpoint with **four sub-queries in one POST**:
- totals (counts/bytes/visits per minute)
- cached (filtered to `cacheStatus_in: ["hit","stream_hit","updating","stale"]`)
- threats (filtered to `securityAction_in: ["block","challenge","jschallenge"]`)
- countries / statuses / hosts (faceted by `clientCountryName` / `edgeResponseStatus` / `clientRequestHTTPHost`)

Uses `httpRequestsAdaptiveGroups` (works on **CF Free plan**, 1-minute granularity). The Pro-only `httpRequests1mGroups` would simplify the JSON but we don't have Pro. `httpRequests1hGroups` (free, hourly) is no longer used.

### `unraid-poll.sh`
Polls the Unraid 7.2 GraphQL endpoint at `http://192.168.1.200/graphql` (auth: `x-api-key: $UNRAID_API_KEY`). One single GraphQL doc fetches array+parityCheck, all disks (data+parity+cache via separate top-level fields, joined client-side), and the notifications list.

**Field naming on Unraid 7.2** (Network 10.x): the parity status is at `array.parityCheckStatus` (not `array.parityCheck`). Disks are at `array.disks` (data), `array.parities` (parity), `array.caches` (cache pool) — all separate arrays you have to concat. Each disk's temp comes from the controller's cached value, so polling does **not** wake spun-down disks (Unraid returns `temp: null` when `isSpinning: false`).

`notifications.list(filter: {limit: N, offset: 0, type: UNREAD})` requires a non-null filter argument — easy to miss. Idempotent insert on `(host, notif_id)` so the same notification doesn't duplicate across polls.

### `media-poll.sh`
Polls 6 *arr-stack apps + Jellyfin every minute, all on `192.168.1.200`. Wide-table writes to `media_app_stats(ts, app, metric, label, value_num)` — keeps cardinality low by aggregating per-state rather than per-title:
- **Sonarr / Radarr** — series/movie counts, episodes-on-disk, missing, queue (warnings + errors), per-root-folder diskspace.
- **Prowlarr** — per-indexer queries, grabs, failures, avg response time (label=indexer name).
- **Jellyfin** — `/Items/Counts` (movies/series/episodes/boxsets/etc), active sessions, transcoding sessions, connected clients.
- **Jellyseerr** — `/api/v1/request/count` (total/movie/tv/pending/approved/processing/available/declined).
- **Audiobookshelf** — per-library items/duration/size + open sessions (label=library name).

Each app's auth differs:
- *arr / Jellyseerr → `X-Api-Key` header
- Jellyfin → `Authorization: MediaBrowser Token=…` header
- ABS → `Authorization: Bearer …` header
- qBittorrent → `POST /api/v2/auth/login` with `username=…&password=…` form body + `Referer` header. Returns `Ok.` on success and sets a session cookie. Captures: global dl/up speed, total library size, downloaded/uploaded totals (for share ratio), torrents-by-state buckets (downloading/stalledDL/uploading/seeding/error/etc).

**qBittorrent password reset** if locked out: stop container, edit `/mnt/user/appdata/qbittorrent/qBittorrent/qBittorrent.conf`, replace `WebUI\Password_PBKDF2=...` with the well-known `adminadmin` hash (`@ByteArray(ARQ77eY1NUZaQsuDHbIMCA==:0WMRkYTUWVT9wVvdDtHAjU9b3b7uB8NR1Gur2hmQCvCDpm39Q+PsJRJPaCU51dEiz+dTzh8qbPsL8WkFljQYFQ==)`), start container. Login with existing username + password `adminadmin`.

Wrap each app block in `if [ -n "$URL" ] && [ -n "$KEY" ]` so partial creds don't crash the whole poll.

### `unifi-poll.sh`
Authenticates to the UniFi controller (UCG Ultra at `192.168.1.1`) using **legacy cookie + CSRF** (creds: `UNIFI_NETWORK_USERNAME` / `UNIFI_NETWORK_PASSWORD` from SOPS). The Integration API key path doesn't expose enough — cookie path gives you `stat/sta`, `stat/device`, `stat/health`, and `stat/report/archive.speedtest`. Captures:
- Per-device snapshot (CPU, memory, uptime, RX/TX bytes, num connected stations)
- Per-client snapshot (hostname, AP/SSID, RSSI, link rate, RX/TX bytes)
- Site-level health (subsystem statuses, WAN throughput, latency)
- Speedtest history (idempotent on `_id`, last 7 days each poll — back-fills automatically)

**DPI does not work on UCG Ultra** (and other consumer UDM/UDR/UCG-Lite). The endpoints respond `rc: ok` but with empty data — no per-app DPI ASIC. Don't waste time on `/stat/dpi`. Get DPI-equivalent data from PiHole/AdGuard logs instead.

### `ha-poll.sh`
Pulls `/api/states`, filters to interesting domains (sensor / binary_sensor / switch / light / climate / number / input_* / media_player / device_tracker / person / lock / cover / fan / weather / sun / zone / counter / select), inserts each row with `state_text`, `state_numeric` (parsed if state is numeric), and `unit`. `ON CONFLICT (time, entity_id) DO NOTHING` so re-polls don't duplicate.

## Critical SQL patterns

### Octopus monotonic counters — daily kWh
The Octopus integration's `current_total_consumption` is a **monotonically increasing meter reading** (resets only on meter swaps). For daily kWh use `MAX-MIN` per bucket — handles meter resets and out-of-order inserts:
```sql
SELECT time_bucket('1 day', time) AS day,
       MAX(state_numeric) - MIN(state_numeric) AS kwh
FROM ha_state_history
WHERE entity_id = 'sensor.octopus_energy_electricity_*_current_total_consumption'
  AND state_numeric IS NOT NULL
  AND $__timeFilter(time)
GROUP BY 1 ORDER BY 1;
```
**Don't** use `last - first` — out-of-order inserts can flip the sign.

### Stat panels that survive any time picker
Stat panels with `format: "time_series"` get filtered to the dashboard time range. For "today's total" and similar, use `format: "table"` and an absolute window:
```sql
SELECT SUM(requests) AS value
FROM cf_zone_stats
WHERE time >= date_trunc('day', NOW());
```

### Cache hit ratio per zone (5-min buckets)
```sql
SELECT time_bucket('5 minutes', time) AS time,
       zone_name AS metric,
       100.0 * SUM(cached_requests)::float / NULLIF(SUM(requests),0) AS value
FROM cf_zone_stats
WHERE $__timeFilter(time)
GROUP BY 1, 2 ORDER BY 1;
```

### Status-class stacked area
```sql
SELECT time_bucket('1 minute', time) AS time,
       CONCAT((status_code/100)::text, 'xx') AS metric,
       SUM(requests) AS value
FROM cf_status_stats
WHERE $__timeFilter(time)
GROUP BY 1, 2 ORDER BY 1;
```

### Vampire / standby load (5th-pct demand, last 24h)
```sql
SELECT NOW() AS time,
       percentile_cont(0.05) WITHIN GROUP (ORDER BY state_numeric) AS vampire
FROM ha_state_history
WHERE entity_id = 'sensor.octopus_energy_electricity_*_current_demand'
  AND state_numeric IS NOT NULL
  AND time > NOW() - INTERVAL '1 day';
```

## Adding a new poller (template)

1. Create a hypertable with `chunk_time_interval => '1 day'`, `compress_segmentby`, and `add_compression_policy(... '7 days')` + `add_retention_policy(... '1 year')`.
2. Drop a poller script into `/mnt/user/appdata/scripts/`.
3. Build the SQL via `jq -r` from the API response — **never** interpolate fields into shell heredocs (quoting hell). Pipe the SQL to `docker exec -i timescaledb psql`.
4. Append the cron line to `/boot/config/crontab.robbohome` and re-install via `( crontab -l | grep -v <script> ; cat /boot/config/crontab.robbohome ) | crontab -`.
5. Add a panel in Grafana using the `Timescale (metrics)` Postgres datasource.

## Gotchas

- **Grafana 11+ Postgres datasource** needs `database` set inside `jsonData`, not just at the top level. See `skills/grafana/SKILL.md` for the fix.
- **Unraid vixie crond** ignores `/etc/cron.d/` — must use root's crontab.
- **Compression** is enabled per-hypertable but **chunks compress only after the policy interval** (we set 7 days). Decompression of recent chunks is automatic on query.
- **Drops + recreates** of a hypertable also drop policies — re-add `add_compression_policy` / `add_retention_policy` after any `DROP TABLE` rebuild.

## Related Skills
- `skills/grafana/SKILL.md` — Grafana on Unraid + dashboards reading from this DB
- `skills/cloudflare/SKILL.md` — CF token/API basics (the `cf-poll.sh` consumer)
- `skills/home-assistant/SKILL.md` — HA URL + long-lived token (the `ha-poll.sh` consumer)
- `skills/secrets/SKILL.md` — where the `TIMESCALE_*` keys live
