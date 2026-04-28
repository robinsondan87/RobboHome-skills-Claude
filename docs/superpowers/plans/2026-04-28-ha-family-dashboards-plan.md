# HA Family Dashboards — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the four per-user Home Assistant dashboards (Nicola / Dan / William placeholder / Kiosk) on the Mushroom HACS card library, sharing one theme + scenes + scripts + macros, per the design spec.

**Architecture:** Approach B — per-user dashboards with a shared layer. Hybrid Lovelace mode (default stays `storage`, our four new dashboards are `yaml`). All Mushroom-based; reusable card snippets via `decluttering-card`; behaviour codified in `script.*` so dashboards stay thin.

**Tech Stack:** Home Assistant OS (KVM VM on Unraid svr001 at `192.168.1.151:8123`), Lovelace YAML, HACS frontend libraries (Mushroom, card-mod, decluttering-card, mini-graph-card, kiosk-mode), HACS integration `browser_mod`, Studio Code Server add-on for editing.

**Spec reference:** `docs/superpowers/specs/2026-04-28-ha-family-dashboards-design.md`

**Editing mechanism:** Studio Code Server HAOS add-on (web-based VS Code at `http://192.168.1.151:8123/_my_redirect/supervisor_addon?addon=core_configurator`). Install in Task 0 if not already present. All file paths in this plan are inside the HA `/config/` directory unless otherwise stated.

**Testing strategy:** HA's `Developer Tools → YAML → Check Configuration` validates syntax after every YAML edit. Each dashboard task ends with a browser load + console-error check. Scripts/scenes are exercised via `Developer Tools → Services` to confirm side effects on real entities. `git commit` after each green step (HA `/config/` repo — `git init`'d in Task 0 if not already).

---

## Phase 0 — Bootstrap & prerequisites

### Task 0: Choose & install editing surface

**Files:** none (HA UI work)

- [ ] **Step 1: Install Studio Code Server add-on**

In HA UI: **Settings → Add-ons → Add-on Store → Studio Code Server → Install**. Once installed, **Configuration → Show in sidebar: ON**, then **Start**. Open it from the sidebar — it loads a web-based VS Code session bound to `/config/`.

- [ ] **Step 2: Verify access**

In the Studio Code terminal:

```bash
ls /config/ && pwd
```

Expected: file listing shows `configuration.yaml`, `automations.yaml`, `scripts.yaml`, `scenes.yaml` etc., and `pwd` returns `/config`.

- [ ] **Step 3: Initialise git in /config/ if not already**

In the Studio Code terminal:

```bash
cd /config && git status 2>&1 || git init && git add -A && git commit -m "chore: snapshot before family-dashboards work"
```

If `git status` succeeds, skip the init/initial-commit; otherwise the chained command runs them. Confirm with `git log --oneline -3`.

### Task 1: Verify HACS is installed

**Files:** none

- [ ] **Step 1: Check for HACS**

In Studio Code terminal:

```bash
ls -d /config/custom_components/hacs && echo "HACS present" || echo "HACS missing"
```

Expected: `HACS present`. If `HACS missing`, install HACS via [the official bootstrap](https://hacs.xyz/docs/setup/download) before continuing — STOP HERE if missing, install, restart HA, re-run Step 1.

- [ ] **Step 2: Confirm HACS is reachable in the sidebar**

UI sidebar should now show **HACS**. Open it, confirm the Frontend + Integrations tabs load.

### Task 2: Install HACS dependencies

**Files:** none (HACS UI work)

- [ ] **Step 1: Install all six HACS deps**

In **HACS → Frontend → "Explore & download repositories"**, install in this order (each one prompts a browser refresh — finish one, refresh, then the next):

1. `Mushroom` (piitaya/lovelace-mushroom)
2. `card-mod` (thomasloven/lovelace-card-mod)
3. `decluttering-card` (RomRider/decluttering-card)
4. `Mini Graph Card` (kalkih/mini-graph-card)
5. `kiosk-mode` (NemesisRE/kiosk-mode)

Then **HACS → Integrations → "Explore & download"**: install:

6. `browser_mod` (thomasloven/hass-browser_mod)

- [ ] **Step 2: Restart Home Assistant**

**Settings → System → Restart → Restart Home Assistant** (full restart, not just reload).

- [ ] **Step 3: Verify resources are registered**

**Settings → Dashboards → Resources** should now list (post-restart, HACS auto-registers them):
- `/hacsfiles/lovelace-mushroom/mushroom.js`
- `/hacsfiles/lovelace-card-mod/card-mod.js`
- `/hacsfiles/decluttering-card/decluttering-card.js`
- `/hacsfiles/mini-graph-card/mini-graph-card-bundle.js`
- `/hacsfiles/kiosk-mode/kiosk-mode.js`

If any are missing, add them manually as **Resource type: JavaScript Module**.

- [ ] **Step 4: Commit**

```bash
cd /config && git add -A && git commit -m "chore: install HACS deps for family dashboards"
```

### Task 3: Audit & tag HA Areas

**Files:** none (HA UI work; affects entity registry)

- [ ] **Step 1: Ensure all 15 areas exist**

**Settings → Areas, Labels & Zones → Areas tab.** Required areas (create any missing):

Indoor: `Lounge`, `Library`, `Kitchen`, `Dining`, `Hall`, `Landing`, `Master Bed`, `William`, `Guest`, `Office`
Outdoor: `Driveway`, `Chickens`, `Side Garden`, `Side Path`, `Doorstep`

- [ ] **Step 2: Tag every controllable entity with an Area**

For each domain, walk the unassigned-area list and tag:

```
Settings → Devices & Services → Entities → filter by domain → bulk-edit "Area"
```

Domains to audit: `light`, `switch`, `scene`, `script`, `climate`, `media_player`, `lock`, `siren`, `camera`, `binary_sensor` (only the human-visible ones — sensors derived for automations don't need an area).

- [ ] **Step 3: Verify no controllable entity is arealess**

In Developer Tools → Template:

```jinja2
{{ states.light | rejectattr('entity_id', 'in',
    integration_entities('lovelace_includes')) | selectattr('entity_id') | list
   | map(attribute='entity_id') | select('eq', states.light | map(attribute='entity_id') | reject('match', '.*')) }}
```

Easier: in **Settings → Areas & Zones**, click each area and confirm the "Entities" count matches your expectation. Spot-check one or two rooms.

### Task 4: Create helper entities

**Files:** none (HA UI work)

- [ ] **Step 1: Create `input_boolean.doorbell_quiet_mode`**

**Settings → Devices & Services → Helpers → Create Helper → Toggle.** Name: `Doorbell Quiet Mode`. Icon: `mdi:bell-sleep`. Confirm `input_boolean.doorbell_quiet_mode` exists in Developer Tools → States.

- [ ] **Step 2: Create `timer.movie_doorbell_quiet`**

**Helpers → Create → Timer.** Name: `Movie Doorbell Quiet`. Duration: `02:00:00`. Confirm `timer.movie_doorbell_quiet` exists.

- [ ] **Step 3: Create `input_text.placeholder_william_status`**

**Helpers → Create → Text.** Name: `Placeholder William Status`. Initial value: `–` (en-dash). Confirm `input_text.placeholder_william_status` exists with state `–`.

- [ ] **Step 4: Commit**

```bash
cd /config && git add -A && git commit -m "feat: helpers for doorbell quiet, movie timer, William placeholder"
```

### Task 5: Create dedicated `kiosk` HA user

**Files:** none (HA UI work)

- [ ] **Step 1: Create user**

**Settings → People → Users tab → Add user.**
- Display name: `Kiosk`
- Username: `kiosk`
- Password: (generate strong, store in SOPS — `~/data/config/.secrets.env` as `HA_KIOSK_PASSWORD`)
- Administrator: **OFF**

- [ ] **Step 2: Verify**

Log out and log in as `kiosk` in a private browser window. Confirm the user can see dashboards but the **Settings** sidebar item is hidden (non-admin).

- [ ] **Step 3: Save kiosk password to SOPS**

On the Mac mini terminal (not Studio Code):

```bash
source ~/data/config/load-secrets.sh
sops ~/data/config/.secrets.env  # opens an editor
# add line: HA_KIOSK_PASSWORD=<the-password>
# save & quit; SOPS re-encrypts automatically
```

---

## Phase 1 — Shared layer: theme, scenes, scripts, automations

### Task 6: Write the Mushroom theme

**Files:**
- Create: `/config/themes/mushroom_robbohome.yaml`
- Modify: `/config/configuration.yaml` (add `frontend: themes: !include_dir_merge_named themes/`)

- [ ] **Step 1: Ensure themes folder exists**

In Studio Code terminal:

```bash
mkdir -p /config/themes
```

- [ ] **Step 2: Write the theme file**

Create `/config/themes/mushroom_robbohome.yaml` with:

```yaml
mushroom_robbohome:
  modes:
    light:
      primary-color: '#4a90c2'
      ha-card-border-radius: '20px'
    dark:
      primary-color: '#7ab8e0'
      ha-card-border-radius: '20px'
  card-mod-theme: mushroom_robbohome
  ha-card-box-shadow: none
  paper-font-body1_-_font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', sans-serif
```

- [ ] **Step 3: Register the themes folder in configuration.yaml**

If `configuration.yaml` doesn't already have a `frontend:` block referencing `themes/`, add:

```yaml
frontend:
  themes: !include_dir_merge_named themes/
```

If `frontend:` exists, add only the `themes:` line under it.

- [ ] **Step 4: Validate config + reload themes**

In **Developer Tools → YAML → Check Configuration → CHECK CONFIGURATION**. Expected: green "Configuration valid".

Then **Developer Tools → YAML → Frontend → Reload**.

- [ ] **Step 5: Verify theme applies**

**Profile (bottom-left) → Theme dropdown** should now list `mushroom_robbohome`. Select it. Page should reload with rounded cards + new accent color.

- [ ] **Step 6: Commit**

```bash
cd /config && git add themes/ configuration.yaml && git commit -m "feat: add mushroom_robbohome theme"
```

### Task 7: Define macro-target scenes

**Files:**
- Modify: `/config/scenes.yaml`

- [ ] **Step 1: Read existing scenes file**

In Studio Code terminal:

```bash
cat /config/scenes.yaml
```

Expected: existing scenes (chicken_coop, office_lights, library_lights). Note their structure — entity IDs vary per setup.

- [ ] **Step 2: Append three new scenes**

Append to `/config/scenes.yaml` (replace `<lounge_*>` and `<kitchen_*>` with actual light entity IDs from your `light` domain — find them via Developer Tools → States):

```yaml
- id: kitchen_morning_warm
  name: Kitchen Morning Warm
  icon: mdi:weather-sunny
  entities:
    light.<kitchen_overhead>:
      state: on
      brightness: 180
      color_temp: 380   # warm-ish
    light.<kitchen_under_cabinet>:
      state: on
      brightness: 140

- id: lounge_morning_warm
  name: Lounge Morning Warm
  icon: mdi:weather-sunny
  entities:
    light.<lounge_main>:
      state: on
      brightness: 160
      color_temp: 400
    light.<lounge_lamp>:
      state: on
      brightness: 200

- id: lounge_movie
  name: Lounge Movie
  icon: mdi:movie-roll
  entities:
    light.<lounge_main>:
      state: off
    light.<lounge_lamp>:
      state: on
      brightness: 50
      color_temp: 470   # very warm
```

- [ ] **Step 3: Reload scenes**

**Developer Tools → YAML → Scenes → Reload.**

- [ ] **Step 4: Test each scene fires**

For each (`kitchen_morning_warm`, `lounge_morning_warm`, `lounge_movie`):
- **Developer Tools → Services → `scene.turn_on` → entity: `scene.<id>` → CALL SERVICE**
- Visually confirm the lights match the intended state. Adjust `brightness`/`color_temp` if needed.

- [ ] **Step 5: Commit**

```bash
cd /config && git add scenes.yaml && git commit -m "feat: macro-target scenes (kitchen/lounge morning, lounge movie)"
```

### Task 8: Define per-room scenes (long-press targets)

**Files:**
- Modify: `/config/scenes.yaml`

- [ ] **Step 1: Append per-room "default" scenes for the 6 primary rooms**

For each of `lounge`, `library`, `kitchen`, `dining`, `hall`, `landing`, append a scene that turns all that room's lights on at a sensible default. Replace `<room_*>` with actual entity IDs:

```yaml
- id: room_lounge_default
  name: Lounge Default
  icon: mdi:sofa
  entities:
    light.<lounge_main>: { state: on, brightness: 200 }
    light.<lounge_lamp>: { state: on, brightness: 180 }

- id: room_library_default
  name: Library Default
  icon: mdi:bookshelf
  entities:
    light.<library_main>: { state: on, brightness: 200 }

- id: room_kitchen_default
  name: Kitchen Default
  icon: mdi:countertop
  entities:
    light.<kitchen_overhead>: { state: on, brightness: 230 }
    light.<kitchen_under_cabinet>: { state: on, brightness: 200 }

- id: room_dining_default
  name: Dining Default
  icon: mdi:chair-rolling
  entities:
    light.<dining_main>: { state: on, brightness: 200 }

- id: room_hall_default
  name: Hall Default
  icon: mdi:door
  entities:
    light.<hall_main>: { state: on, brightness: 220 }

- id: room_landing_default
  name: Landing Default
  icon: mdi:stairs
  entities:
    light.<landing_main>: { state: on, brightness: 200 }
```

- [ ] **Step 2: Reload + test each**

Reload scenes (Dev Tools → YAML → Scenes → Reload). For each scene, call `scene.turn_on` and visually confirm.

- [ ] **Step 3: Commit**

```bash
cd /config && git add scenes.yaml && git commit -m "feat: per-room default scenes for the 6 primary rooms"
```

### Task 9: Write `script.goodnight`

**Files:**
- Modify: `/config/scripts.yaml`

- [ ] **Step 1: Append the script**

Append to `/config/scripts.yaml`. Replace `<...>` placeholders with actual entity IDs (Sonos players, etc.):

```yaml
goodnight:
  alias: Goodnight Macro
  icon: mdi:weather-night
  mode: single
  sequence:
    - service: light.turn_off
      target:
        area_id:
          - lounge
          - library
          - kitchen
          - dining
          - office
    - service: light.turn_on
      target:
        area_id:
          - hall
          - landing
      data:
        brightness_pct: 15
    - service: climate.turn_off
      target:
        entity_id: climate.fire
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.doorbell_quiet_mode
    - service: media_player.media_pause
      target:
        entity_id:
          - media_player.office
          - media_player.roam
          - media_player.office_sonos
      continue_on_error: true
    - service: notify.mobile_app_dans_iphone
      data:
        title: Goodnight
        message: "Goodnight done — lights off, fire off, doorbell quiet."
```

- [ ] **Step 2: Reload scripts**

**Developer Tools → YAML → Scripts → Reload.**

- [ ] **Step 3: Test**

**Developer Tools → Services → `script.goodnight` → CALL SERVICE.** Watch:
- All 6 areas' lights go off / hall+landing dim to 15%
- Fire turns off
- Doorbell quiet helper turns on (verify in States)
- Phone notification arrives

- [ ] **Step 4: Commit**

```bash
cd /config && git add scripts.yaml && git commit -m "feat: script.goodnight macro"
```

### Task 10: Write `script.morning`

**Files:**
- Modify: `/config/scripts.yaml`

- [ ] **Step 1: Append**

Append (note: Step 2's Alexa call is a logged warning until the Alexa↔HA TODO lands — `media_player.kitchen_echo` doesn't exist yet, so we wrap it in `continue_on_error`):

```yaml
morning:
  alias: Morning Macro
  icon: mdi:weather-sunny
  mode: single
  sequence:
    - service: scene.turn_on
      target:
        entity_id:
          - scene.kitchen_morning_warm
          - scene.lounge_morning_warm

    # Kitchen Alexa wakeup — no-op until Alexa Media Player integration lands
    - alias: Kitchen Alexa BBC R4
      continue_on_error: true
      service: media_player.play_media
      target:
        entity_id: media_player.kitchen_echo
      data:
        media_content_id: "tunein.com/radio/BBC-Radio-4-933/"
        media_content_type: tunein

    # Bin-day reminder
    - choose:
        - conditions:
            - condition: template
              value_template: >
                {% set next = state_attr('calendar.stafford_borough_council', 'start_time') %}
                {{ next is not none and next.startswith(now().strftime('%Y-%m-%d')) }}
          sequence:
            - service: notify.mobile_app_dans_iphone
              data:
                title: Bin day
                message: >
                  {{ state_attr('calendar.stafford_borough_council', 'message') }}
            - service: notify.mobile_app_nics_iphone_16
              data:
                title: Bin day
                message: >
                  {{ state_attr('calendar.stafford_borough_council', 'message') }}

    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.doorbell_quiet_mode

    - service: notify.mobile_app_dans_iphone
      data: { title: Morning, message: "Morning macro fired." }
    - service: notify.mobile_app_nics_iphone_16
      data: { title: Morning, message: "Morning macro fired." }
```

- [ ] **Step 2: Reload + test**

Reload scripts. Call `script.morning` from Dev Tools → Services. Verify:
- Kitchen + Lounge scenes activate
- Alexa step logs a warning (entity not found) — does not halt the script
- Doorbell quiet flips off
- Both phones get a notification

- [ ] **Step 3: Commit**

```bash
cd /config && git add scripts.yaml && git commit -m "feat: script.morning macro"
```

### Task 11: Write `script.movie`

**Files:**
- Modify: `/config/scripts.yaml`

- [ ] **Step 1: Append**

```yaml
movie:
  alias: Movie Macro
  icon: mdi:movie-roll
  mode: single
  sequence:
    - service: scene.turn_on
      target:
        entity_id: scene.lounge_movie
    - service: media_player.media_pause
      target:
        entity_id: media_player.roam
      continue_on_error: true
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.doorbell_quiet_mode
    - service: timer.start
      target:
        entity_id: timer.movie_doorbell_quiet
    - service: notify.mobile_app_dans_iphone
      data: { title: Movie, message: "Movie mode for 2h." }
```

- [ ] **Step 2: Add automation to clear doorbell quiet when timer ends**

Append to `/config/automations.yaml`:

```yaml
- id: movie_timer_clear_doorbell_quiet
  alias: "Movie: clear doorbell quiet on timer end"
  trigger:
    - platform: event
      event_type: timer.finished
      event_data: { entity_id: timer.movie_doorbell_quiet }
  action:
    - service: input_boolean.turn_off
      target: { entity_id: input_boolean.doorbell_quiet_mode }
```

- [ ] **Step 3: Reload scripts + automations**

Dev Tools → YAML → Scripts → Reload. Then YAML → Automations → Reload.

- [ ] **Step 4: Test**

Call `script.movie`. Verify:
- Lounge movie scene activates
- `input_boolean.doorbell_quiet_mode` turns on
- `timer.movie_doorbell_quiet` shows status `active` with ~2h remaining
- Phone notification arrives

Optional: change timer duration to `00:00:30` temporarily, re-run, wait 30s, confirm doorbell quiet flips off automatically. Then revert duration to `02:00:00`.

- [ ] **Step 5: Commit**

```bash
cd /config && git add scripts.yaml automations.yaml && git commit -m "feat: script.movie + timer-end clear-doorbell-quiet automation"
```

### Task 12: Schedule automations for Goodnight + Morning

**Files:**
- Modify: `/config/automations.yaml`

- [ ] **Step 1: Append both schedule automations**

```yaml
- id: goodnight_late_fallback
  alias: "Goodnight: fallback at 23:00 if not already run"
  trigger:
    - platform: time
      at: "23:00:00"
  condition:
    - condition: template
      value_template: >
        {% set last = states.script.goodnight.attributes.last_triggered %}
        {{ last is none or (now() - last).total_seconds() > 7200 }}
  action:
    - service: script.goodnight

- id: morning_weekday
  alias: "Morning: 06:30 Mon–Fri"
  trigger:
    - platform: time
      at: "06:30:00"
  condition:
    - condition: time
      weekday: [mon, tue, wed, thu, fri]
  action:
    - service: script.morning
```

- [ ] **Step 2: Reload automations**

Dev Tools → YAML → Automations → Reload.

- [ ] **Step 3: Smoke-test**

In Developer Tools → States, search for `automation.goodnight_late_fallback` and `automation.morning_weekday` — both should show `state: on`. Trigger each manually via the three-dot menu → "Run actions" to confirm they call their script (you'll see lights/scenes change).

- [ ] **Step 4: Commit**

```bash
cd /config && git add automations.yaml && git commit -m "feat: schedule automations for goodnight (23:00) and morning (06:30 weekday)"
```

---

## Phase 2 — Lovelace registration

### Task 13: Register the four dashboards in `configuration.yaml`

**Files:**
- Modify: `/config/configuration.yaml`

- [ ] **Step 1: Add the lovelace block**

Add (or merge into the existing `lovelace:` block — the default mode stays `storage` so the existing dashboard isn't disturbed):

```yaml
lovelace:
  mode: storage
  dashboards:
    nicola:
      mode: yaml
      filename: dashboards/nicola.yaml
      title: Home
      icon: mdi:home
      show_in_sidebar: true
      require_admin: false
    dan:
      mode: yaml
      filename: dashboards/dan.yaml
      title: Power
      icon: mdi:tools
      show_in_sidebar: true
      require_admin: false
    william:
      mode: yaml
      filename: dashboards/william.yaml
      title: Will
      icon: mdi:account-tie
      show_in_sidebar: true
      require_admin: false
    kiosk:
      mode: yaml
      filename: dashboards/kiosk.yaml
      title: Kiosk
      icon: mdi:tablet
      show_in_sidebar: false
      require_admin: false
```

- [ ] **Step 2: Create empty placeholder dashboard files**

In Studio Code terminal:

```bash
mkdir -p /config/dashboards /config/lovelace_includes
for f in nicola dan william kiosk; do
  cat > /config/dashboards/$f.yaml <<EOF
title: $f (placeholder)
views:
  - title: Home
    icon: mdi:home
    cards:
      - type: markdown
        content: "Dashboard $f — placeholder, not yet implemented."
EOF
done
```

- [ ] **Step 3: Validate config**

Dev Tools → YAML → Check Configuration. Expected: green.

- [ ] **Step 4: Restart HA (full restart — Lovelace registration requires it)**

Settings → System → Restart → Restart Home Assistant.

- [ ] **Step 5: Verify all four dashboards appear in the sidebar**

After login, sidebar should show **Home**, **Power**, **Will** (Kiosk hidden by `show_in_sidebar: false`). Each loads the placeholder markdown without console errors.

The Kiosk dashboard URL is `http://192.168.1.151:8123/kiosk/0` — verify it loads when navigated to directly.

- [ ] **Step 6: Commit**

```bash
cd /config && git add configuration.yaml dashboards/ && git commit -m "feat: register four yaml-mode dashboards (placeholders)"
```

### Task 14: Set per-user default dashboards

**Files:** none (HA UI work, per user)

- [ ] **Step 1: Set Nicola's default**

Log in as Nicola → **Profile (bottom-left) → Default dashboard → "Home" (`nicola`)** → Save. Confirm: log out and log back in as Nicola; she lands on `/nicola/0`.

- [ ] **Step 2: Set Dan's default**

Log in as Dan (you) → **Profile → Default dashboard → "Power" (`dan`)** → Save.

- [ ] **Step 3: Set kiosk user's default**

Log in as `kiosk` → **Profile → Default dashboard → "Kiosk" (`kiosk`)** → Save. Then **Profile → Hide sidebar (kiosk-mode is full-page anyway)**.

---

## Phase 3 — Reusable card snippets (decluttering-card)

### Task 15: Write `chips_status.yaml`

**Files:**
- Create: `/config/lovelace_includes/chips_status.yaml`

- [ ] **Step 1: Write the include**

> Note: decluttering-card uses simple `[[var]]` substitution, not conditionals. So this file defines only the **base** chip bar shared across all four dashboards. Dan's extra EV + cameras chips are added inline in `dashboards/dan.yaml` — no separate template needed.

```yaml
default: []
card:
  type: custom:mushroom-chips-card
  alignment: center
  chips:
    - type: entity
      entity: person.dan
      icon_color: blue
      content_info: none
      tap_action: { action: more-info }
    - type: entity
      entity: person.nicola
      icon_color: pink
      content_info: none
      tap_action: { action: more-info }
    - type: template
      icon: mdi:account-school
      icon_color: orange
      content: "Will {{ states('input_text.placeholder_william_status') }}"
      tap_action: { action: more-info, entity: input_text.placeholder_william_status }
    - type: weather
      entity: weather.forecast_home
      show_temperature: true
      show_conditions: true
    - type: entity
      entity: binary_sensor.doorbell_camera_motion
      icon: mdi:doorbell
      content_info: none
    - type: entity
      entity: climate.fire
      icon: mdi:fire
      content_info: none
```

If `binary_sensor.doorbell_camera_motion` doesn't match your actual entity ID, find the right one with **Developer Tools → States → filter "doorbell"** and substitute.

- [ ] **Step 2: Register decluttering-card templates in the lovelace dashboard files**

Decluttering-card templates are referenced inside each dashboard YAML's `decluttering_templates:` block. We won't load them globally — we'll inline the templates section into each dashboard. Skip global registration.

- [ ] **Step 3: Commit**

```bash
cd /config && git add lovelace_includes/chips_status.yaml && git commit -m "feat: chips_status include (basic chip bar)"
```

### Task 16: Write `tile_energy.yaml`

**Files:**
- Create: `/config/lovelace_includes/tile_energy.yaml`

- [ ] **Step 1: Write**

```yaml
default: []
card:
  type: vertical-stack
  cards:
    - type: custom:mushroom-template-card
      primary: "{{ states('sensor.electricity_cost_rate') }} p/kWh"
      secondary: >-
        Today {{ states('sensor.electricity_today') | round(1) }} kWh
        {% if is_state('binary_sensor.import_rising_fast', 'on') %}
          ⚡ Import rising
        {% endif %}
      icon: mdi:lightning-bolt
      icon_color: >-
        {% if is_state('binary_sensor.import_rising_fast', 'on') %}red
        {% else %}amber{% endif %}
      tap_action:
        action: navigate
        navigation_path: /dan/energy
    - type: custom:mini-graph-card
      entities:
        - entity: sensor.electricity_cost_rate
      hours_to_show: 24
      points_per_hour: 2
      line_color: amber
      show:
        labels: false
        name: false
        icon: false
        state: false
      height: 60
```

- [ ] **Step 2: Commit**

```bash
cd /config && git add lovelace_includes/tile_energy.yaml && git commit -m "feat: tile_energy include"
```

### Task 17: Write `tile_weather_bins.yaml`

**Files:**
- Create: `/config/lovelace_includes/tile_weather_bins.yaml`

- [ ] **Step 1: Write**

```yaml
default: []
card:
  type: custom:mushroom-template-card
  primary: >-
    Today {{ state_attr('weather.forecast_home', 'temperature') }}°
    {{ states('weather.forecast_home') }}
  secondary: >-
    {% set ev = state_attr('calendar.stafford_borough_council', 'message') %}
    {% set st = state_attr('calendar.stafford_borough_council', 'start_time') %}
    {% if ev and st %}
      Next bin: {{ ev }} ({{ as_timestamp(st) | timestamp_custom('%a %-d %b') }})
    {% else %}
      No bin scheduled
    {% endif %}
  icon: mdi:weather-partly-cloudy
  icon_color: blue
  tap_action: { action: more-info, entity: weather.forecast_home }
```

- [ ] **Step 2: Commit**

```bash
cd /config && git add lovelace_includes/tile_weather_bins.yaml && git commit -m "feat: tile_weather_bins include"
```

### Task 18: Write `tile_laundry.yaml`

**Files:**
- Create: `/config/lovelace_includes/tile_laundry.yaml`

- [ ] **Step 1: Write**

```yaml
default: []
card:
  type: horizontal-stack
  cards:
    - type: custom:mushroom-template-card
      primary: Washer
      secondary: >-
        {% if is_state('binary_sensor.washing_machine_running', 'on') %}
          Running
        {% else %}
          Idle
        {% endif %}
      icon: mdi:washing-machine
      icon_color: >-
        {% if is_state('binary_sensor.washing_machine_running', 'on') %}cyan
        {% else %}grey{% endif %}
    - type: custom:mushroom-template-card
      primary: Dryer
      secondary: >-
        {% if is_state('binary_sensor.tumble_dryer_running', 'on') %}
          Running
        {% else %}
          Idle
        {% endif %}
      icon: mdi:tumble-dryer
      icon_color: >-
        {% if is_state('binary_sensor.tumble_dryer_running', 'on') %}orange
        {% else %}grey{% endif %}
```

- [ ] **Step 2: Commit**

```bash
cd /config && git add lovelace_includes/tile_laundry.yaml && git commit -m "feat: tile_laundry include"
```

### Task 19: Write `tile_room_lights.yaml`

**Files:**
- Create: `/config/lovelace_includes/tile_room_lights.yaml`

- [ ] **Step 1: Write**

```yaml
# Variables:
#   area_id:  HA area slug (lounge, kitchen, library, dining, hall, landing, …)
#   name:     Display name (Lounge, Kitchen, …)
#   icon:     mdi icon
#   scene_id: scene to fire on tap (e.g. scene.room_lounge_default)
default:
  - icon: mdi:lightbulb
card:
  type: custom:mushroom-light-card
  entity: >-
    {{ expand(area_entities('[[area_id]]'))
       | selectattr('domain', 'equalto', 'light')
       | map(attribute='entity_id') | list | first }}
  name: "[[name]]"
  icon: "[[icon]]"
  show_brightness_control: false
  use_light_color: false
  layout: vertical
  fill_container: true
  tap_action:
    action: call-service
    service: scene.turn_on
    target: { entity_id: "[[scene_id]]" }
  hold_action:
    action: more-info
```

> NOTE: This card displays *one* light from the area as a representative — the on/off state and brightness ring reflect that light only. For an averaged/all-lights view, use `light.<area>_group` (HA group) once those groups are created. For now, single-representative is fine.

- [ ] **Step 2: Commit**

```bash
cd /config && git add lovelace_includes/tile_room_lights.yaml && git commit -m "feat: tile_room_lights include"
```

### Task 20: Write `tile_heating.yaml`

**Files:**
- Create: `/config/lovelace_includes/tile_heating.yaml`

- [ ] **Step 1: Write `tile_heating.yaml`** (Nicola / William / Kiosk variant — fire only)

```yaml
default: []
card:
  type: custom:mushroom-climate-card
  entity: climate.fire
  name: Fire
  icon: mdi:fire
  show_temperature_control: false
  hvac_modes:
    - "off"
    - "heat"
```

- [ ] **Step 2: Write `lovelace_includes/tile_heating_dan.yaml`** (Dan's variant — fire + office aircon)

```yaml
default: []
card:
  type: vertical-stack
  cards:
    - type: custom:mushroom-climate-card
      entity: climate.fire
      name: Fire
      icon: mdi:fire
      show_temperature_control: false
      hvac_modes: ["off", "heat"]
    - type: custom:mushroom-climate-card
      entity: climate.office_aircon
      name: Office AC
      icon: mdi:air-conditioner
```

- [ ] **Step 2: Commit**

```bash
cd /config && git add lovelace_includes/ && git commit -m "feat: tile_heating + tile_heating_dan includes"
```

### Task 21: Write `tile_macros.yaml`

**Files:**
- Create: `/config/lovelace_includes/tile_macros.yaml`

- [ ] **Step 1: Write**

```yaml
default: []
card:
  type: horizontal-stack
  cards:
    - type: custom:mushroom-template-card
      primary: Goodnight
      icon: mdi:weather-night
      icon_color: indigo
      tap_action:
        action: call-service
        service: script.goodnight
        confirmation: false
        haptic: success
    - type: custom:mushroom-template-card
      primary: Morning
      icon: mdi:weather-sunny
      icon_color: amber
      tap_action:
        action: call-service
        service: script.morning
        confirmation: false
        haptic: success
    - type: custom:mushroom-template-card
      primary: Movie
      icon: mdi:movie-roll
      icon_color: deep-purple
      tap_action:
        action: call-service
        service: script.movie
        confirmation: false
        haptic: success
```

- [ ] **Step 2: Commit**

```bash
cd /config && git add lovelace_includes/tile_macros.yaml && git commit -m "feat: tile_macros include"
```

---

## Phase 4 — Build dashboards

### Task 22: Build `dashboards/nicola.yaml` Home view

**Files:**
- Modify: `/config/dashboards/nicola.yaml`

- [ ] **Step 1: Replace placeholder with Home view**

Overwrite `/config/dashboards/nicola.yaml`:

```yaml
title: Home
decluttering_templates:
  chips_status: !include ../lovelace_includes/chips_status.yaml
  tile_energy: !include ../lovelace_includes/tile_energy.yaml
  tile_weather_bins: !include ../lovelace_includes/tile_weather_bins.yaml
  tile_laundry: !include ../lovelace_includes/tile_laundry.yaml
  tile_room_lights: !include ../lovelace_includes/tile_room_lights.yaml
  tile_heating: !include ../lovelace_includes/tile_heating.yaml
  tile_macros: !include ../lovelace_includes/tile_macros.yaml

views:
  - title: Home
    path: home
    icon: mdi:home
    type: sections
    max_columns: 1
    sections:
      - type: grid
        cards:
          - type: custom:decluttering-card
            template: chips_status
          - type: custom:decluttering-card
            template: tile_energy
          - type: custom:decluttering-card
            template: tile_weather_bins
          - type: custom:decluttering-card
            template: tile_laundry
          - type: grid
            columns: 3
            square: false
            cards:
              - type: custom:decluttering-card
                template: tile_room_lights
                variables:
                  area_id: lounge
                  name: Lounge
                  icon: mdi:sofa
                  scene_id: scene.room_lounge_default
              - type: custom:decluttering-card
                template: tile_room_lights
                variables:
                  area_id: kitchen
                  name: Kitchen
                  icon: mdi:countertop
                  scene_id: scene.room_kitchen_default
              - type: custom:decluttering-card
                template: tile_room_lights
                variables:
                  area_id: library
                  name: Library
                  icon: mdi:bookshelf
                  scene_id: scene.room_library_default
              - type: custom:decluttering-card
                template: tile_room_lights
                variables:
                  area_id: dining
                  name: Dining
                  icon: mdi:chair-rolling
                  scene_id: scene.room_dining_default
              - type: custom:decluttering-card
                template: tile_room_lights
                variables:
                  area_id: hall
                  name: Hall
                  icon: mdi:door
                  scene_id: scene.room_hall_default
              - type: custom:decluttering-card
                template: tile_room_lights
                variables:
                  area_id: landing
                  name: Landing
                  icon: mdi:stairs
                  scene_id: scene.room_landing_default
          - type: custom:decluttering-card
            template: tile_heating
          - type: custom:decluttering-card
            template: tile_macros
```

- [ ] **Step 2: Validate + restart HA**

Dev Tools → YAML → Check Configuration → green. **Settings → System → Restart → Restart Home Assistant** — YAML-mode dashboards require a full restart to pick up structural changes (incl. new `decluttering_templates`).

- [ ] **Step 3: Load `/nicola/home` in browser**

Open `http://192.168.1.151:8123/nicola/home` in the HA Companion app on Nicola's iPhone (or simulate in a private browser as Nicola).

Verify:
- Theme renders with rounded cards + sky blue accent
- All 7 sections appear in order: chips → energy → weather/bins → laundry → 6-room grid → heating → macros
- No errors in browser console (F12)
- Each room tile shows current state of one of its lights

- [ ] **Step 4: Tap-test each tile**

- Chips: tap each → "More info" sheet appears
- Energy NOW: tap → navigates to `/dan/energy` (will be empty until Task 25 — that's fine)
- Weather + Bins: tap → forecast popup
- Laundry: visual state matches `binary_sensor.washing_machine_running` etc.
- Room tiles: tap = room scene fires (lights change); long-press = full controls sheet
- Heating: tap = toggles `climate.fire`
- Macros: tap each = corresponding script fires (you've already tested the scripts in Phase 1)

- [ ] **Step 5: Commit**

```bash
cd /config && git add dashboards/nicola.yaml && git commit -m "feat: Nicola dashboard Home view"
```

### Task 23: Add Nicola's other tabs

**Files:**
- Modify: `/config/dashboards/nicola.yaml`

- [ ] **Step 1: Append three more views (All Rooms, Outside, More)**

Append to `views:` (after the `- title: Home` block):

```yaml
  - title: All Rooms
    path: rooms
    icon: mdi:home-group
    cards:
      - type: custom:mushroom-title-card
        title: Bedrooms & Office
      - type: grid
        columns: 2
        cards:
          - type: custom:decluttering-card
            template: tile_room_lights
            variables: { area_id: master_bed, name: Master Bed,  icon: mdi:bed,            scene_id: scene.room_master_bed_default }
          - type: custom:decluttering-card
            template: tile_room_lights
            variables: { area_id: william,    name: William,     icon: mdi:account-school, scene_id: scene.room_william_default }
          - type: custom:decluttering-card
            template: tile_room_lights
            variables: { area_id: guest,      name: Guest,       icon: mdi:bed-empty,      scene_id: scene.room_guest_default }
          - type: custom:decluttering-card
            template: tile_room_lights
            variables: { area_id: office,     name: Office,      icon: mdi:desk,           scene_id: scene.room_office_default }
      - type: custom:mushroom-title-card
        title: Living Spaces (full controls)
      - type: grid
        columns: 2
        cards:
          - type: area
            area: lounge
          - type: area
            area: kitchen
          - type: area
            area: library
          - type: area
            area: dining

  - title: Outside
    path: outside
    icon: mdi:tree
    cards:
      - type: grid
        columns: 2
        cards:
          - type: picture-entity
            entity: camera.chickens_fluent
            name: Chickens
          - type: picture-entity
            entity: camera.side_camera_fluent
            name: Side Garden
          - type: picture-entity
            entity: camera.side_path_fluent
            name: Side Path
          - type: picture-entity
            entity: camera.driveway_camera
            name: Driveway

  - title: More
    path: more
    icon: mdi:dots-horizontal
    cards:
      - type: custom:mushroom-title-card
        title: Volvo EX40
      - type: entities
        entities:
          - sensor.volvo_ex40_battery_charge_level
          - sensor.volvo_ex40_battery_distance_to_empty
          - lock.volvo_ex40_lock
          - device_tracker.volvo_ex40_location
      - type: custom:mushroom-title-card
        title: Sonos
      - type: entities
        entities:
          - media_player.office
          - media_player.roam
          - media_player.office_sonos
```

> NOTE: The "All Rooms" tab references `scene.room_master_bed_default`, `scene.room_william_default`, etc. — those don't exist yet (Task 8 only created the 6 primary). Either:
> (a) defer this view's bedroom rows to a Task 23.5 once you create those scenes, or
> (b) create the four missing scenes inline now in `scenes.yaml` following Task 8's pattern.

For implementation: do **(b)** before reloading.

- [ ] **Step 2: Append the four missing room-default scenes**

Append to `/config/scenes.yaml`:

```yaml
- id: room_master_bed_default
  name: Master Bed Default
  entities:
    light.<master_bed_main>: { state: on, brightness: 180 }

- id: room_william_default
  name: William Room Default
  entities:
    switch.<sonoff_williams_room>: { state: on }   # the existing Sonoff plug

- id: room_guest_default
  name: Guest Default
  entities:
    switch.<sonoff_guest_bedroom_light>: { state: on }

- id: room_office_default
  name: Office Default
  entities:
    light.<office_main>: { state: on, brightness: 200 }
```

Replace `<...>` with actual entity IDs.

- [ ] **Step 3: Reload + verify**

Reload scenes (Dev Tools → YAML → Scenes → Reload). Reload Lovelace (Dev Tools → YAML → Lovelace → Reload).

Open `/nicola/rooms`, `/nicola/outside`, `/nicola/more` — confirm each renders without errors.

- [ ] **Step 4: Commit**

```bash
cd /config && git add dashboards/nicola.yaml scenes.yaml && git commit -m "feat: Nicola dashboard - All Rooms, Outside, More tabs + bedroom scenes"
```

### Task 24: Build `dashboards/dan.yaml` Home view (sketched-tabs variant)

**Files:**
- Modify: `/config/dashboards/dan.yaml`

- [ ] **Step 1: Overwrite with Dan's Home + sketched tabs**

```yaml
title: Power
decluttering_templates:
  chips_status: !include ../lovelace_includes/chips_status.yaml
  tile_energy: !include ../lovelace_includes/tile_energy.yaml
  tile_weather_bins: !include ../lovelace_includes/tile_weather_bins.yaml
  tile_laundry: !include ../lovelace_includes/tile_laundry.yaml
  tile_room_lights: !include ../lovelace_includes/tile_room_lights.yaml
  tile_heating_dan: !include ../lovelace_includes/tile_heating_dan.yaml
  tile_macros: !include ../lovelace_includes/tile_macros.yaml

views:
  - title: Home
    path: home
    icon: mdi:home
    type: sections
    max_columns: 1
    sections:
      - type: grid
        cards:
          - type: custom:decluttering-card
            template: chips_status
          # Dan-specific extra chips
          - type: custom:mushroom-chips-card
            chips:
              - type: entity
                entity: sensor.volvo_ex40_battery_charge_level
                icon: mdi:car-electric
                icon_color: green
              - type: entity
                entity: sensor.volvo_ex40_battery_distance_to_empty
                icon: mdi:map-marker-distance
              - type: entity
                entity: binary_sensor.doorbell_camera_motion
                icon: mdi:cctv
          - type: custom:decluttering-card
            template: tile_energy
          - type: custom:decluttering-card
            template: tile_weather_bins
          - type: custom:decluttering-card
            template: tile_laundry
          - type: grid
            columns: 3
            square: false
            cards:
              - type: custom:decluttering-card
                template: tile_room_lights
                variables: { area_id: lounge,   name: Lounge,   icon: mdi:sofa,           scene_id: scene.room_lounge_default }
              - type: custom:decluttering-card
                template: tile_room_lights
                variables: { area_id: kitchen,  name: Kitchen,  icon: mdi:countertop,     scene_id: scene.room_kitchen_default }
              - type: custom:decluttering-card
                template: tile_room_lights
                variables: { area_id: library,  name: Library,  icon: mdi:bookshelf,      scene_id: scene.room_library_default }
              - type: custom:decluttering-card
                template: tile_room_lights
                variables: { area_id: dining,   name: Dining,   icon: mdi:chair-rolling,  scene_id: scene.room_dining_default }
              - type: custom:decluttering-card
                template: tile_room_lights
                variables: { area_id: hall,     name: Hall,     icon: mdi:door,           scene_id: scene.room_hall_default }
              - type: custom:decluttering-card
                template: tile_room_lights
                variables: { area_id: landing,  name: Landing,  icon: mdi:stairs,         scene_id: scene.room_landing_default }
          - type: custom:decluttering-card
            template: tile_heating_dan
          - type: custom:decluttering-card
            template: tile_macros

  - title: All Rooms
    path: rooms
    icon: mdi:home-group
    cards: [{ type: markdown, content: "Power-user All Rooms — Track-1.5 follow-on" }]

  - title: Outside
    path: outside
    icon: mdi:tree
    cards: [{ type: markdown, content: "Outside (cameras + driveway scripts) — Track-1.5 follow-on" }]

  - title: Energy
    path: energy
    icon: mdi:lightning-bolt
    cards: [{ type: markdown, content: "Energy detail — Track-1.5 follow-on" }]

  - title: Volvo
    path: volvo
    icon: mdi:car-electric
    cards: [{ type: markdown, content: "Volvo EX40 detail — Track-1.5 follow-on" }]

  - title: Media
    path: media
    icon: mdi:speaker
    cards: [{ type: markdown, content: "Media (Sonos) — Track-1.5 follow-on" }]

  - title: System
    path: system
    icon: mdi:tools
    cards: [{ type: markdown, content: "System health (HA + Unraid + NR) — Track-1.5 follow-on" }]
```

- [ ] **Step 2: Reload + verify**

Reload Lovelace. Load `/dan/home` — verify it renders Dan's Home with extra EV/cameras chips and the Dan-specific heating tile (fire + office aircon). Other tabs render markdown placeholders.

- [ ] **Step 3: Commit**

```bash
cd /config && git add dashboards/dan.yaml && git commit -m "feat: Dan dashboard Home view + tab placeholders"
```

### Task 25: Build `dashboards/william.yaml` placeholder

**Files:**
- Modify: `/config/dashboards/william.yaml`

- [ ] **Step 1: Overwrite with placeholder**

```yaml
title: Will
views:
  - title: Home
    path: home
    icon: mdi:home
    cards:
      - type: custom:mushroom-template-card
        primary: William
        secondary: Dashboard coming soon
        icon: mdi:account-school
        icon_color: orange
      - type: markdown
        content: |
          # William's Dashboard

          This dashboard goes live once William's Pixel 8a + iPad are added to Home Assistant.

          For now, talk to **William's Agent** on WhatsApp:
          [Open WhatsApp](https://wa.me/447884926246)
```

- [ ] **Step 2: Reload + verify**

Load `/william/home`. Verify the placeholder renders.

- [ ] **Step 3: Commit**

```bash
cd /config && git add dashboards/william.yaml && git commit -m "feat: William dashboard placeholder"
```

### Task 26: Build `dashboards/kiosk.yaml`

**Files:**
- Modify: `/config/dashboards/kiosk.yaml`

- [ ] **Step 1: Overwrite with kiosk layout**

```yaml
title: Kiosk
kiosk_mode:
  hide_header: true
  hide_sidebar: true

decluttering_templates:
  tile_room_lights: !include ../lovelace_includes/tile_room_lights.yaml
  tile_heating: !include ../lovelace_includes/tile_heating.yaml
  tile_macros: !include ../lovelace_includes/tile_macros.yaml

views:
  - title: Home
    path: home
    icon: mdi:tablet
    type: panel
    cards:
      - type: grid
        columns: 2
        square: false
        cards:
          # LEFT COLUMN — glance
          - type: vertical-stack
            cards:
              - type: custom:mushroom-template-card
                primary: "{{ now().strftime('%H:%M') }}"
                secondary: "{{ now().strftime('%a %-d %b') }}"
                icon: mdi:clock-outline
                primary_info: state
                multiline_secondary: true
                fill_container: true
              - type: custom:mushroom-template-card
                primary: >-
                  Today {{ state_attr('weather.forecast_home', 'temperature') }}°
                  {{ states('weather.forecast_home') }}
                secondary: >-
                  {% set ev = state_attr('calendar.stafford_borough_council', 'message') %}
                  {% set st = state_attr('calendar.stafford_borough_council', 'start_time') %}
                  {% if ev and st %}
                    Next bin: {{ ev }} ({{ as_timestamp(st) | timestamp_custom('%a %-d %b') }})
                  {% endif %}
                icon: mdi:weather-partly-cloudy
              - type: custom:mushroom-chips-card
                alignment: start
                chips:
                  - type: entity
                    entity: person.dan
                    icon_color: blue
                  - type: entity
                    entity: person.nicola
                    icon_color: pink
                  - type: template
                    icon: mdi:account-school
                    icon_color: orange
                    content: "Will {{ states('input_text.placeholder_william_status') }}"
                  - type: entity
                    entity: binary_sensor.doorbell_camera_motion
                    icon: mdi:doorbell

          # RIGHT COLUMN — control
          - type: vertical-stack
            cards:
              - type: grid
                columns: 2
                square: false
                cards:
                  - type: custom:decluttering-card
                    template: tile_room_lights
                    variables: { area_id: lounge,   name: Lounge,   icon: mdi:sofa,           scene_id: scene.room_lounge_default }
                  - type: custom:decluttering-card
                    template: tile_room_lights
                    variables: { area_id: kitchen,  name: Kitchen,  icon: mdi:countertop,     scene_id: scene.room_kitchen_default }
                  - type: custom:decluttering-card
                    template: tile_room_lights
                    variables: { area_id: library,  name: Library,  icon: mdi:bookshelf,      scene_id: scene.room_library_default }
                  - type: custom:decluttering-card
                    template: tile_room_lights
                    variables: { area_id: dining,   name: Dining,   icon: mdi:chair-rolling,  scene_id: scene.room_dining_default }
                  - type: custom:decluttering-card
                    template: tile_room_lights
                    variables: { area_id: hall,     name: Hall,     icon: mdi:door,           scene_id: scene.room_hall_default }
                  - type: custom:decluttering-card
                    template: tile_room_lights
                    variables: { area_id: landing,  name: Landing,  icon: mdi:stairs,         scene_id: scene.room_landing_default }
              - type: custom:decluttering-card
                template: tile_heating
              - type: custom:decluttering-card
                template: tile_macros
```

- [ ] **Step 2: Reload + verify in landscape**

Load `/kiosk/home` in a desktop browser at landscape proportions (e.g. 1280×800). Verify the 2-column layout, no header, no sidebar.

- [ ] **Step 3: Commit**

```bash
cd /config && git add dashboards/kiosk.yaml && git commit -m "feat: Kiosk dashboard (landscape, panel mode, no header/sidebar)"
```

### Task 27: Configure Kiosk Fire-tablet behaviours via browser_mod

**Files:** none (configured per-browser via browser_mod settings)

- [ ] **Step 1: Register the Fire tablet's browser**

On the Fire tablet (or its emulator), open `/kiosk/home` while logged in as `kiosk`. Then in **Settings → Browser Mod → Devices**, name this browser `kiosk_wall_tablet`.

- [ ] **Step 2: Add a night-dim automation**

Append to `/config/automations.yaml`:

```yaml
- id: kiosk_night_dim
  alias: "Kiosk: dim screen 22:30–06:30"
  trigger:
    - platform: time
      at: "22:30:00"
  action:
    - service: browser_mod.command
      data:
        command: brightness
        brightness: 5
        browser_id: kiosk_wall_tablet

- id: kiosk_morning_brighten
  alias: "Kiosk: brighten screen at 06:30"
  trigger:
    - platform: time
      at: "06:30:00"
  action:
    - service: browser_mod.command
      data:
        command: brightness
        brightness: 100
        browser_id: kiosk_wall_tablet
```

- [ ] **Step 3: Reload automations + smoke test**

Reload automations. Trigger each manually (Run actions). Confirm Fire tablet brightness changes.

- [ ] **Step 4: Commit**

```bash
cd /config && git add automations.yaml && git commit -m "feat: kiosk night-dim + morning-brighten via browser_mod"
```

> NOTE: motion-wake on Fire tablet requires the Fully Kiosk Browser (separate Android app) — its motion-detection feature integrates with HA via the Fully Kiosk integration. Configuring Fully Kiosk is a Track-1.5 follow-on; for v1, fixed-schedule dim is enough.

---

## Phase 5 — Apple Watch + verification

### Task 28: Configure Dan's Apple Watch Actions

**Files:** none (HA Companion app on Dan's iPhone)

- [ ] **Step 1: Open Companion → Settings → Watch → Watch Actions**

Add five actions in this order:

| Slot | Name | Icon | Service |
|---|---|---|---|
| 1 | Goodnight | weather-night | `script.goodnight` |
| 2 | Movie | movie-roll | `script.movie` |
| 3 | Fire | fire | `climate.toggle` for `climate.fire` |
| 4 | EV preheat | car-electric | `volvo.start_climate` (or whichever Volvo integration service exists; if absent, leave a placeholder action calling `notify` with "EV preheat — TBD") |
| 5 | Driveway | spotlight | `script.driveway_lights_on` |

- [ ] **Step 2: Open the Watch app on iPhone**

Verify the 5 actions appear in Apple Watch's HA Watch app. Tap each → confirm corresponding side effect in HA.

### Task 29: Configure Nicola's Apple Watch Actions

**Files:** none (HA Companion app on Nicola's iPhone)

- [ ] **Step 1: Open Companion → Settings → Watch → Watch Actions**

| Slot | Name | Icon | Service |
|---|---|---|---|
| 1 | Goodnight | weather-night | `script.goodnight` |
| 2 | Morning | weather-sunny | `script.morning` |
| 3 | Movie | movie-roll | `script.movie` |
| 4 | Lounge lights | sofa | `scene.turn_on` for `scene.room_lounge_default` |
| 5 | Front door | door | placeholder action — `notify` with "Front door — coming with smart-lock hardware" (replace with `lock.unlock` once hardware lands) |

- [ ] **Step 2: Verify on Watch**

Tap each, confirm side effect.

### Task 30: Verify all acceptance criteria from spec §14

**Files:** none (manual verification)

- [ ] **Step 1: Console-error pass on all four dashboards**

Open browser DevTools (F12) → Console. Load each of `/nicola/home`, `/dan/home`, `/william/home`, `/kiosk/home`. Each should load with **zero errors**. (Warnings are OK.)

- [ ] **Step 2: Auto-default-dashboard test**

Log in as Dan (already done) → land on `/dan/home`. Log in as Nicola (private window) → `/nicola/home`. As `kiosk` → `/kiosk/home`.

- [ ] **Step 3: Tile + macro full pass**

Through Nicola's Home view: tap each tile, fire each macro. Confirm side effects for each (lights / fire / scripts).

- [ ] **Step 4: Watch Actions pass**

Tap each Watch Action on Dan's and Nicola's Watches. Confirm side effects.

- [ ] **Step 5: Theme renders in light + dark**

On iPhone, toggle iOS appearance light/dark. Confirm the dashboard auto-follows (theme accent shifts from `#4a90c2` to `#7ab8e0`).

- [ ] **Step 6: William swap-in path documented in repo**

In Studio Code, open `/config/dashboards/william.yaml` and confirm the comment block at the top describes the post-onboarding swap (or add one if missing):

```yaml
# When William is onboarded:
# 1. Add him as a Person entity (Settings → People → Add Person → William)
# 2. Tag his Pixel 8a + iPad as his devices for tracking
# 3. Replace this placeholder with the post-onboarding design (see spec §6)
# 4. Update the chips_status include's "Will" chip from input_text.placeholder_william_status to person.william
```

- [ ] **Step 7: Final commit**

```bash
cd /config && git add -A && git commit -m "feat: family dashboards complete (Track 1)" --allow-empty
```

If the commit is empty (no pending changes), that's fine — it's a tag-point.

- [ ] **Step 8: Tag the milestone**

```bash
cd /config && git tag -a track1-family-dashboards-v1 -m "Track 1 — family dashboards live"
```

---

## Closing notes

**Track 1 is complete when all 30 tasks are checked.**

Naturally next:
- **Alexa ↔ HA TODO** (its own item in `~/.claude/TODO.md`) — unblocks the Morning macro's Kitchen Alexa step
- **Track 2** (MCP / AI control surface) — now has a stable user-facing surface to design over
- **Track 1.5 follow-ons** — Dan's Energy/Volvo/Media/System tab content, Fully Kiosk + motion-wake on the Fire tablet, William's full dashboard once he's onboarded
