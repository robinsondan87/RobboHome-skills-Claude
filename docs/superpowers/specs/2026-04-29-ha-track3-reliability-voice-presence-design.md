# Home Assistant Track 3 — Reliability, Voice, Presence

**Status:** Draft, brainstormed 2026-04-29
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
| Remote access | **Stay on Tailscale** (no Nabu Casa) | Family doesn't want subscriptions; Tailscale already covers all fleet machines and works for HA Companion remote access |
| Alexa voice control | **`emulated_hue` (DIY, free)** | No subscription; LAN-only is fine; covers lights + scenes which is 80% of family use |
| Local voice stack | **Skip** (no Wyoming/Whisper/Piper) | Family lives on Alexa already; revisit only if Alexa proves limiting |
| Automation engine | **Stay native HA YAML** | 50 automations + 16 scripts work fine; no complexity problem |
| Notifier | **Mobile push + Alexa announce only** | No need for SMS/email/chat; OpenClaw agents handle their own |
| Backup strategy | **Three layers: local + Unraid syncthing chain + Google Drive** | Defence in depth for free; Drive cloud agent is the off-LAN copy |
| Config-as-code | **Auto-commit + push to private GitHub** | Full edit history; `/config/.git` already initialised |
| Presence hardware | **Single FP2 pilot in Dan's office** | Validate value before scaling |

## 3. Sub-track decomposition

Three independent specs, each with its own acceptance criteria. Order: 3a → 3b → 3c (each is independent; nothing strictly blocks the others, but doing 3a first means later changes are git-tracked from day one).

| Sub-track | Scope summary |
|---|---|
| **3a — Reliability** | HA backups → Unraid network share + Google Drive cloud agent. `/config/.git` auto-commit + push to GitHub. |
| **3b — Voice (Alexa)** | `emulated_hue` integration so Echos can control lights + scenes + macro scripts. Morning macro Kitchen Echo step fixed to use `notify.kitchen_echo_show_announce`. |
| **3c — Presence pilot** | Deploy FP2 in Dan's office. Three starter presence automations (office on, office off, kitchen night-light via Echo motion). |

## 4. Sub-track 3a — Reliability

### 4.1 Backup architecture

```
        HA daily auto-backup
                │
        ┌───────┼───────────┐
        ▼       ▼           ▼
  /backup/   /share/     Google Drive
  (HAOS)     unraid      (Backup add-on)
             backup
              │
              ▼
        syncthing on Unraid
              │
              ▼
            svr003
```

Three geographic copies of every backup: HAOS partition, Unraid + svr003 (LAN-twinned), Google Drive (cloud). The user's existing syncthing chain on Unraid is reused — no new sync infrastructure needed.

### 4.2 Components

1. **HAOS network storage mount** — *Settings → System → Storage → Add network storage* with a CIFS/SMB share to Unraid (e.g. `\\192.168.1.200\backup_ha`). HAOS automatically exposes mounted shares as backup *agents*.
2. **Backup agent configuration** — daily auto-backup writes to both `local` and the new network share. Retention: 7 local, 14 network share.
3. **Google Drive Backup add-on** — community add-on at `https://github.com/sabeechen/hassio-google-drive-backup`. Installs alongside; runs daily. Retention: 30 days. User authenticates once via OAuth.
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

- [ ] HA daily auto-backup writes to both `local` and the Unraid network share (verify in HA backup logs)
- [ ] Google Drive Backup add-on is installed, authenticated, and uploads daily backups
- [ ] Syncthing on Unraid carries new HA backups to svr003 (verify by listing svr003's backup dir)
- [ ] `/share/auto_commit.sh` runs every 15 min via cron, commits any pending `/config/` changes, pushes to GitHub
- [ ] `github.com/robinsondan87/robbohome-ha-config` private repo exists and shows recent commits
- [ ] `.gitignore` excludes secrets, `.storage/`, db files, logs

## 5. Sub-track 3b — Voice (Alexa via emulated_hue)

### 5.1 Configuration

In `/config/configuration.yaml`:

```yaml
emulated_hue:
  host_ip: 192.168.1.151
  listen_port: 80
  expose_by_default: false
  exposed_domains:
    - light
    - switch
    - scene
    - script
```

Per-entity exposure via `customize.yaml` (explicit opt-in only):

| Entity | `emulated_hue_name` Alexa hears |
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

After config save + HA restart → Alexa app → Devices → Add Device → "Other" / "Hue" → Echos discover HA-as-Hue. LAN-only.

### 5.2 Morning macro Kitchen Echo fix

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

- [ ] `emulated_hue` integration loaded with explicit per-entity exposure list
- [ ] All 6 room light groups + Office + 3 macro scripts + Movie scene discoverable by Alexa
- [ ] "Alexa, turn off lounge lights" works on at least one Echo
- [ ] "Alexa, goodnight" fires `script.goodnight`
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
| Port 80 already bound on HAOS (rare — but Pi-hole / adguard add-ons can do this) | Configure `emulated_hue` with `listen_port: 8300, advertise_port: 80, upnp_bind_multicast: true` and rely on UPnP for Echo discovery. Most Echos honour this; if not, run emulated_hue in a separate container or change Pi-hole's port. |

## 9. Implementation prerequisites

Before implementation can start:

1. **Unraid `backup_ha` Samba share** must exist and be writable from HAOS network. Test with `smbclient -L //192.168.1.200/backup_ha`.
2. **Google account** for the Backup add-on OAuth.
3. **GitHub `gh` CLI authenticated** as `robinsondan87` for repo create.
4. **Aqara FP2** physically at hand and powered.

## 10. Out of scope

- haaska upgrade — separately TODO'd
- BBC R4 playback in Morning macro — Voicemonkey bridge separately TODO'd
- Wyoming/Whisper/Piper local voice — explicitly skipped
- NodeRED/Pyscript/AppDaemon — explicitly skipped
- Email / SMS / chat notifiers — explicitly skipped
- More FP2 / mmWave devices — depends on pilot success
- William's LED strip — separately TODO'd (will wire in once user adds it)
- Tablet / wall-mount kiosk hardware — Track 4 territory
- Smart locks, TRVs, additional cameras — Track 4 territory

## 11. Open questions / future work

- **Voicemonkey BBC R4 bridge** — TODO. Trigger via HA webhook → Alexa Routine plays radio.
- **emulated_hue → haaska upgrade** — TODO. When family hits limits of emulated_hue (climate via voice, sensor reads, full HA service surface).
- **More presence devices** — depends on FP2 office pilot's value over the next 1-2 weeks.
- **Office light group entity** — minor cleanup; currently tile_room_lights references a single bulb (`light.office_front_1`).
