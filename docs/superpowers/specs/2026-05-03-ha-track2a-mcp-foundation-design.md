# Home Assistant Track 2a — MCP Foundation

**Status:** Draft, brainstormed 2026-05-03
**Track:** 2 of 4 (HA overhaul) — sub-spec **2a** of 2 (2b will design specific agent behaviours on top of this foundation)
**Owner:** Dan Robinson
**Plane:** HA-7 (epic) + this spec landed → fresh issue per implementation phase

## 1. Context & goals

Track 2 of the HA overhaul is about agent ↔ HA integration: what Claude (in this CLI session and any future one) and the OpenClaw agents should be able to do with HA, and how that wiring is set up.

Originally scoped at six sub-areas during Track 3 brainstorming (permissions / William-agent crossover / eve-family broadcasting / Claude Code MCP / off-network exposure / capability inventory). Decomposed and trimmed during 2026-05-03 brainstorming to:

- **2a (this spec):** the foundation — install/wire the MCP transport, define identities, set permissions
- **2b (next spec):** the behaviours — what specific things agents do (BG-aware patterns, family Q&A, Echo broadcasts, etc.)

The original 2c (off-network MCP exposure via Cloudflare Tunnel) was **dropped** — Tailscale already covers any remote access need, and everything important runs from the Mac mini on LAN.

## 2. Decisions made during brainstorming

| Decision | Choice | Why |
|---|---|---|
| Primary use cases | **Claude Code productivity (A) + EVE Home / family Q&A + broadcast (C)** | William-agent (B) keeps its own Nightscout pipeline; off-network (D) covered by Tailscale |
| Claude Code permission level | **Read + control + admin** (full surface) | Matches what's currently being done via SSH + REST anyway; familiar territory |
| EVE Home permission level | **Read + announce + control** (no admin) | Family-safe scope; can talk to Echos and toggle lights but can't restart HA or edit YAML |
| Claude Code transport | **HAOS `homeassistant-ai/ha-mcp` add-on** (already installed, port 9583, secret-path-in-URL auth) | Already there; richer than HA's native Assist-scoped `mcp_server` integration |
| EVE Home transport | **HA REST + WebSocket** with long-lived access token | OpenClaw agents already have HTTP tools; no waiting for OpenClaw native MCP-client support |
| `unraid-mcp` registration | **Out of scope** for Track 2a | Different backend (Unraid host); ~5-min follow-on, not part of HA agent integration |
| Off-network exposure (was 2c) | **Dropped** | Tailscale covers it |
| Identity model | **Two HA users**: `claude_code` (admin), `eve_home` (non-admin, curated entity exposure) | Permissions enforced HA-side, not transport-side; auditable per-user |

## 3. Architecture

```
                  ┌─────────────────────┐
                  │  Home Assistant OS  │
                  │   192.168.1.151     │
                  │                     │
                  │  ┌───────────────┐  │
                  │  │ ha-mcp add-on │  │ ◄── Claude Code (your sessions)
                  │  │   :9583       │  │     via secret-path URL
                  │  │   /private_…  │  │     auth = HA user `claude_code` (admin)
                  │  └───────────────┘  │
                  │                     │
                  │  ┌───────────────┐  │
                  │  │  HA REST + WS │  │ ◄── EVE Home (OpenClaw)
                  │  │  :8123/api/…  │  │     via LLAT
                  │  └───────────────┘  │     auth = HA user `eve_home` (non-admin)
                  └─────────────────────┘
```

Both transports talk to the same HA backend; both are gated by per-HA-user permissions. Agent identity is recorded against every action via the HA user → useful for audit + accountability.

## 4. Components

### 4.1 HAOS `ha-mcp` add-on (already installed)

- Image: `ghcr.io/homeassistant-ai/ha-mcp` (HACS add-on by `homeassistant-ai`)
- Endpoint: `http://192.168.1.151:9583/<secret_path>`
- Auth: secret-path-in-URL (single-secret model — anyone with the URL has full access; treat the URL as a password)
- Current secret path: `/private_eDpQpJz3krP08FFNJVVXZA`
- **Action:** take the **7.4.0 → 7.4.1** update (one tap on the add-on Info screen)
- **Configuration confirmed:**
  - `Backup hint: normal`
  - `Enable skills: ON` (best-practice MCP resources)
  - `Enable skills as tools: ON` (clients without resources support get them as tools)
  - `Enable tool search: OFF` (Claude has plenty of context — no need for the 5K vs 46K reduction)

### 4.2 HA users

Created in **Settings → People → Users**:

1. **`claude_code`** — administrator. Password stored in SOPS as `HA_CLAUDE_CODE_PASSWORD`. LLAT stored as `HA_CLAUDE_CODE_TOKEN`.
2. **`eve_home`** — non-administrator. Password stored in SOPS as `HA_EVE_HOME_PASSWORD`. LLAT stored as `HA_EVE_HOME_TOKEN`.

**Auth-flow clarification (caveat — verify on first run):**

- The `ha-mcp` add-on uses its **own secret-path-in-URL auth**, *not* per-HA-user. Actions performed via the add-on may be attributed to a single shared system identity (e.g. `Supervisor`) rather than the `claude_code` user. **Audit attribution caveat:** if it turns out the add-on doesn't impersonate per-user, the `claude_code` HA user becomes cosmetic / fallback only. Verify by hitting an `light.toggle` via MCP and checking HA Logbook for the user attribution.
- The `eve_home` LLAT, by contrast, IS used directly in `Authorization: Bearer …` headers by the OpenClaw `ha-tools` skill — every REST call carries the `eve_home` identity, so audit attribution is clean here.
- **Implication:** if claude_code-via-MCP doesn't get per-user audit, fall back to the long-lived token approach for Claude Code too (REST instead of MCP), or open an issue with the `ha-mcp` add-on upstream. Decision deferred to implementation.

### 4.3 Entity exposure curation for EVE Home

`eve_home` user gets an opt-in exposure list scoped to read + announce + control. Pattern same as Alexa exposure (Track 3b §5.2). Initial list:

- All 8 light groups (lounge / kitchen / library / dining / hall / landing / office / office_outside)
- All 4 Echo announce notify entities (`notify.kitchen_echo_show_announce`, `notify.williams_echo_dot_announce`, `notify.dan_s_2nd_echo_dot_announce`, etc.)
- All scenes (`scene.lounge_movie`, `scene.room_*_default`, `scene.kitchen_morning_warm`, etc.)
- Scripts: `script.goodnight`, `script.morning`, `script.movie` (so EVE can fire macros)
- All `binary_sensor.*` for occupancy / motion / door / window
- All `sensor.*` for temp / humidity / energy / cat feeders / Volvo / weather (read-only useful)
- `climate.fire`, `climate.office_aircon` (control)
- `lock.volvo_ex40_lock` (read only — locking the car remotely is an admin act)
- `person.dan`, `person.nicola`, `person.william`

Explicit exclusions (NOT exposed to EVE Home):
- HA admin entities (system updates, addon controls, restart buttons)
- Settings-level entities
- Any `input_*` helper entities not needed for family Q&A
- `update.*` entities

### 4.4 Claude Code MCP registration

Add to `~/.claude.json` under `mcpServers`:

```json
{
  "mcpServers": {
    "home-assistant": {
      "transport": "http",
      "url": "http://192.168.1.151:9583/private_eDpQpJz3krP08FFNJVVXZA"
    }
  }
}
```

Restart Claude Code after editing for tools to load. Confirm available via `claude mcp list`.

### 4.5 OpenClaw `ha-tools` skill (new)

Custom skill at `~/Projects/RobboHome-skills-Claude/plugins/ha-tools/skills/ha-tools/SKILL.md` exposing four tool patterns to EVE Home:

```python
ha_get_state(entity_id: str) -> str          # current state
ha_get_attributes(entity_id: str) -> dict    # full attribute dict
ha_call_service(domain, service, target, data) -> dict   # generic service call
ha_announce(echo: str, message: str)         # wrapper for notify.<echo>_announce
ha_list_entities(area=None, label=None, domain=None)     # discovery via /api/states + filters
```

All four use HA REST API at `http://192.168.1.151:8123/api/...` with `Authorization: Bearer ${HA_EVE_HOME_TOKEN}` from OpenClaw's secrets store.

### 4.6 EVE Home agent updates

Edit `~/.openclaw/workspace/agents/home-assistant/TOOLS.md` to replace the template with the real inventory:

- **Areas + their floors** (10 indoor + 5 outdoor + outside-office)
- **Echos** (5: kitchen_echo_show, williams_echo_dot, dan_s_2nd/3rd/4th_echo_dot — with rooms once HA-15 lands)
- **Light groups** (8) and what each contains
- **Macros** (goodnight / morning / movie)
- **People** (dan / nicola / william + their device trackers)
- **Cats** (3 feeders: khaos / kiki / lunar — for fed-today Q&A)

## 5. Permissions / safety enforcement

| Layer | Mechanism | Who's protected |
|---|---|---|
| Transport | ha-mcp add-on uses secret-path URL; only those with the URL can hit it | All other LAN users / lateral movement |
| HA-side identity | Per-user HA accounts; admin vs non-admin | Stops EVE Home from doing admin acts even with valid token |
| Entity exposure | Per-user opt-in exposure list (`eve_home` gets a curated list, NOT default-expose) | Stops accidental discovery of sensitive entities (cameras, locks, internal sensors) |
| Audit | HA logs every service call with the user identity; visible in Logbook + History | All actions traceable to which agent did what |

## 6. Acceptance criteria

- [ ] `ha-mcp` add-on updated to 7.4.1 and running
- [ ] HA user `claude_code` exists, admin, password in SOPS
- [ ] HA user `eve_home` exists, non-admin, password + LLAT in SOPS, curated entity exposure applied
- [ ] `~/.claude.json` registers the `home-assistant` MCP; new Claude Code session shows tools loaded via `claude mcp list`
- [ ] OpenClaw `ha-tools` skill exists with 5 documented tool patterns; `EVE Home` agent's TOOLS.md updated with real inventory
- [ ] EVE Home can answer "is the back door open?" and "who's home?" via the new tools (one round-trip test each)
- [ ] EVE Home can announce a test message on `notify.kitchen_echo_show_announce`
- [ ] HA Logbook shows each action attributed to the right HA user

## 7. Out of scope (saved for later)

- **2b (next spec):** specific agent *behaviours* — BG-aware patterns, ambient Q&A phrasings, scheduled summaries, alert routing. Just plumbing here, not what they DO.
- **`unraid-mcp` registration in `~/.claude.json`** — separate 5-min task. Not blocked on this spec.
- **William-agent ↔ HA crossover** — explicitly dropped during brainstorm; William keeps his own Nightscout pipeline.
- **Off-network MCP exposure (Cloudflare Tunnel + Access)** — explicitly dropped during brainstorm; Tailscale covers it.
- **Polishing the native HA `mcp_server` integration** (Assist-based) — not removed but not used; coexists harmlessly.

## 8. Risks

| Risk | Mitigation |
|---|---|
| Secret URL path leaked → unauthenticated full HA access | Treat URL as password; rotate via add-on Configuration if exposed; keep URL out of git/TODO/spec history (placeholder text only) |
| EVE Home LLAT leaked → impersonation up to its scope | LLAT is non-admin + curated exposure → blast radius limited; rotate at HA Profile screen |
| ha-mcp add-on update breaks tool surface | Take updates one at a time; verify with `claude mcp list` post-update; rollback via add-on UI if needed |
| Per-user HA permission complexity drifts | Document the `eve_home` exposure list in TOOLS.md; review on quarterly basis |

## 9. Open questions / future work

- **2b** — design specific agent behaviours (the next spec)
- Audit dashboard — could surface "what EVE Home and Claude Code did in the last 7 days" via HA Logbook + Plane TODO summaries
- Token rotation cadence — currently no formal schedule; consider quarterly or on-event
