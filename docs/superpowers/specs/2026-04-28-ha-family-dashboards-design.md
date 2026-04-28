# Home Assistant family dashboards — design spec

**Status:** Draft, brainstormed 2026-04-28
**Track:** 1 of 4 in the broader "Home Assistant overhaul" initiative (see `~/.claude/TODO.md` → "Home Assistant overhaul")
**Owner:** Dan Robinson
**Authoring session:** 2026-04-28

## 1. Context & goals

The household runs a mature Home Assistant OS install (KVM VM on Unraid svr001 at `192.168.1.151:8123`, ~662 entities, 47 automations, zigbee2mqtt, Octopus Energy, Volvo EX40, Reolink cameras, Wiz/Nanoleaf/Sonoff lights, Sonos, Mealie, Stafford BC waste calendar, fireplace + office aircon).

The existing dashboards are not designed for the family — they expose every entity to every user. Track 1 is to design **purpose-built dashboards per persona** so each family member opens HA and immediately sees what's relevant to them, no more, no less. The Mushroom HACS card library and per-user dashboards are the chosen vehicles.

Tracks 2 (MCP / AI control surface), 3 (HACS integrations & automations engines), and 4 (Hardware roadmap) are *deliberately blocked* on this Track 1 design — many of their decisions are driven by what users actually touch in the dashboards, so this design must land first.

## 2. Personas

| Persona | Devices | Comfort | Primary need |
|---|---|---|---|
| **Dan** (you) | iPhone, Apple Watch, Mac (browser) | Power user — comfortable with HA UI, will edit YAML | Info-dense control surface, system health, EV, energy detail |
| **Nicola** | iPhone, Apple Watch | Light user — wants glance + one-tap, not configuration | "Home" landing with glance info up top + control rows below |
| **William** (10, T1D, AAPS closed-loop) | Pixel 8a, iPad | Not yet onboarded to HA. Has dedicated OpenClaw `william-agent` (WhatsApp + Nightscout). | Age-appropriate view (post-onboarding); placeholder slot today |
| **Kiosk** (future) | Fire tablet, wall-mounted | Anyone walking past. No login. | Always-on, big-tap, no swipe required |

## 3. Architecture — Approach B (per-user dashboards)

Four dashboards, each its own URL: `dashboard-nicola`, `dashboard-dan`, `dashboard-william`, `dashboard-kiosk`. HA's "Set as default dashboard" routes each user to theirs on login. All four share theme, scenes, scripts, macros, and entity model — only the *cards/views* differ.

Two alternatives were considered and rejected:

- **A: Single dashboard with view-level visibility filters** — rejected because the Fire tablet kiosk signs in as one HA user; visibility filters can't say "show on the wall, hide on Nicola's phone" without breaking phone usage.
- **C: Single dashboard with browser_mod-driven auto-redirect per user** — rejected for fragility and the same kiosk-collision problem.

**Authoring mode: hybrid YAML.** The legacy default dashboard stays in `storage` mode (so existing setup is undisturbed). Our four new dashboards are registered with `mode: yaml` individually under `lovelace.dashboards.*` — they become git-tracked files we can template across, while the default dashboard remains UI-editable as a "tinker" surface for Nicola.

**Escape hatch for Nicola:** a fifth dashboard called `personal` left in HA's default UI/storage mode. Nicola (or Dan) can experiment with custom cards there without touching the YAML-managed ones.

### File layout in `/config/` (HA root):

```
configuration.yaml              # registers lovelace.mode: yaml + 4 dashboards + theme dir
themes/
  └── mushroom_robbohome.yaml   # customised Mushroom-compat theme
dashboards/
  ├── nicola.yaml               # Nicola's "Home" — the canonical primary
  ├── dan.yaml                  # Dan's power-user view
  ├── william.yaml              # William placeholder → real dashboard once onboarded
  └── kiosk.yaml                # Fire tablet wall view
lovelace_includes/              # reusable card snippets (decluttering-card)
  ├── tile_energy.yaml
  ├── tile_weather_bins.yaml
  ├── tile_laundry.yaml
  ├── tile_room_lights.yaml
  ├── tile_heating.yaml
  ├── tile_macros.yaml
  └── chips_status.yaml
scripts.yaml                    # script.goodnight, script.morning, script.movie
scenes.yaml                     # scene.* per-room + macro-target scenes
```

### Shared layer — single source of truth

| Building block | Path | Why |
|---|---|---|
| Theme | `themes/mushroom_robbohome.yaml` | One theme, four front-doors |
| Card snippets | `lovelace_includes/*.yaml` | Each tile defined once via `decluttering-card`, instantiated per dashboard with parameters |
| Scripts | `scripts.yaml` | All dashboards trigger the same `script.goodnight` / `script.morning` / `script.movie` |
| Scenes | `scenes.yaml` | Per-room and per-macro scenes referenced by all dashboards |
| Areas registry | HA core | Every controllable entity tagged with its room. Tiles iterate areas, not hard-coded entity IDs. |

## 4. Nicola's "Home" dashboard (canonical primary)

Designed for **iPhone portrait** as the primary form factor.

### View tabs (used across all four dashboards):

- 🏠 **Home** (default — the layout below)
- 🏘 **All Rooms** (Master · William · Guest · Office + a fuller version of the 6 primary rooms)
- 🌳 **Outside** (cameras: chickens · driveway · side garden · side path · doorbell)
- ⋯ **More** (EV · Sonos · Family/people · Energy detail · Settings)

### Home-view layout (top to bottom):

```
┌─────────────────────────────────────────┐
│ CHIPS BAR  (mushroom-chips-card)        │
│ [Dan🏠] [Nic🏠] [Will–]                  │
│ [☀17°] [🚪 ok] [🔥 off]                  │
├─────────────────────────────────────────┤
│ ENERGY NOW  (template + mini-graph)     │
│ 27.3 p/kWh · Today 8.2 kWh ↓ vs avg     │
│ ⚡ Import rising  (only when active)     │
├─────────────────────────────────────────┤
│ WEATHER + BINS  (template-card)         │
│ Today 17° ☀  Tomorrow 14° ⛅            │
│ Next bin: General · Mon 4 May           │
├─────────────────────────────────────────┤
│ LAUNDRY  (template-card)                │
│ Washer · Idle      Dryer · 23 min left  │
├─────────────────────────────────────────┤
│ ROOMS  (3×2 grid of mushroom-light)     │
│ [Lounge]   [Kitchen]   [Library]        │
│ [Dining]   [Hall  ]    [Landing]        │
│  Tile: room name, light count, on/off,  │
│  brightness ring (room avg).            │
│  Tap → opens room view in 'All Rooms'.  │
│  Long-press → toggles room scene.       │
├─────────────────────────────────────────┤
│ HEATING  (mushroom-climate-card)        │
│ 🔥 Fire · Off                           │
├─────────────────────────────────────────┤
│ MACROS  (mushroom-template ×3)          │
│ [🌙 Goodnight] [☀ Morning] [🎬 Movie]   │
└─────────────────────────────────────────┘
```

### Tile mechanics

| Tile | Card type | Source entities | Behaviour |
|---|---|---|---|
| Chips bar | `mushroom-chips-card` (entity / template chips) | `person.dan`, `person.nicola`, `input_text.placeholder_william_status` (template), `weather.forecast_home`, `binary_sensor.doorbell_camera_motion`, `climate.fire` | Tap chip → entity "More info" sheet |
| Energy NOW | `mushroom-template-card` + `mini-graph-card` | `sensor.electricity_cost_rate`, `sensor.electricity_today`, `binary_sensor.import_rising_fast` | Tap → "More" tab Energy detail |
| Weather + Bins | `mushroom-template-card` | `weather.forecast_home`, `calendar.stafford_borough_council` (next event) | Tap weather → forecast popup; tap bin → calendar |
| Laundry | `mushroom-template-card` | `binary_sensor.washing_machine_running`, `binary_sensor.tumble_dryer_running`, `input_select.laundry_status` | State changes when running; tap → laundry detail page |
| Rooms 3×2 | `mushroom-light-card` × 6, area-driven | `light.*` filtered by HA Area | Tap = scene; long-press = full controls; ring = avg brightness |
| Heating | `mushroom-climate-card` | `climate.fire` (only `climate.*` outside office today) | Tap = toggle |
| Macros | `mushroom-template-card` × 3 | `script.goodnight`, `script.morning`, `script.movie` | Single tap fires script + haptic |

### William placeholder strategy

Until `person.william` exists, the "Will–" chip uses `input_text.placeholder_william_status` showing `–`. When William's onboarded:

1. Add him as an HA Person entity (William iPad → device tracker; later Pixel 8a Companion app)
2. Re-point the chip in `lovelace_includes/chips_status.yaml` from `input_text.placeholder_william_status` → `person.william`
3. Done — the shared chip definition is the only edit; all four dashboards update.

## 5. Dan's "Power" dashboard

Same shared theme/scripts/scenes/macros, denser cards, more tabs:

- 🏠 **Home** — same 7-block stack as Nicola's, plus inserted chips: **EV** (`sensor.volvo_ex40_charge_pct`, `sensor.volvo_ex40_range`) and **cameras** (last motion thumbnail). Heating tile expands to include `climate.office_aircon`.
- 🏘 **All Rooms** — Master · William · Guest · Office + the primary 6 with deeper controls (per-bulb selectors, color, transitions)
- 🌳 **Outside** — live camera grid, driveway lights script buttons, doorbell siren toggle
- ⚡ **Energy** — `mini-graph-card` for live import/export, Octopus rate timeline, Greener Nights + saving sessions calendar, EV charging schedule, import-rising history
- 🚗 **Volvo** — EX40 detail: charge, range, climate preheat, lock, location
- 🎵 **Media** — Sonos rooms, Office speaker, media browser
- 🛠 **System** — HA core/supervisor/add-on update tiles, automation health, Unraid disk/CPU (via `unraid` MCP), New Relic alert chips

The contents of Energy / Volvo / Media / System tabs are sketched at the tab-level only in this spec — their full card-level content is a Track-1.5 follow-on (not blocking anything).

## 6. William's dashboard

**Pre-onboarding (today):** single view with a friendly mushroom-template-card "William's dashboard — coming soon when his Pixel + iPad are added to HA". One static link to the OpenClaw `william-agent` WhatsApp.

**Post-onboarding** (deferred until William has HA entities — exact card design re-validated then):

- 🏠 **Home** — big clock + today's date/weather, "who's home" chips, his own **room lights** (scenes only — no per-bulb fiddling), bin-day reminder, shortcut to william-agent
- 📚 **School** — calendar for school events, today's lessons, homework reminder
- 🤖 **Agent** — opens william-agent in WhatsApp via deep link

**Hard exclusions** (deliberately *not* on William's dashboard): no cameras, no door lock, no heating control, no admin/automation editing, no other people's rooms, **no Nightscout BG tile by default** — that lives only on Dan's phone unless Dan explicitly opts in.

## 7. Kiosk dashboard (Fire tablet, wall-mounted, future)

Single view, **landscape layout**, always-on, no tabs.

```
┌────────────────────────────┬─────────────────────────────┐
│  TIME · DATE  (huge)       │  ROOMS  (2×3 grid, big)     │
│  19:42  Tue 28 Apr         │  [Lounge ] [Kitchen]        │
│                            │  [Library] [Dining ]        │
│  ☀ 17° → ⛅ 14° tomorrow   │  [Hall   ] [Landing]        │
│  Next bin: Mon 4 May       ├─────────────────────────────┤
│                            │  HEATING                    │
│  🏠 Dan home · Nic home    │  🔥 Fire — Off              │
│  Will– (placeholder)       │                             │
│                            ├─────────────────────────────┤
│  🚪 Doorbell quiet         │  MACROS  (3 huge buttons)   │
│  📷 Cameras OK             │  [🌙 Goodnight]             │
│                            │  [☀ Morning  ]              │
│                            │  [🎬 Movie   ]              │
└────────────────────────────┴─────────────────────────────┘
```

### Kiosk-specific behaviours (browser_mod-driven)

- Dim screen 22:30–06:30 to 5% brightness
- Wake on motion via Fire tablet's built-in sensor → 100% brightness for 60s
- Auto-refresh page every 10 min (recovers from connectivity blips)
- Disable HA sidebar + header (`kiosk-mode` HACS plugin)
- Logged in as a dedicated `kiosk` HA user with limited permissions (can run scripts, can't edit dashboards)

## 8. Apple Watch surface (Dan + Nicola)

Configured in HA Companion app → Watch settings.

### Watch Actions per user

| | Dan | Nicola |
|---|---|---|
| 1 | 🌙 Goodnight | 🌙 Goodnight |
| 2 | 🎬 Movie | ☀ Morning |
| 3 | 🔥 Fire toggle | 🎬 Movie |
| 4 | 🚗 EV preheat (Volvo) | 💡 Lounge lights toggle |
| 5 | 💡 Driveway lights | 🚪 Front door (when smart lock lands — placeholder action) |

### Complications (one slot each, optional)

- Dan: small complication showing `binary_sensor.import_rising_fast` (energy spike awareness)
- Nicola: small complication showing `climate.fire` state (on/off)

## 9. Theme — `mushroom_robbohome.yaml`

Built on the **Mushroom Themes** bundle (ships with `piitaya/lovelace-mushroom`):

```yaml
mushroom_robbohome:
  modes:
    light:
      primary-color: '#4a90c2'      # muted sky blue accent
      ha-card-border-radius: '20px' # bigger than mushroom default for Apple-Home feel
    dark:
      primary-color: '#7ab8e0'      # lighter accent for dark mode
      ha-card-border-radius: '20px'
  card-mod-theme: mushroom_robbohome  # enables card-mod CSS overrides
  ha-card-box-shadow: none            # flat cards, no shadow
  paper-font-body1_-_font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', sans-serif
```

Auto light/dark follows iOS system mode (HA Companion supports this natively).

## 10. Macros (full bodies in `scripts.yaml`)

### `script.goodnight`

Manual via macro tile, or auto at **23:00** via `automation.goodnight_late` if not already run.

1. Turn off all `light.*` in Lounge, Library, Kitchen, Dining, Office
2. Hall + Landing dim to 15%
3. `climate.fire` → off
4. `input_boolean.doorbell_quiet_mode` → on
5. Pause all Sonos rooms
6. Notify Dan's iPhone: *"Goodnight done — {n} lights off, fire off, doorbell quiet."*
7. *(Future: arm cameras + lock doors when hardware lands — currently no-ops)*

### `script.morning`

Manual via macro tile, or auto-fires at **06:30 Mon–Fri** via `automation.morning_weekday`.

1. Activate `scene.kitchen_morning_warm` + `scene.lounge_morning_warm`
2. Tell Kitchen Alexa (`media_player.kitchen_echo`, dependent on Alexa↔HA TODO landing) → play BBC Radio 4, volume 0.3, for 20 min
3. If today's `calendar.stafford_borough_council` next event = today → push notification *"Bin day: {bin_type}"*
4. `input_boolean.doorbell_quiet_mode` → off
5. Notify both phones: *"Morning"*

Until the Alexa TODO lands, step 2 is a no-op (logs a warning); rest of macro works.

### `script.movie`

Manual only.

1. Activate `scene.lounge_movie` (low warm)
2. Sonos lounge → pause
3. `input_boolean.doorbell_quiet_mode` → on for 2h via `timer.movie_doorbell_quiet`
4. Notify Dan: *"Movie mode — 2h"*

All three scripts are idempotent and fail-soft (a missing entity logs a warning but doesn't halt the rest of the macro).

## 11. HACS dependencies

| Type | Name | Why |
|---|---|---|
| Frontend | `lovelace-mushroom` (piitaya) | The card library |
| Frontend | `lovelace-card-mod` (thomasloven) | CSS overrides for theme + custom border radius |
| Frontend | `decluttering-card` (RomRider) | Define each tile once, reuse across 4 dashboards |
| Frontend | `mini-graph-card` (kalkih) | Sparkline in the Energy NOW tile |
| Frontend | `kiosk-mode` (NemesisRE) | Hide sidebar/header on the Kiosk dashboard |
| Integration | `browser_mod` (thomasloven) | Kiosk auto-refresh, night-dim, motion-wake |

Optional later (deferred to Track 3): `apexcharts-card` if Dan's Energy tab needs richer graphs than `mini-graph-card`.

## 12. Prerequisites checklist

Before implementation can start:

1. **HACS installed** — verify `/config/custom_components/hacs` exists on the HA VM. If not, install via standard HACS bootstrap.
2. **Areas tagged on every controllable entity.** Audit the registry: each light/switch/scene must have an Area assigned. Areas needed: Lounge, Library, Kitchen, Dining, Hall, Landing, Master Bed, William, Guest, Office, Driveway, Chickens, Side Garden, Side Path, Doorstep.
3. **Hybrid Lovelace mode** — default stays `storage`; four new dashboards registered with `mode: yaml`. In `configuration.yaml`:

   ```yaml
   lovelace:
     mode: storage   # default dashboard stays UI-editable; our 4 are yaml-mode below
     dashboards:
       nicola:  { mode: yaml, filename: dashboards/nicola.yaml,  title: Home,  icon: mdi:home,        show_in_sidebar: true,  require_admin: false }
       dan:     { mode: yaml, filename: dashboards/dan.yaml,     title: Power, icon: mdi:tools,       show_in_sidebar: true,  require_admin: false }
       william: { mode: yaml, filename: dashboards/william.yaml, title: Will,  icon: mdi:account-tie, show_in_sidebar: true,  require_admin: false }
       kiosk:   { mode: yaml, filename: dashboards/kiosk.yaml,   title: Kiosk, icon: mdi:tablet,      show_in_sidebar: false, require_admin: false }
   frontend:
     themes: !include_dir_merge_named themes/
   ```

4. **Helper entities to create:**
   - `input_boolean.doorbell_quiet_mode`
   - `timer.movie_doorbell_quiet` (duration 2h)
   - `input_text.placeholder_william_status` (default value `–`)

5. **Scenes to define** in `scenes.yaml`: `scene.kitchen_morning_warm`, `scene.lounge_morning_warm`, `scene.lounge_movie`, plus per-room scenes the 6 primary tiles invoke on long-press.

6. **Scripts to define** in `scripts.yaml`: `script.goodnight`, `script.morning`, `script.movie` (full bodies in §10).

7. **Automations to define** in `automations.yaml` (each is a thin wrapper that calls one of the scripts):
   - `automation.goodnight_late` — trigger at 23:00 daily, condition: `script.goodnight` not run in last 2h, action: `script.goodnight`
   - `automation.morning_weekday` — trigger at 06:30 Mon–Fri, action: `script.morning`

8. **Per-user default dashboard** — Profile → Default dashboard for each HA user (Dan → `dan`, Nicola → `nicola`, kiosk-user → `kiosk`).

9. **A dedicated `kiosk` HA user** with no admin rights, used to log in the Fire tablet.

10. **HA Companion app installed** on Dan's iPhone, Nicola's iPhone, and (later) William's Pixel 8a + iPad.

## 13. Out of scope

- **Alexa ↔ HA integration** — its own TODO. Until landed, Morning macro's Kitchen Alexa step is a logged no-op.
- **Smart-lock + camera-arming** in Goodnight — no-ops in the script today; trivially uncommented when hardware lands (Track 4).
- **Track 2** (MCP / AI control surface design) — separate spec; Track 1 makes dashboards usable by humans, Track 2 makes them usable by agents.
- **Track 3** (HACS integrations beyond dashboard ones — NodeRED, Whisper voice, automation engines).
- **Track 4** (Hardware roadmap — TRVs, smart locks, cameras, Wyoming voice satellites, Matter).
- **William's full dashboard content** — placeholder only in Track 1; full design lands when he's onboarded to HA.
- **Per-room Sonos detail / EV deep page / System tab on Dan's dashboard** — sketched at the tab-level only; full card content is a Track-1.5 follow-on.
- **Nicola's "personal" UI-mode tinker dashboard** — created empty as the escape hatch; populated by her over time.
- **Energy dashboard data sources beyond what already exists** — solar inverter, individual circuit clamps — Track 4.

## 14. Acceptance criteria

Track 1 is "done" when:

- [ ] All 4 dashboards load without console errors on Dan's iPhone, Nicola's iPhone, and a Fire tablet (or emulator).
- [ ] Each user lands on their dashboard automatically when opening the HA Companion app.
- [ ] Each of the 6 primary tiles on Nicola's Home view renders correctly with real data.
- [ ] All 6 primary room tiles control their lights (tap = scene, long-press = full controls).
- [ ] All 3 macros fire end-to-end (Morning's Alexa step is allowed to be a no-op until the Alexa TODO lands).
- [ ] Watch Actions appear on Dan's and Nicola's Watches and trigger correctly.
- [ ] Kiosk dashboard runs on the Fire tablet, dims overnight, wakes on motion.
- [ ] Theme renders correctly in both light and dark mode following iOS system setting.
- [ ] William placeholder slot is in place and the swap-in path is documented in the spec.

## 15. Open questions / future work

- **Smart lock choice** (Track 4) — to enable Goodnight's lock step. Options: Aqara U200, Yale Linus, Aqara A100 Zigbee. TBD when hardware roadmap is brainstormed.
- **TRV choice** (Track 4) — to make the Heating tile actually heat the house and not just toggle the fireplace. Aqara, Sonoff, Tado all candidates.
- **Wyoming voice satellites** (Track 3 + 4) — local voice in addition to / replacing Alexa. Decision deferred until Track 3.
- **Nightscout integration as an HA sensor** — currently only via OpenClaw william-agent. If we want a BG tile on Dan's phone (and only Dan's phone) we'll add the `nightscout` HA integration. Track 1.5 follow-on if Dan wants this.
- **Energy dashboard polish** on Dan's view — sparklines exist; full Track 1.5 spec for the Energy tab content.
