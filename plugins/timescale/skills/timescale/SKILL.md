---
name: timescale
description: TimescaleDB on svr005:5432 — the home metrics store fed by per-minute systemd collectors and queried by Grafana.
---

# Skill: Timescale (home metrics)

A single Postgres + TimescaleDB extension instance on `svr005`, used as the
source of truth for **anything we want time-series-shaped without paying NR
ingest**. Grafana runs beside it and reads this database over the compose
network while LAN clients can use port 5432.

## Connection

| | |
|---|---|
| Host | `svr005` (`192.168.1.92`) |
| Port | `5432` |
| Container | `timescaledb` (image `timescale/timescaledb:2.26.3-pg16`) |
| Database | `metrics` (user: `metrics`) |
| Data dir on host | `/srv/monitoring/timescaledb` |
| Compose bundle | `/opt/monitoring/compose.yaml` |
| Creds | SOPS keys `TIMESCALE_HOST` / `TIMESCALE_PORT` / `TIMESCALE_USER` / `TIMESCALE_PASS` / `TIMESCALE_DB` |

## Deploy
```bash
ssh svr005-lan 'cd /opt/monitoring && ./start-stack.sh up -d timescaledb'
ssh svr005-lan 'docker exec timescaledb pg_isready -U metrics -d metrics'
```

Keep the image pinned to `2.26.3-pg16` until an intentional Timescale
extension upgrade is planned. Restoring the 2.26.3 catalogue into a newer
image failed before `timescaledb_post_restore()` could run.

Quick psql shell:
```bash
ssh -t svr005-lan 'docker exec -it timescaledb psql -U metrics -d metrics'
```

## Active hypertables

| Table | Granularity | Retention | Compression | Source |
|---|---|---|---|---|
| `cf_zone_stats` | 1 min | 180 days | after 7d | `cf-poll.sh` (totals per zone) |
| `cf_country_stats` | 1 min | 180 days | after 7d | `cf-poll.sh` (per-country) |
| `cf_status_stats` | 1 min | 180 days | after 7d | `cf-poll.sh` (per-status-code) |
| `cf_host_stats` | 1 min | 180 days | after 7d | `cf-poll.sh` (per-subdomain) |
| `ha_state_history` | per state-change | 180 days | after 7d | `ha-poll.sh` (HA `/api/states`) |
| `unifi_device_stats` | 1 min | 180 days | after 7d | `unifi-poll.sh` (`/stat/device`) |
| `unifi_client_stats` | 1 min | 90 days | after 3d | `unifi-poll.sh` (`/stat/sta`) |
| `unifi_site_stats` | 1 min | 180 days | after 7d | `unifi-poll.sh` (`/stat/health`) |
| `unifi_speedtest` | per test (~1/day) | 180 days | none | `unifi-poll.sh` (`archive.speedtest`) |
| `media_app_stats` | 1 min | 180 days | after 7d | `media-poll.sh` (media stack + workers) |
| `media_download_events` | event | 180 days | none | `media-poll.sh` (Arr imports + downloader completions) |
| `unraid_array` | 1 min | 180 days | after 7d | `unraid-poll.sh` (GraphQL `array.parityCheckStatus`) |
| `unraid_disk` | 1 min | 90 days | after 7d | `unraid-poll.sh` (GraphQL `array.disks` + `parities` + `caches`) |
| `unraid_notifications` | event | 180 days | none | `unraid-poll.sh` (GraphQL `notifications.list`, idempotent on `notif_id`) |

All hypertables: `chunk_time_interval => '1 day'`, `compress_segmentby` set to the most-faceted text column.

## Pollers

Production pollers live at `/opt/monitoring/collectors/` on `svr005`. The
templated `monitoring-collector@.service` and `.timer` units run each poller
once per minute with an overlap-preventing lock. Check them with:

```bash
ssh svr005-lan 'systemctl list-timers --all | grep monitoring-collector'
ssh svr005-lan 'journalctl -u monitoring-collector@media-poll.service -n 50'
```

The old Unraid poller cron entries were removed at cutover. Their rollback copy
is `/root/crontab.pre-svr005-monitoring-20260811T122045Z`. The operational
Usenet backlog feeder and media backlog controller remain on Unraid; they are
not metric collectors.

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
Polls the media services and qBittorrent on Unraid plus SABnzbd on `svr004`
every minute. Wide-table writes to
`media_app_stats(ts, app, metric, label, value_num)` keep cardinality low by
aggregating per-state rather than per-title:
- **Sonarr / Radarr** — series/movie counts, episodes-on-disk, missing, queue (warnings + errors), per-root-folder diskspace.
- **Sonarr / Radarr import events** — latest successful import history is
  deduplicated into `media_download_events`; the initial backfill captured
  1,000 Sonarr and all 333 Radarr imports, while steady-state polling checks
  the latest 100 per app.
- **Prowlarr** — per-indexer queries, grabs, failures, avg response time (label=indexer name).
- **Jellyfin** — `/Items/Counts` (movies/series/episodes/boxsets/etc), active sessions, transcoding sessions, connected clients.
- **Jellyseerr** — `/api/v1/request/count` (total/movie/tv/pending/approved/processing/available/declined).
- **Audiobookshelf** — per-library items/duration/size + open sessions (label=library name).

Each app's auth differs:
- *arr / Jellyseerr → `X-Api-Key` header
- Jellyfin → `Authorization: MediaBrowser Token=…` header
- ABS → `Authorization: Bearer …` header
- qBittorrent → `POST /api/v2/auth/login` with `username=…&password=…` form body + `Referer` header. Returns `Ok.` on success and sets a session cookie. Captures: global dl/up speed, total library size, downloaded/uploaded totals (for share ratio), torrents-by-state buckets (downloading/stalledDL/uploading/seeding/error/etc).
- SABnzbd → API key in the `apikey` query parameter. SAB labels every queued
  slot as `Downloading`, so active NZBs must come from the global queue state;
  counting slot statuses can turn one active job into hundreds.
- qBittorrent completion events are stored by torrent hash in
  `media_download_events`. Cumulative session transfer counters can be turned
  into bytes transferred over a dashboard range by summing only positive
  deltas, which survives qBittorrent container restarts.

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

1. Create a hypertable with `chunk_time_interval => '1 day'`,
   `compress_segmentby`, and `add_compression_policy(... '7 days')` plus the
   appropriate 90- or 180-day retention policy.
2. Drop a poller script into `/opt/monitoring/collectors/` on `svr005`.
3. Build the SQL via `jq -r` from the API response — **never** interpolate fields into shell heredocs (quoting hell). Pipe the SQL to `docker exec -i timescaledb psql`.
4. Enable `monitoring-collector@<script-without-.sh>.timer`.
5. Add a panel in Grafana using the `Timescale (metrics)` Postgres datasource.

## Gotchas

- **Grafana 11+ Postgres datasource** needs `database` set inside `jsonData`, not just at the top level. See `skills/grafana/SKILL.md` for the fix.
- **Compression** is enabled per-hypertable but **chunks compress only after the policy interval** (we set 7 days). Decompression of recent chunks is automatic on query.
- **Drops + recreates** of a hypertable also drop policies — re-add `add_compression_policy` / `add_retention_policy` after any `DROP TABLE` rebuild.
- **Backups** run at 02:45 Europe/London and copy to
  `/mnt/user/backups/svr005-monitoring` on Unraid. Retention enforcement runs at
  03:20. Both are systemd timers on `svr005`.

## Related Skills
- `skills/grafana/SKILL.md` — Grafana beside this database on `svr005`
- `skills/cloudflare/SKILL.md` — CF token/API basics (the `cf-poll.sh` consumer)
- `skills/home-assistant/SKILL.md` — HA URL + long-lived token (the `ha-poll.sh` consumer)
- `skills/secrets/SKILL.md` — where the `TIMESCALE_*` keys live
