# HA Track 2a — MCP Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire up agent ↔ HA integration: install/configure the ha-mcp add-on for Claude Code, create per-agent HA users with scoped permissions, build a small OpenClaw `ha-tools` skill so EVE Home can do family Q&A and Echo broadcasts.

**Architecture:** Two transports, one HA backend. Claude Code uses the existing `homeassistant-ai/ha-mcp` HAOS add-on (port 9583, secret-path-in-URL auth). EVE Home (OpenClaw home-assistant agent) uses HA's REST + WebSocket API directly with a long-lived access token. Per-agent HA users provide identity + audit; entity-exposure curation enforces EVE Home's read+control+announce scope.

**Tech Stack:** HA OS (2026.4.4), HA REST + WebSocket API, `homeassistant-ai/ha-mcp` HAOS add-on, Python `websocket-client` + `urllib`, SOPS-encrypted secrets, OpenClaw skill format (markdown SKILL.md).

**Spec reference:** `~/Projects/RobboHome-skills-Claude/docs/superpowers/specs/2026-05-03-ha-track2a-mcp-foundation-design.md`

**Access:**
- HA OS: `ssh dan@192.168.1.151` (passwordless sudo); HA REST/WS via `$HOMEASSISTANT_TOKEN` from SOPS
- SOPS: `~/data/config/.secrets.env`, edit via `sops -e -i` after decrypt+modify+overwrite
- Mac mini: where `~/.claude.json` lives + where Claude Code runs

---

## Phase 0 — Prerequisites (one-time UI step)

### Task 0: Take the ha-mcp add-on update + confirm config

**Files:** none (HA UI work on phone or browser)

- [ ] **Step 1: Take the 7.4.0 → 7.4.1 update**

In HA UI: **Settings → Add-ons → Home Assistant MCP Server → Update**. Wait for "Up-to-date" banner.

- [ ] **Step 2: Confirm Configuration tab settings**

Same add-on → **Configuration tab**. Verify:

- `Backup hint`: `normal`
- `Secret path override`: `/private_eDpQpJz3krP08FFNJVVXZA` (or note whatever it currently is — used in Task 7 below)
- `Enable skills`: ON
- `Enable skills as tools`: ON
- `Enable tool search`: OFF (we're using Claude with full context, not a small-context model)

If `Secret path override` is different to what's in the spec, copy it down — it's needed in Task 7.

- [ ] **Step 3: Restart the add-on**

Add-on Info tab → **Restart**. Wait ~10 sec.

- [ ] **Step 4: Verify endpoint reachable**

From Mac mini terminal:

```bash
SECRET="/private_eDpQpJz3krP08FFNJVVXZA"  # update if different
curl -s -o /dev/null -w '%{http_code}\n' --max-time 5 \
  "http://192.168.1.151:9583$SECRET"
```

Expected output: `405` (means: path correct, method GET not supported by MCP — that's fine, MCP uses POST).

---

## Phase 1 — HA users + tokens

### Task 1: Create `claude_code` HA user

**Files:** none (HA UI work)

- [ ] **Step 1: Create the user**

In HA UI: **Settings → People → Users → Add user**. Fill in:

- Display name: `Claude Code`
- Username: `claude_code`
- Password: generate strong (use `openssl rand -base64 24`); save it — needed in Task 4
- **Administrator: ON**

- [ ] **Step 2: Verify**

Log out and log in as `claude_code` in a private browser window. Confirm Settings sidebar item is visible (admin = full access). Log back out.

### Task 2: Create `eve_home` HA user

**Files:** none (HA UI work)

- [ ] **Step 1: Create the user**

Same UI flow as Task 1. Fill in:

- Display name: `EVE Home`
- Username: `eve_home`
- Password: generate strong; save it — needed in Task 4
- **Administrator: OFF**

- [ ] **Step 2: Verify**

Log out and log in as `eve_home` in a private browser window. Confirm Settings sidebar item is **hidden** (non-admin). Stay logged in for Task 3.

### Task 3: Generate LLATs for both users

**Files:** none (HA UI work, per user)

- [ ] **Step 1: Generate LLAT for `eve_home`**

While logged in as `eve_home`: **Profile (bottom-left) → Long-lived access tokens → Create token**. Name: `EVE Home OpenClaw`. Copy the token — won't be shown again. Save for Task 4.

- [ ] **Step 2: Generate LLAT for `claude_code`**

Log out → log in as `claude_code` → same flow. Token name: `Claude Code MCP fallback`. Copy and save.

(This LLAT is a fallback — the ha-mcp add-on uses its own URL-secret auth and doesn't need this token. We capture it in case audit attribution requires falling back to direct REST per spec §4.2.)

### Task 4: Save passwords + LLATs to SOPS

**Files:**
- Modify: `~/data/config/.secrets.env` (SOPS-encrypted)

- [ ] **Step 1: Decrypt, append, re-encrypt**

On the Mac mini terminal:

```bash
cd ~/data/config && \
  SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -d .secrets.env > /tmp/secrets.plain && \
  cat >> /tmp/secrets.plain <<EOF
HA_CLAUDE_CODE_PASSWORD=<paste from Task 1>
HA_CLAUDE_CODE_TOKEN=<paste from Task 3 step 2>
HA_EVE_HOME_PASSWORD=<paste from Task 2>
HA_EVE_HOME_TOKEN=<paste from Task 3 step 1>
EOF
cp /tmp/secrets.plain .secrets.env && \
  SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt sops -e -i .secrets.env && \
  rm /tmp/secrets.plain
```

- [ ] **Step 2: Verify**

```bash
source ~/data/config/load-secrets.sh
for k in HA_CLAUDE_CODE_PASSWORD HA_CLAUDE_CODE_TOKEN HA_EVE_HOME_PASSWORD HA_EVE_HOME_TOKEN; do
  if [ -n "${!k}" ]; then echo "$k: ✓ set"; else echo "$k: ✗ missing"; fi
done
```

Expected: all four lines show `✓ set`.

---

## Phase 2 — EVE Home entity exposure curation

### Task 5: Curate `eve_home` exposure list

**Files:** none (uses HA WebSocket API)

- [ ] **Step 1: Find `eve_home`'s user_id**

```bash
ssh dan@192.168.1.151 'sudo cat /config/.storage/auth' | python3 -c "
import json, sys
d = json.load(sys.stdin)
for u in d['data']['users']:
    if u.get('name','').lower() == 'eve home' or u.get('username','').lower() == 'eve_home':
        print(f\"user_id: {u['id']}\")
        break"
```

Save the user_id (e.g. `01ABC…`).

- [ ] **Step 2: Build the exposure list + apply via WS API**

(The exposure mechanism uses the `homeassistant/expose_entity` WS command, scoped via `assistants` field. EVE Home isn't a built-in assistant key — see Step 4 fallback if this approach doesn't apply per-user. For now we curate the **`conversation`** assistant which `eve_home` can be configured to use.)

```bash
source ~/data/config/load-secrets.sh && python3 - <<'EOF'
import json, websocket, os
ws = websocket.create_connection('ws://192.168.1.151:8123/api/websocket'); ws.recv()
ws.send(json.dumps({'type': 'auth', 'access_token': os.environ['HOMEASSISTANT_TOKEN']})); ws.recv()

include = []
# 8 light groups
for r in ['lounge','kitchen','library','dining','hall','landing','office','office_outside']:
    include.append(f'light.{r}_group')
# Echos (announce notify) — names from HA-15 work
for echo in ['kitchen_echo_show','williams_echo_dot','dan_s_2nd_echo_dot','dan_s_3rd_echo_dot','dan_s_4th_echo_dot']:
    include.append(f'notify.{echo}_announce')
# Scenes
for s in ['lounge_movie','kitchen_morning_warm','lounge_morning_warm']:
    include.append(f'scene.{s}')
for r in ['lounge','kitchen','library','dining','hall','landing','office','william','guest']:
    include.append(f'scene.room_{r}_default')
# Macros
include.extend(['script.goodnight','script.morning','script.movie'])
# People
include.extend(['person.dan','person.nicola','person.william'])
# Climate (control)
include.extend(['climate.fire','climate.office_aircon'])
# Cat feeders read-only-useful sensors
for cat in ['khaos','kiki','lunar']:
    include.extend([
        f'sensor.{cat}_smart_feeder_last_feed_time',
        f'sensor.{cat}_smart_feeder_today_s_feeding_quantity_weight',
        f'binary_sensor.{cat}_smart_feeder_food_status',
        f'binary_sensor.{cat}_smart_feeder_wi_fi',
    ])
# Presence + door + temp + key sensors
include.extend([
    'binary_sensor.fp2_office_presence',
    'sensor.fp2_office_illuminance',
    'binary_sensor.kitchen_echo_show_motion',
    'binary_sensor.williams_echo_dot_motion',
    'binary_sensor.doorbell_camera_motion',
    'sensor.blood_sugar',
    'weather.forecast_home',
    'calendar.stafford_borough_council',
])

ws.send(json.dumps({
    'id': 1,
    'type': 'homeassistant/expose_entity',
    'assistants': ['conversation'],
    'entity_ids': include,
    'should_expose': True
}))
print('expose:', json.loads(ws.recv()).get('success'), '— count:', len(include))
ws.close()
EOF
```

- [ ] **Step 3: Verify count**

```bash
source ~/data/config/load-secrets.sh && python3 - <<'EOF'
import json, websocket, os, subprocess
ws = websocket.create_connection('ws://192.168.1.151:8123/api/websocket'); ws.recv()
ws.send(json.dumps({'type': 'auth', 'access_token': os.environ['HOMEASSISTANT_TOKEN']})); ws.recv()
ws.send(json.dumps({'id': 1, 'type': 'config/entity_registry/list'}))
ents = json.loads(ws.recv())['result']
exposed = [e for e in ents if e.get('options', {}).get('conversation', {}).get('should_expose') is True]
print(f'Exposed to conversation assistant: {len(exposed)}')
ws.close()
EOF
```

Expected: number matches the `include` list count from Step 2 (~50 entities).

- [ ] **Step 4: Document fallback if exposure-by-user is needed**

If EVE Home turns out NOT to honour the `conversation` assistant exposure (i.e. it makes raw REST calls and gets ALL entities the LLAT user can see), the alternative is to scope at the **HA user level** instead — `eve_home` is non-admin, so it can't access entities a user can't access. The exposure list is then aspirational / advisory rather than enforced. Spec §5 covers this nuance.

---

## Phase 3 — Claude Code MCP wiring

### Task 6: Register `home-assistant` MCP in `~/.claude.json`

**Files:**
- Modify: `/Users/robbohomebot/.claude.json`

- [ ] **Step 1: Read current `mcpServers` block**

```bash
python3 -c "
import json
c = json.load(open('/Users/robbohomebot/.claude.json'))
print('current mcpServers:', list((c.get('mcpServers') or {}).keys()))
"
```

Expected: empty list (per the discovery in §1 of the spec).

- [ ] **Step 2: Add the `home-assistant` MCP entry**

```bash
python3 - <<'EOF'
import json
path = '/Users/robbohomebot/.claude.json'
c = json.load(open(path))
c.setdefault('mcpServers', {})
c['mcpServers']['home-assistant'] = {
    'transport': 'http',
    'url': 'http://192.168.1.151:9583/private_eDpQpJz3krP08FFNJVVXZA'
}
json.dump(c, open(path, 'w'), indent=2)
print('added: home-assistant MCP')
print('mcpServers now:', list(c['mcpServers'].keys()))
EOF
```

If your secret path is different (per Task 0 Step 2), substitute it into the URL.

- [ ] **Step 3: Restart Claude Code**

You need to fully exit Claude Code and re-launch it for the new MCP to load. From the prompt, type `/exit`, then re-launch with `claude`.

- [ ] **Step 4: Verify MCP loaded**

In the new Claude Code session:

```bash
claude mcp list
```

Expected: `home-assistant: http://192.168.1.151:9583/… (Connected)`. If `Disconnected`, re-check the URL/secret-path matches the add-on's configured value.

### Task 7: Audit-attribution check (per spec §4.2 caveat)

**Files:** none (HA Logbook check)

- [ ] **Step 1: From a new Claude Code session, fire a no-op service via the MCP tool**

In Claude (the new session with MCP loaded), type a request like:

> "Use the home-assistant MCP to call light.toggle on light.bulb_landing once"

Claude should invoke the MCP tool and Light Landing should toggle.

- [ ] **Step 2: Check HA Logbook for attribution**

In HA UI: **Logbook → filter by entity `light.bulb_landing` → look at the most recent toggle**. The "by" field should show either `claude_code` (if the add-on does per-user impersonation) or a system identity like `Supervisor` / `Home Assistant MCP Server` (if not).

- [ ] **Step 3: Record the finding in the spec**

If the action is attributed to `claude_code` → leave spec §4.2 caveat as-is, marked resolved.

If attributed to a system identity → update spec §4.2 with the actual identity name, and note that audit attribution is at the *add-on* level not the *HA-user* level for Claude Code. (This doesn't break anything — it's a transparency note.)

```bash
# Edit spec inline based on what was observed
ssh dan@192.168.1.151 'sudo cd /config && sudo git log --oneline -3'
# (and update the spec doc with findings)
```

---

## Phase 4 — OpenClaw `ha-tools` skill + EVE Home wiring

### Task 8: Create `ha-tools` OpenClaw skill

**Files:**
- Create: `~/Projects/RobboHome-skills-Claude/plugins/ha-tools/skills/ha-tools/SKILL.md`

- [ ] **Step 1: Make the directory + write the skill file**

```bash
mkdir -p ~/Projects/RobboHome-skills-Claude/plugins/ha-tools/skills/ha-tools
cat > ~/Projects/RobboHome-skills-Claude/plugins/ha-tools/skills/ha-tools/SKILL.md <<'EOF'
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

Wrapper for `notify.<echo>_announce`:

```bash
curl -s -X POST -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  -H "Content-Type: application/json" \
  "http://192.168.1.151:8123/api/services/notify/${echo}_announce" \
  -d '{"message": "'"${message}"'"}'
```

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
EOF
echo "wrote SKILL.md"
ls -la ~/Projects/RobboHome-skills-Claude/plugins/ha-tools/skills/ha-tools/SKILL.md
```

- [ ] **Step 2: Commit**

```bash
cd ~/Projects/RobboHome-skills-Claude && \
  git add plugins/ha-tools/skills/ha-tools/SKILL.md && \
  git commit -m "feat(ha-tools): new OpenClaw skill — direct HA REST/WS access for EVE Home"
```

### Task 9: Update EVE Home agent's `TOOLS.md`

**Files:**
- Modify: `~/.openclaw/workspace/agents/home-assistant/TOOLS.md`

- [ ] **Step 1: Replace template with real inventory**

```bash
cat > ~/.openclaw/workspace/agents/home-assistant/TOOLS.md <<'EOF'
# TOOLS.md — EVE Home (Robinson home-assistant agent)

Skill: see `ha-tools/SKILL.md` for the 5 tool patterns (`ha_get_state`, `ha_get_attributes`, `ha_call_service`, `ha_announce`, `ha_list_entities`).

## Floors

- **Ground floor** — Living Room, Kitchen, Library, Dining, Hall, Downstairs Toilet, Office, Misc/System, Utility
- **Upstairs** — Master Bedroom, Landing, Williams Bedroom, Guest Room, Dressing Room
- **Outside** — Driveway, Back Garden, Office Outside (separate building)
- **Loft** — Loft

## People + device trackers

| Person | HA entity | Tracker |
|---|---|---|
| Dan | `person.dan` | `device_tracker.dans_iphone` |
| Nicola | `person.nicola` | `device_tracker.nics_iphone_16` |
| William (10) | `person.william` | `device_tracker.williams_phone` |

## Echos (per Echo identification — see HA-15 for room mapping work-in-progress)

| HA prefix | Currently known location |
|---|---|
| `kitchen_echo_show` | Kitchen (confirmed) |
| `williams_echo_dot` | Williams Bedroom (likely) |
| `dan_s_2nd_echo_dot` | TBD — pending HA-15 walk-around |
| `dan_s_3rd_echo_dot` | TBD |
| `dan_s_4th_echo_dot` | TBD |

To announce: `ha_announce('kitchen_echo_show', 'Hello')`. To check motion: `ha_get_state('binary_sensor.kitchen_echo_show_motion')`.

## Light groups (8) — preferred control surface

| Group | Members |
|---|---|
| `light.lounge_group` | living_room TV lights, big lamp, small lamp |
| `light.kitchen_group` | overhead, under-counter, cupboard, LED's |
| `light.library_group` | 5× library spots |
| `light.dining_group` | dining-room single light |
| `light.hall_group` | bulb_hallway |
| `light.landing_group` | bulb_landing |
| `light.office_group` | 6× office ceiling (front/middle/back ×2) |
| `light.office_outside_group` | 4× office_door + 4× office_outside (exterior of office building) |

## Macros (3 family-wide scripts)

| Script | Effect |
|---|---|
| `script.goodnight` | All living-area lights off, fire off, doorbell quiet, Sonos pause, notify |
| `script.morning` | Kitchen + lounge morning scenes, BBC R1 on Kitchen Echo via Alexa Routine, bin-day check, doorbell un-quiet, notify |
| `script.movie` | Lounge movie scene, Sonos lounge pause, doorbell silent for 2h, notify |

## Cats — Petlibro feeders

| Cat | Feeder entity prefix | Notes |
|---|---|---|
| Khaos | `khaos_smart_feeder` | last/next feed time, today's count + weight |
| Kiki | `kiki_smart_feeder` | same |
| Lunar | `lunar_smart_feeder` | same |

To check "fed today": `ha_get_state('sensor.<cat>_smart_feeder_today_s_feeding_times')` (returns count, e.g. `1`). To manual feed: `ha_call_service('button', 'press', 'button.<cat>_smart_feeder_manual_feed')`.

## Climate

- `climate.fire` — gas fireplace (on/off only via voice; full HVAC modes via HA UI)
- `climate.office_aircon` — Fujitsu split unit (set point, mode)

## Sensors of family Q&A interest

- `weather.forecast_home` — current weather + tomorrow forecast
- `calendar.stafford_borough_council` — next bin day
- `sensor.blood_sugar` — William's BG via Nightscout (mg/dL; ÷18 for mmol/L)
- `binary_sensor.fp2_office_presence` — Dan's office occupancy (mmWave)
- `sensor.fp2_office_illuminance` — office lux
- `binary_sensor.doorbell_camera_motion` — front-door motion

## Things this agent CAN'T do (per scope)

- Restart HA, reload integrations, edit YAML — admin operations
- Access cameras (no `camera.*` exposure)
- Unlock the Volvo (`lock.volvo_ex40_lock` is read-only here)
- Access entities not in the curated exposure list (~50 entities total)
EOF
echo "wrote TOOLS.md"
wc -l ~/.openclaw/workspace/agents/home-assistant/TOOLS.md
```

- [ ] **Step 2: Verify the OpenClaw agent picks it up**

OpenClaw agents read TOOLS.md on next invocation. No restart needed; the file is read at runtime.

---

## Phase 5 — Acceptance verification

### Task 10: Verify EVE Home can read HA state

**Files:** none (test via OpenClaw or shell)

- [ ] **Step 1: Test from shell with EVE Home's token**

```bash
source ~/data/config/load-secrets.sh
curl -s -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  "http://192.168.1.151:8123/api/states/binary_sensor.fp2_office_presence" \
  | python3 -c "import json,sys;d=json.load(sys.stdin);print(f\"state={d.get('state','?')} fname={d.get('attributes',{}).get('friendly_name','?')}\")"
```

Expected: `state=off fname=Fp2 Office Presence` (or `on` if currently present).

- [ ] **Step 2: Test denied access**

```bash
curl -s -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  "http://192.168.1.151:8123/api/states/camera.chickens_fluent" \
  | python3 -c "import json,sys;d=json.load(sys.stdin);print(d.get('message','no message'),' state=',d.get('state'))"
```

Expected: empty / 404 / "Entity not found" — `eve_home` doesn't have access to cameras (curated exposure excludes them).

### Task 11: Verify EVE Home can announce on a kitchen Echo

**Files:** none

- [ ] **Step 1: Test announce**

```bash
source ~/data/config/load-secrets.sh
curl -s -X POST -H "Authorization: Bearer $HA_EVE_HOME_TOKEN" \
  -H "Content-Type: application/json" \
  "http://192.168.1.151:8123/api/services/notify/kitchen_echo_show_announce" \
  -d '{"message": "EVE Home announcement test, please ignore"}'
```

Expected: empty `[]` HTTP body (means service call queued); Kitchen Echo Show speaks the message within 5 sec.

### Task 12: Verify HA Logbook attribution

**Files:** none (HA UI check)

- [ ] **Step 1: Open HA Logbook**

In HA UI: **Logbook → filter by entity `notify.kitchen_echo_show_announce`**. Find the test announce entry from Task 11.

- [ ] **Step 2: Confirm "by" attribution**

The entry should show "by **EVE Home**" (the user friendly_name from Task 2). If it does → audit attribution working as designed.

If it shows "by Home Assistant" or similar → LLAT-attributed actions aren't getting per-user attribution; document in spec §4.2 update.

### Task 13: Final sweep + Plane filing

**Files:** none

- [ ] **Step 1: Run all acceptance criteria from spec §6**

Mental checklist:

- [ ] ha-mcp 7.4.1 running (Task 0)
- [ ] `claude_code` HA user exists, admin, password+token in SOPS (Tasks 1, 3, 4)
- [ ] `eve_home` HA user exists, non-admin, password+token in SOPS, curated exposure applied (Tasks 2, 3, 4, 5)
- [ ] `~/.claude.json` registers `home-assistant` MCP; Claude Code session shows it loaded (Tasks 6, 7)
- [ ] `ha-tools` skill exists with 5 documented tool patterns (Task 8)
- [ ] EVE Home `TOOLS.md` updated with real inventory (Task 9)
- [ ] EVE Home can answer "what's in the office" via `ha_get_state` (Task 10)
- [ ] EVE Home can announce via `ha_announce` (Task 11)
- [ ] HA Logbook shows action attribution (Task 12)

- [ ] **Step 2: File a Plane HA issue capturing what landed (closes 2a)**

```bash
source ~/data/config/load-secrets.sh
HA_PROJECT=08a831d2-f1fc-4fcd-9dfd-ec6c4d9d73c1
DONE_STATE=019aa5bf-dfda-47e1-9dab-a0b603c7ded8
URL="$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/projects/$HA_PROJECT/issues/"

curl -s -X POST -H "x-api-key: $PLANE_API_KEY" -H "Content-Type: application/json" "$URL" -d @- <<JSON
{
  "name": "Track 2a — MCP Foundation (DONE)",
  "description_html": "<p>2a shipped 2026-05-03. ha-mcp add-on wired into ~/.claude.json for Claude Code; EVE Home (OpenClaw) wired via REST+WS with curated entity exposure; per-agent HA users (claude_code admin / eve_home non-admin) provide identity + audit. Spec at <code>~/Projects/RobboHome-skills-Claude/docs/superpowers/specs/2026-05-03-ha-track2a-mcp-foundation-design.md</code>; plan at <code>~/Projects/RobboHome-skills-Claude/docs/superpowers/plans/2026-05-03-ha-track2a-mcp-foundation-plan.md</code>.</p><p>Next: <strong>2b</strong> — design specific agent behaviours (BG-aware patterns, family Q&A phrasings, scheduled summaries, alert routing).</p>",
  "priority": "medium",
  "state": "${DONE_STATE}"
}
JSON
```

---

## Closing notes

**13 tasks across 5 phases.** Estimated 1–2 hours total, mostly UI clicks for users + tokens, then API calls for exposure + automation work.

Naturally next:
- **2b spec** — what specific things EVE Home / Claude Code should DO with HA. Day-in-the-life patterns, ambient triggers, family routines.
- **`unraid-mcp` registration in `~/.claude.json`** — separate ~5-min task, parked.
