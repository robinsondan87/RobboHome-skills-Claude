# Home Assistant Track 3 — Reliability, Voice, Presence

**Status:** ✅ DONE end-to-end on 2026-04-29 (same-day brainstorm + implementation). All three sub-tracks shipped and verified.
**Track:** 3 of 4 in the broader "Home Assistant overhaul" initiative (see `~/.claude/TODO.md`)
**Owner:** Dan Robinson
**Authoring session:** 2026-04-29

## 1. Context & goals

Track 1 (family dashboards) shipped 2026-04-28 with tag `track1-family-dashboards-v1`. Track 1.5 polish landed on top (Living Spaces popups, Dan's tabs, light groups, Nightscout BG, William onboarded with `person.william`).

Track 3 was scoped as "HACS / software gaps" and originally listed seven sub-areas (automation engines, voice, remote access, backups, presence, notifiers, config-as-code). Brainstorming on 2026-04-29 reduced this to **three concrete sub-tracks** plus a set of explicit "no design needed" decisions documented in §11.

Goals, in priority order:

1. **Reliability** — eliminate the single-point-of-failure of HA backups living only on the HA partition. Auto-track config changes in git so any future tinkering has a reversible history.
2. **Voice control via existing Alexa devices** — let the family say "Alexa, turn off the lounge lights" / "Alexa, goodnight" without paying for Nabu Casa or installing a local voice stack.
3. **Presence pilot** — deploy the single Aqara FP2 the family already owns (in Dan's office) and prove the value of presence-driven automations before buying more hardware (Track 4).

## 2. Decisions made during brainstorming

| Decision | Choice | Why |
|---|---|---|
| Remote access + Alexa + cloud backup | **Nabu Casa subscription (~£5/mo)** | One toggle covers polished Companion app remote access, official Alexa Smart Home Skill, and cloud backup. Light touch over three otherwise-DIY problems. |
| Local voice stack | **Skip** (no Wyoming/Whisper/Piper) | Family lives on Alexa already; revisit only if Alexa proves limiting |
| Automation engine | **Stay native HA YAML** | 50 automations + 16 scripts work fine; no complexity problem |
| Notifier | **Mobile push + Alexa announce only** | No need for SMS/email/chat; OpenClaw agents handle their own |
| Backup strategy | **Three layers: local + Unraid syncthing chain + Nabu Casa Cloud Backup** | Defence in depth; Nabu Casa Cloud is the off-LAN copy (no separate Google Drive add-on needed) |
| Config-as-code | **Auto-commit + push to private GitHub** | Full edit history; `/config/.git` already initialised |
| Presence hardware | **Single FP2 pilot in Dan's office** | Validate value before scaling |
| Tailscale | **Keep** | Still useful for the rest of the fleet (svr001/002/003, vmi3091030); just not the primary HA remote-access path anymore |

## 3. Sub-track decomposition

Three independent specs, each with its own acceptance criteria. Order: 3a → 3b → 3c (each is independent; nothing strictly blocks the others, but doing 3a first means later changes are git-tracked from day one).

| Sub-track | Scope summary |
|---|---|
| **3a — Reliability** | HA backups → Unraid network share + Nabu Casa Cloud Backup. `/config/.git` auto-commit + push to GitHub. |
| **3b — Voice (Alexa) + Remote access** | Nabu Casa subscription enabled. Alexa Smart Home Skill linked → Echos can control lights/scenes/scripts/climate/sensors. HA Companion app remote access via Cloud. Morning macro Kitchen Echo step fixed to use `notify.kitchen_echo_show_announce`. |
| **3c — Presence pilot** | Deploy FP2 in Dan's office. Three starter presence automations (office on, office off, kitchen night-light via Echo motion). |

## 4. Sub-track 3a — Reliability

### 4.1 Backup architecture

```
        HA daily auto-backup
                │
        ┌───────┼───────────────┐
        ▼       ▼               ▼
  /backup/   /share/        Nabu Casa
  (HAOS)     unraid         Cloud Backup
             backup         (off-LAN copy)
              │
              ▼
        syncthing on Unraid
              │
              ▼
            svr003
```

Three geographic copies of every backup: HAOS partition, Unraid + svr003 (LAN-twinned), Nabu Casa Cloud (off-LAN). The user's existing syncthing chain on Unraid is reused — no new sync infrastructure needed. Nabu Casa Cloud Backup is included with the subscription, no extra add-on.

### 4.2 Components

1. **HAOS network storage mount** — *Settings → System → Storage → Add network storage* with a CIFS/SMB share to Unraid (e.g. `\\192.168.1.200\backup_ha`). HAOS automatically exposes mounted shares as backup *agents*.
2. **Backup agent configuration** — daily auto-backup writes to all three agents: `local`, the new network share, and Nabu Casa Cloud. Retention: 7 local, 14 network share, default cloud retention (Nabu Casa-managed).
3. **Nabu Casa subscription** — *Settings → Home Assistant Cloud → Sign up / Sign in*. Cloud Backup is enabled per dashboard once the subscription is active. No add-on installation needed.
4. **Syncthing** on Unraid (already running) carries `/mnt/user/backup_ha/` to svr003. No action needed.

### 4.3 Config-as-code: auto-commit + push

```
   /config/.git ──── 15-min cron ──── commit + push
   (initialised        (in SSH                  │
    in Track 1)         add-on)                 ▼
                                       github.com/robinsondan87/
                                         robbohome-ha-config
                                         (private)
```

### 4.4 Components

1. **Auto-commit script** at `/share/auto_commit.sh`:

   ```bash
   #!/bin/bash
   cd /config
   sudo git add -A
   sudo git diff --staged --quiet || sudo git commit -m "auto-commit $(date +%F\ %T)"
   sudo git push origin master 2>/dev/null
   ```

2. **Cron entry** added to the `Advanced SSH & Web Terminal` add-on's `init_commands` — runs `/share/auto_commit.sh` every 15 minutes:

   ```yaml
   init_commands:
     - "echo '*/15 * * * * /share/auto_commit.sh' >> /etc/crontabs/root"
     - "crond -l 8"
   ```

3. **GitHub private repo** — `robinsondan87/robbohome-ha-config`, created via `gh repo create --private`.
4. **Deploy key auth** — generate ed25519 keypair inside HAOS at `/root/.ssh/id_ed25519_github`; pub key added to the GitHub repo as a deploy key with write access. Private key path saved in SOPS as `HA_GIT_DEPLOY_KEY_PATH` for portability.
5. **`.gitignore` audit** — confirm exclusion of: `secrets.yaml`, `.storage/`, `home-assistant.db`, `home-assistant_v2.db`, `home-assistant.log*`, `__pycache__/`, `.cloud/`. Most already covered from Track 1; final sweep needed.

Manual commits still work; auto-commit catches what humans forget.

### 4.5 3a Acceptance criteria

- [ ] HA daily auto-backup writes to all three agents: `local`, Unraid network share, Nabu Casa Cloud (verify in HA backup logs / Cloud dashboard)
- [ ] Nabu Casa subscription is active and Cloud Backup is enabled
- [ ] Syncthing on Unraid carries new HA backups to svr003 (verify by listing svr003's backup dir)
- [ ] `/share/auto_commit.sh` runs every 15 min via cron, commits any pending `/config/` changes, pushes to GitHub
- [ ] `github.com/robinsondan87/robbohome-ha-config` private repo exists and shows recent commits
- [ ] `.gitignore` excludes secrets, `.storage/`, db files, logs

## 5. Sub-track 3b — Voice (Alexa via Nabu Casa) + Remote access

### 5.1 Configuration

Nabu Casa subscription enables three things at once: HA Cloud (remote Companion app), Cloud Backup (§4), and the official **Alexa Smart Home Skill**. No `emulated_hue` needed; full support for lights, scenes, scripts, climate, and sensor reads via voice.

**Setup:**

1. **Subscribe** — *Settings → Home Assistant Cloud → Start your free trial / Sign in* (1-month free trial, then ~£5/mo).
2. **Enable Alexa** — within Cloud settings, toggle on **Alexa**. HA Cloud generates a per-instance OAuth endpoint.
3. **Link the skill** — in the Alexa app on Dan's phone, *Skills & Games → Search "Home Assistant Smart Home" → Enable to use*. Sign in with the Nabu Casa account, accept HA permissions.
4. **Discover devices** — Alexa app *Devices → Add Device → Other → Discover*. All exposed HA entities appear.

### 5.2 Entity exposure list

Cloud's "Alexa entities" UI (Settings → HA Cloud → Alexa → Manage entities) is per-entity opt-in. v1 list:

| Entity | Spoken-name override |
|---|---|
| `light.lounge_group` | "Lounge lights" |
| `light.kitchen_group` | "Kitchen lights" |
| `light.library_group` | "Library lights" |
| `light.dining_group` | "Dining lights" |
| `light.hall_group` | "Hall lights" |
| `light.landing_group` | "Landing lights" |
| `light.office_front_1` (or `light.office_group` if created) | "Office lights" |
| `script.goodnight` | "Goodnight" |
| `script.morning` | "Morning" |
| `script.movie` | "Movie" |
| `scene.lounge_movie` | "Movie scene" |
| `climate.fire` | "Fire" |
| `climate.office_aircon` | "Office air conditioning" |
| `sensor.blood_sugar` | "William's blood sugar" |

Climate and sensor reads are now possible — "Alexa, fire on", "Alexa, what's William's blood sugar", "Alexa, set office to 21".

### 5.2 Morning macro Kitchen Echo fix (resolved during Track 3 implementation)

**Final pattern (BBC Radio 1, 30 min):** scripts expose to Alexa as Scenes (Alexa.SceneController) which can be *activated by* a Routine but cannot *trigger* one. Bridge via `input_boolean.morning_alexa_radio_trigger`:

1. `input_boolean.morning_alexa_radio_trigger` defined in `configuration.yaml` and exposed to Alexa (appears as a "Contact Sensor" in Alexa's UI)
2. `script.morning` toggles the boolean ON, waits 2 s, toggles OFF — gives Alexa enough time to see the Open event
3. Alexa Routine: *When Morning Alexa Radio opens → Play BBC Radio 1 on TuneIn for 30 minutes on Kitchen Echo Show*

The original §5.2 below documents the legacy approach (TTS announce as a fallback when no Routine bridge existed). Kept as reference.

#### 5.2.bis Legacy fallback (TTS announce)

Currently `script.morning` step 2 calls `media_player.kitchen_echo` which doesn't exist (the new `alexa_devices` integration doesn't expose media players). Replacement using `notify.kitchen_echo_show_announce`:

```yaml
- alias: Kitchen Alexa morning announce
  service: notify.kitchen_echo_show_announce
  data:
    message: >-
      Good morning. {{ now().strftime('%A') }}, {{ states('weather.forecast_home') }},
      {{ state_attr('weather.forecast_home', 'temperature') }} degrees.
      {% set bin = state_attr('calendar.stafford_borough_council', 'message') %}
      {% set st  = state_attr('calendar.stafford_borough_council', 'start_time') %}
      {% if bin and st and st.startswith(now().strftime('%Y-%m-%d')) %}
        Bin day today — {{ bin }}.
      {% endif %}
```

BBC R4 playback is parked as a TODO (separate Voicemonkey bridge work documented in `~/.claude/TODO.md`).

### 5.3 3b Acceptance criteria

- [ ] Nabu Casa subscription active; HA Cloud + Alexa toggles on
- [ ] HA Companion app on Dan's + Nicola's iPhones works off-LAN via Cloud (no Tailscale needed)
- [ ] Alexa Smart Home Skill linked in the Alexa app; HA discovered as a hub
- [ ] All entities in the §5.2 exposure list discoverable
- [ ] "Alexa, turn off lounge lights" works on at least one Echo
- [ ] "Alexa, goodnight" fires `script.goodnight`
- [ ] "Alexa, fire on" toggles `climate.fire` (Nabu Casa-only capability — verify)
- [ ] "Alexa, what's William's blood sugar" returns the value (Nabu Casa-only — verify)
- [ ] Morning macro Kitchen Echo TTS announcement plays cleanly with weather + bin-day data

## 6. Sub-track 3c — Presence pilot

### 6.1 Hardware deployment

- **1× Aqara FP2** in Dan's office, ceiling-mounted opposite the desk
- **Pairing:** ZHA (already loaded as integration). FP2 may need ZHA quirks — install `zha-toolkit` HACS if entities don't auto-create
- **Resulting entities** (typical FP2 set):
  - `binary_sensor.fp2_office_presence` (whole-room) — *exact name TBD post-pairing; placeholder used in automations*
  - `sensor.fp2_office_illuminance` (lux)
  - `binary_sensor.fp2_office_zone_*` — zone-level (not used in v1)

### 6.2 Echo motion sensors leveraged for free

The `alexa_devices` integration already exposes motion binary sensors per Echo:
- `binary_sensor.kitchen_echo_show_motion`
- `binary_sensor.williams_echo_dot_motion`
- `binary_sensor.dan_s_2nd_echo_dot_motion`
- `binary_sensor.dan_s_3rd_echo_dot_motion`
- `binary_sensor.dan_s_4th_echo_dot_motion`

These are usable today, no new hardware needed.

### 6.3 Three starter automations

```yaml
# /config/automations.yaml — append

- id: office_presence_on
  alias: "Office: lights on when present"
  trigger:
    - platform: state
      entity_id: binary_sensor.fp2_office_presence
      to: 'on'
  condition:
    - condition: numeric_state
      entity_id: sensor.fp2_office_illuminance
      below: 50
    - condition: state
      entity_id: input_boolean.office_ac_override
      state: 'off'
  action:
    - service: scene.turn_on
      target: { entity_id: scene.room_office_default }

- id: office_presence_off
  alias: "Office: lights off after 5 min vacant"
  trigger:
    - platform: state
      entity_id: binary_sensor.fp2_office_presence
      to: 'off'
      for: { minutes: 5 }
  action:
    - service: light.turn_off
      target: { area_id: office }

- id: kitchen_night_light
  alias: "Kitchen: night light when motion 22:00–06:00"
  trigger:
    - platform: state
      entity_id: binary_sensor.kitchen_echo_show_motion
      to: 'on'
  condition:
    - condition: time
      after: "22:00:00"
      before: "06:00:00"
    - condition: state
      entity_id: light.kitchen_group
      state: 'off'
  action:
    - service: light.turn_on
      target: { entity_id: light.kitchen_group }
      data: { brightness_pct: 30, color_temp: 470 }
    - delay: { minutes: 3 }
    - service: light.turn_off
      target: { entity_id: light.kitchen_group }
```

### 6.4 3c Acceptance criteria

- [ ] Aqara FP2 paired to ZHA and reports presence + illuminance (real entity names captured for the automations above)
- [ ] FP2 placed in Dan's office; `office_presence_on` automation triggers on entry when lux <50 and override is off
- [ ] `office_presence_off` automation turns off office lights after 5 min vacant
- [ ] `kitchen_night_light` automation triggers when Echo Show motion fires between 22:00–06:00, lights kitchen at 30% for 3 min

## 7. Items wired with placeholders (resolve once entities land)

- **FP2 entity names** — guesses used in automations §6.3; verify after pairing and fix if different. Estimated 5-min sweep.
- **William's bedroom LED strip** — separately TODO'd (`~/.claude/TODO.md` → "William's LED strip"). Once added: tag into `william` area, group as `light.william_group`, expose to emulated_hue as "William's lights", surface on William's dashboard with full controls.
- **Office light group** (`light.office_group`) — currently `tile_room_lights` uses `light.office_front_1` directly. If a group is created (Track 4 polish), update §5.1 customize entry.

## 8. Risk register

| Risk | Mitigation |
|---|---|
| Network share backup briefly fails if Unraid rebooting | HA retries next day; Drive cloud agent provides redundancy |
| `emulated_hue` conflicts with a real Hue bridge on the LAN | None present; no conflict |
| FP2 false-positives in low light | Lux gate at <50 mitigates; iterate after a week |
| GitHub deploy key compromised | Private repo limits blast radius; rotate via deploy-key page |
| Auto-commit script masks accidental edits to `secrets.yaml` | `.gitignore` audit prevents commit; secrets file is never staged |
| Nabu Casa subscription lapses (forgotten payment) | Local + Unraid backups still work; Cloud Backup paused but no data loss. Alexa skill stops functioning; emulated_hue remains as documented fallback path (§11). |

## 9. Implementation prerequisites

Before implementation can start:

1. **Unraid `backup_ha` Samba share** must exist and be writable from HAOS network. Test with `smbclient -L //192.168.1.200/backup_ha`.
2. **Nabu Casa account** (free trial signup, payment method ready for the £5/mo).
3. **GitHub `gh` CLI authenticated** as `robinsondan87` for repo create.
4. **Aqara FP2** physically at hand and powered.

## 10. Out of scope

- emulated_hue / haaska — superseded by Nabu Casa Alexa Skill. Documented in §11 as fallback path if Nabu Casa is ever cancelled.
- BBC R4 playback in Morning macro — Voicemonkey bridge separately TODO'd (Nabu Casa's Alexa Skill is HA→Alexa direction, doesn't expose Echo media playback to HA)
- Wyoming/Whisper/Piper local voice — explicitly skipped
- NodeRED/Pyscript/AppDaemon — explicitly skipped
- Email / SMS / chat notifiers — explicitly skipped
- More FP2 / mmWave devices — depends on pilot success
- William's LED strip — separately TODO'd (will wire in once user adds it)
- Tablet / wall-mount kiosk hardware — Track 4 territory
- Smart locks, TRVs, additional cameras — Track 4 territory

## 10b. Implementation notes / gotchas observed during landing

- **Cloud Backup agent requires HA Core restart.** Toggling Alexa/Cloud prefs alone doesn't make the `cloud.cloud` backup agent appear. After enabling subscription + Alexa/Remote, restart HA core (`POST /api/services/homeassistant/restart`) — cloud component re-init then registers the backup agent. Without restart, only `hassio.local` and any network-share agents show up.
- **Unraid Docker template "extra paths" don't always persist.** When you click Edit on a container in Unraid Docker UI and change one config (e.g. add a new bind mount), Unraid sometimes drops *previously-added* path bindings that weren't in the original template (only the template's `<Config Type="Path">` entries persist by default). Workaround: after any Edit, immediately verify all previous binds are still listed via `docker inspect $cid --format "{{range .Mounts}}{{.Source}} → {{.Destination}}{{println}}{{end}}"`. If any are missing, re-add them in the same Edit pass.
- **Unraid UI input has trailing-space hazard.** Path/Container Path text fields silently accept trailing whitespace, which becomes part of the actual bind mount destination. Always trim. Symptom: `docker exec $cid ls /your-path` returns "No such file or directory" even though `docker inspect` shows the mount. Fix is to edit the template XML directly: `sed -i 's|Target="/your-path "|Target="/your-path"|g' /boot/config/plugins/dockerMan/templates-user/my-syncthing.xml` then recreate container.
- **HA scripts expose to Alexa as `Alexa.SceneController` (not switches).** This means `script.morning` etc. show up in the Alexa app's **Scenes** section, not Devices. They can be *activated by* a Routine but *cannot trigger* one. To trigger a Routine from a script firing, bridge through an `input_boolean` (which exposes to Alexa as a contact sensor — Routine fires on "opens"). Pattern is `script.morning → input_boolean.turn_on → 2-sec delay → input_boolean.turn_off` so the boolean blip is detectable by Alexa cloud.
- **Nabu Casa exposure: per-entity `should_expose` is the master gate.** `alexa_default_expose` controls only the default for unset entities. To strictly limit to a curated list: set `should_expose: True` for each entity in the keep-list, then `should_expose: False` for all others. Use WS `homeassistant/expose_entity` with `assistants: ['cloud.alexa']`. Alexa filters discovery against per-entity flags; `cloud/alexa/entities` shows capabilities for *all* entities regardless and is misleading — actual discovery uses the should_expose gate.

## 11. Open questions / future work

- **Voicemonkey BBC R4 bridge** — TODO. Trigger via HA webhook → Alexa Routine plays radio. (Nabu Casa's Alexa Skill doesn't help here — that's HA→Alexa control, not Echo media playback exposed to HA.)
- **More presence devices** — depends on FP2 office pilot's value over the next 1-2 weeks.
- **Office light group entity** — minor cleanup; currently tile_room_lights references a single bulb (`light.office_front_1`).
- **Fallback path if Nabu Casa is cancelled** — `emulated_hue` for Alexa control + Tailscale-only for remote access + Google Drive Backup add-on. All three were the originally-specced DIY paths and remain documented patterns to fall back to. No design work required to revert.
