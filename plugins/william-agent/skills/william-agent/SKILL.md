---
description: Full setup guide for William Robinson's dedicated child-safe OpenClaw agent — WhatsApp only, Nightscout/AAPS-aware, diabetes monitoring, school support, weekly tasks.
---

# William's Agent

William Robinson's dedicated OpenClaw agent. Fully configured and live as of April 2026.

## About William

| Field | Value |
|---|---|
| Full name | William Robinson |
| DOB | 10 January 2016 (age 10, turns 11 Jan 2027) |
| Year group | Year 5 (2025/26) → Year 6 Sept 2026 → KS2 SATs May 2027 |
| Medical | Type 1 diabetes since age 2, AAPS closed loop (SMB), Dexcom CGM |
| Interests | Lego (especially Technic), maths puzzles, Minecraft (weekends only) |
| Academic | Strong at maths — stretch to Y7/8 level |
| WhatsApp | +447884926246 |
| Parents | Dan Robinson +447356168888, Nicola Robinson +447966081535 |

## Agent config

| Item | Value |
|---|---|
| Agent ID | `william-agent` |
| Workspace | `~/.openclaw/workspace/agents/william-agent/` |
| Agent dir | `~/.openclaw/agents/william-agent/agent/` |
| Model | `openai-codex/gpt-5.4-mini` / fallback `openrouter/openai/gpt-5.4-mini` |
| Channel | **WhatsApp only** — no Discord |
| Tools | `profile: full`, `alsoAllow: [browser, message]` |

## WhatsApp binding

William's number is routed to `william-agent` via a specific peer binding **placed before** the eve-family WhatsApp catch-all in `bindings[]`:

```json
{
  "agentId": "william-agent",
  "match": {
    "channel": "whatsapp",
    "peer": { "kind": "dm", "id": "+447884926246" }
  }
}
```

`dmPolicy` is `pairing` but numbers in `allowFrom` connect without a manual pairing step — William messaged straight through without needing approval.

## Nightscout

| Item | Value |
|---|---|
| URL | `https://william.eu.nightscoutpro.com` |
| Auth | **None required** — public read access |
| Units | mg/dL — divide by 18 for mmol/L |
| App | AAPS (AndroidAPS, SMB algorithm) on Google Pixel 8a |
| Key endpoints | `/pebble` (BG, direction, IOB, COB), `/api/v1/entries.json`, `/api/v1/devicestatus.json?count=1` |

### AAPS-aware correction logic

William runs a **closed loop** — never calculate corrections with naive ISF maths. Always read `openaps.suggested` from devicestatus:

- `insulinReq = 0` + eventualBG in range → loop handling it, alert parents but no manual correction needed
- `insulinReq > 0` → loop wants more insulin, flag as potential manual correction
- `eventualBG < 4.0 mmol/L` (72 mg/dL) → loop predicting drop — **do not correct**, warn parents
- Persistent high (≥15 for 45+ min, not falling) → suggest manual correction even if insulinReq=0, flag "check site/cannula"

Target BG: 6.5 mmol/L (117 mg/dL). ISF is dynamic (`runningDynamicIsf: true`), ~2.3 mmol/L per unit.

Lows are handled by Dexcom — don't send hypo alerts unless William says he feels unwell AND Nightscout confirms <4.

### Fetching Nightscout from cron

**Use the `fetch` tool directly — never exec or inline scripts.** Exec is blocked by security policy for complex invocations (heredocs, inline node/python).

```
GET https://william.eu.nightscoutpro.com/pebble
GET https://william.eu.nightscoutpro.com/api/v1/devicestatus.json?count=1
```

## Cron jobs

| Job | Schedule | ID |
|---|---|---|
| `william-high-glucose-check` | every 30 min | `9a6c3836-1c3c-4df0-a363-f35d32742f05` |
| `william-weekly-summary` | Sun 7pm Europe/London | `7db7a4e7-5f1d-4b80-a6cf-1515c68864ed` |
| `william-monday-tasks` | Mon 6:15am Europe/London | `beb6d5dd-4921-4f96-a517-cb7fb082c3c2` |

All use `--model openai-codex/gpt-5.4-mini --light-context --session isolated`.

### Critical: cron WhatsApp delivery

**Always set `--channel whatsapp --to <number>` explicitly.** Without `--channel`, delivery fails with:
> "Channel is required when multiple channels are configured: discord, whatsapp"

```bash
openclaw cron add ... --channel whatsapp --to +447884926246
# or edit existing:
openclaw cron edit <id> --channel whatsapp --to +447884926246
```

The `message` tool is **unavailable in isolated cron sessions** — use `announce` delivery mode (default) with explicit `--to` and `--channel`. The cron's output text becomes the WhatsApp message.

## Workspace files

All live at `~/.openclaw/workspace/agents/william-agent/`:

| File | Purpose |
|---|---|
| `SOUL.md` | Full personality, diabetes protocol, school support, safety rules |
| `IDENTITY.md` | Name, role, vibe |
| `AGENTS.md` | Routing rules, domain ownership table |
| `USER.md` | William's profile, family, SATs timeline |
| `TOOLS.md` | Nightscout endpoints, unit conversion, parent numbers |
| `MEMORY.md` | Stable context, seeded on setup |
| `memory/glucose-log.md` | CGM readings + context |
| `memory/meal-log.md` | Food / carb estimates |
| `memory/insulin-log.md` | Bolus events |
| `memory/homework-log.md` | School topics covered |
| `memory/interests.md` | Favourite builds, sets, topics |
| `memory/weekly-notes.md` | Running notes for Sunday summary (cleared weekly) |
| `memory/weekly-tasks.md` | This week's 5 tasks (written Sunday, read Monday) |
| `memory/last-high-alert.md` | Debounce timestamp for high glucose alerts |

**Create stub memory files on setup** — the agent errors if it tries to read missing files:
```bash
touch ~/.openclaw/workspace/agents/william-agent/memory/{glucose-log,meal-log,insulin-log,homework-log,interests,weekly-notes,weekly-tasks,last-high-alert}.md
```

## Food photos → carb estimation

When William sends a photo of food with no context, treat it as a carb estimation request. Identify the food, estimate grams with a range (e.g. "roughly 45–55g"), note the main carb sources, log to meal-log.md. Never suggest an insulin dose — AAPS handles dosing.

## Weekly tasks rules

1. **No Minecraft** — William has no computer access during the week
2. **No puzzle answers** — William asks to check his answer; agent confirms and explains why
3. **Every task needs a written paragraph** (3–5 sentences) — that's the measurable completion
4. Rotate across: YouTube video, article/read, maths puzzle, Lego challenge, wildcard

Each task format:
```
Task N: [emoji] [title] — [one line]
What to do: [instruction]
Write back: [specific paragraph prompt]
```

Sunday cron generates tasks → saves to `memory/weekly-tasks.md` → Monday 6:15am cron reads and sends to William via WhatsApp announce delivery.

## Key gotchas

- **`message` tool unavailable in isolated cron sessions** — use announce delivery with `--channel whatsapp --to <number>`
- **Never use exec for HTTP** — blocked by security policy. Use fetch tool directly
- **AAPS closed loop** — always check `insulinReq` and `eventualBG` before suggesting correction
- **Nightscout units are mg/dL** — always divide by 18, never pass raw values to William
- **dmPolicy pairing** — numbers in `allowFrom` bypass pairing, no manual approval step needed
- **WhatsApp binding order matters** — william-agent binding must come before the eve-family catch-all in `bindings[]`
- **Stub memory files** — create them on setup or the agent will error on first cron run

## Related agents / skills

- `eve-family` — family front door (WhatsApp catch-all for Dan + Nicola)
- `dan-gym-coach` — Dan's health/gym agent (similar isolated cron pattern)
