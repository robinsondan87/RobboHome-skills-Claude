---
description: ha-tools — direct HA REST + WebSocket access for OpenClaw agents (primary user: EVE Home). Read state, call services, announce on Echos, list entities by area/label/domain.
---

# Skill: ha-tools

Direct HA access for OpenClaw agents. Designed for EVE Home (the home-assistant specialist agent). Uses HA's REST + WebSocket API with a long-lived access token scoped to the agent's HA user.

## Auth

- HA URL: `http://192.168.1.151:8123`
- Token: `$HA_EVE_HOME_TOKEN` (loaded from SOPS via `source ~/data/config/load-secrets.sh`)
- Header for all calls: `Authorization: Bearer $HA_EVE_HOME_TOKEN`

## Tool 1: `ha_get_state(entity_id)`

```bash
curl -s -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  "http://192.168.1.151:8123/api/states/${entity_id}" \
  | jq '.state'
```

Returns the current state string (e.g. `"on"`, `"off"`, a number, a timestamp).

## Tool 2: `ha_get_attributes(entity_id)`

```bash
curl -s -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  "http://192.168.1.151:8123/api/states/${entity_id}" \
  | jq '.attributes'
```

Returns the full attribute dict (friendly_name, unit_of_measurement, etc.).

## Tool 3: `ha_call_service(domain, service, entity_id, [data])`

```bash
curl -s -X POST -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  -H "Content-Type: application/json" \
  "http://192.168.1.151:8123/api/services/${domain}/${service}" \
  -d '{"entity_id": "'"${entity_id}"'"}'
```

For services with extra data (e.g. `light.turn_on` with `brightness_pct`):

```bash
-d '{"entity_id": "light.lounge_group", "brightness_pct": 60}'
```

## Tool 4: `ha_announce(echo, message)`

The `alexa_devices` integration registers each Echo's announce surface as a `notify.*` **entity** (not a legacy notify-domain service). Use `notify.send_message` with the entity ID:

```bash
curl -s -X POST -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  -H "Content-Type: application/json" \
  "http://192.168.1.151:8123/api/services/notify/send_message" \
  -d '{"entity_id": "notify.'"${echo}"'_announce", "message": "'"${message}"'"}'
```

Expected: HTTP 200 with the entity row in the response body. Echo speaks within ~5s.

Do **not** use the legacy `POST /api/services/notify/<echo>_announce` shape — that returns HTTP 400 in this HA build (no service of that name exists).

Available `echo` values: `kitchen_echo_show`, `williams_echo_dot`, `dan_s_2nd_echo_dot`, `dan_s_3rd_echo_dot`, `dan_s_4th_echo_dot`.

## Tool 5: `ha_list_entities(area=None, label=None, domain=None)`

Discovery via `/api/states` filtered client-side:

```bash
curl -s -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  "http://192.168.1.151:8123/api/states" \
  | jq --arg dom "$domain" '[.[] | select(.entity_id | startswith($dom + "."))]'
```

For richer area/label filtering, query the entity registry via WebSocket (`config/entity_registry/list`) and filter on `area_id`/`labels`.

## Permissions / scope

EVE Home's HA user (`eve_home`) is non-admin with curated entity exposure. It can:
- Read state of all exposed entities (~50 — see spec §4.3)
- Call `light.*`, `scene.turn_on`, `script.*`, `climate.*` for exposed entities
- Call `notify.<echo>_announce` for the 5 Echos

It cannot:
- Restart HA
- Edit YAML / config
- Access cameras, locks, admin entities
- Access entities not in the curated exposure list

## Error patterns

- `401` → token expired/invalid; rotate via HA Profile screen
- `404` → entity_id wrong or not exposed to this user
- `500` → service call failed inside HA; check HA logs
