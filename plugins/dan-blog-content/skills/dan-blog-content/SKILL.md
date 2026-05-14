---
description: Reference for Dan's blog at blog.robbohome.com — the admin UI, the JSON ideas API, and the dan-blog-content OpenClaw agent. Use when helping Dan with anything blog-shaped (capture, draft, publish, voice, agent troubleshooting).
---

# Dan's blog — admin, API, and agent

A small companion reference so future-Claude knows the blog stack exists without Dan re-explaining.

## Site

| Item | Value |
|---|---|
| Project path | `/Users/robbohomebot/Projects/BLOG` |
| Public URL | https://blog.robbohome.com |
| Repo | `robinsondan87/robbohome-blog` (private) |
| Stack | Astro 5 hybrid (Node adapter, standalone), nginx-removed in v0.7.0 |
| Internal URL | `http://192.168.1.17:3003` on svr002 |
| Container | `robbohome-blog` (Docker on svr002) |
| Deploy | Pattern A — `make bump-patch` → `git push` → CI on the repo's self-hosted runner |
| Data volume | `/home/robbohomebot/data/robbohome-blog/data` on svr002 (SQLite admin DB lives here) |

## Routes

**Public, prerendered:**
- `/`, `/blog`, `/blog/<slug>`, `/blog/tag/<tag>`, `/notes`, `/notes/<slug>`, `/log`, `/projects`, `/projects/<slug>`, `/about`, `/now`, `/contact`, `/404`, `/rss.xml`, `/sitemap-index.xml`
- `/og/default.png`, `/og/blog/<slug>.png` — OG card images (build-time)

**SSR (require auth):**
- `/admin/login` — password form (open)
- `/admin/ideas` — capture + list + state transitions (cookie-gated)
- `/admin/logout` — POST clears cookie
- `/api/health` — bearer-gated ping
- `/api/ideas[?state=]` — list / create
- `/api/ideas/:id` — read / patch / delete

## Auth

| Surface | Mechanism | Where to find |
|---|---|---|
| `/admin/*` UI | `APP_PASSWORD` env var + HMAC-signed cookie (`AUTH_SECRET`) | GitHub secret on the repo; stash in 1Password |
| `/api/*` JSON | `Authorization: Bearer $ADMIN_API_TOKEN` | GitHub secret on the repo |
| Agent's API token | Plain text file (mode 0600) | `/Users/robbohomebot/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt` |

`security.checkOrigin` is disabled in `astro.config.mjs` — the HMAC cookie does the CSRF job.

## Ideas API quick reference

```bash
TOKEN=$(cat ~/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt)
BASE=https://blog.robbohome.com/api

curl -sS -H "Authorization: Bearer $TOKEN" $BASE/health
curl -sS -H "Authorization: Bearer $TOKEN" "$BASE/ideas?state=idea"
curl -sS -X POST $BASE/ideas \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"title":"...","notes":"..."}'
curl -sS -X PATCH $BASE/ideas/<id> \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"state":"drafting","draft_markdown":"..."}'
curl -sS -X DELETE -H "Authorization: Bearer $TOKEN" $BASE/ideas/<id>
```

**Idea shape:** `{id, title, notes, state, draft_markdown, created_at, updated_at}` where `state ∈ {idea, drafting, done, dropped}`.

## OpenClaw agent: `dan-blog-content`

Runs on Discord channel `#dan-blog` (id `1504369430438608896`) in the personal guild (`1473044168925253724`). Workspace at `~/.openclaw/workspace/agents/dan-blog-content/`. Model: `openai-codex/gpt-5.4`.

### Three-layer Discord agent setup (the gotcha)

When wiring a new dan-* agent to a Discord channel, three config layers in `~/.openclaw/openclaw.json` need entries — `openclaw agents add` only does layer 1, `openclaw agents bind` only does layer 2:

1. **`c.agents.list[]`** — agent identity, model `{primary, fallbacks}`, **`tools` block with `profile: "full"` + `alsoAllow: ["browser","fetch","message"]` + exec block**. Without `message` in the tools list, the agent returns empty replies.
2. **`c.bindings[]`** — `{agentId, match: {channel: "discord", peer: {kind: "channel", id: "<channelId>"}}}`.
3. **`c.channels.discord.guilds[<guildId>].channels[<channelId>]`** — `{requireMention: false, users: ["472730567163772928"], enabled: true}`. **This is the silent-failure layer** — without it, the discord plugin drops inbound messages before routing sees them.

After editing: **`launchctl kickstart -k gui/$UID/ai.openclaw.gateway`** (NOT just `openclaw secrets reload` — the channel allowlist needs a real reconnect).

### Voice rules pinned in SOUL.md

When generating any blog copy (whether for Dan to publish or just in conversation), respect these. They were established across several sessions and survive every session via the agent's SOUL.md:

1. **OpenClaw was NOT written by Dan** — he adopted it. Upstream is openclaw.ai. Frame as "the runtime I run my fleet on," never "the runtime I built."
2. **3D printing / GeekyThings is a hobby in public copy** — never "side business," "side hustle," "marketplaces," "product listings," "customers," "selling." Bet365 employment terms.
3. **No identifying child / school / clinic / address detail.** Dan's son's existence is on the site; specifics are not.
4. **No Bet365 internal specifics.** Day-job framing is "Site Reliability Tech Lead at a large UK tech company."
5. **British spelling, first-person, no marketing voice.** No "elevate," "leverage," "unlock," no exclamation marks, no "What do you think? Drop a comment."
6. **Don't claim authorship of anything Dan only runs/configured.**

## Content collections + frontmatter

`src/content/blog/<slug>.md`:
```yaml
title: ...
description: ...
pubDate: 2026-MM-DD
tags: [sre-at-home, agents, homelab, home-assistant, observability, walked-back]
heroIllustration: homelab | agent-fleet | none
pinned: true | false
pinnedOrder: 1
series: <name>  # optional
seriesPart: 1
seriesTotal: 3
```

`src/content/notes/<slug>.md`: `title`, `pubDate`, optional `sourceUrl` + `sourceLabel` + `tags`.
`src/content/log/<slug>.md`: `date`, `kind` (`reading|sketch|parked|note|shipped`), `title`, optional `detail` + `link`.
`src/content/projects/<slug>.md`: `name`, `tagline`, `status`, `order`, `section`, `heroIllustration`, etc.

## Common operations

| Task | How |
|---|---|
| Capture an idea from Discord | Post in `#dan-blog`, agent calls `POST /api/ideas` |
| Capture from anywhere | Curl `POST $BASE/ideas` with bearer token |
| See current inbox | https://blog.robbohome.com/admin/ideas or `GET /api/ideas?state=idea` |
| Develop an idea into a draft | Ask agent in `#dan-blog`; agent PATCHes `draft_markdown` + state=drafting |
| Read a draft | Admin UI shows `<details>` Draft (N chars) panel; or `GET /api/ideas/:id` |
| Publish a post | Copy `draft_markdown` from admin UI into `src/content/blog/<slug>.md` with frontmatter, commit, `make bump-patch`, push. Then PATCH state=done on the idea. |
| Rotate `APP_PASSWORD` | `gh secret set APP_PASSWORD --repo robinsondan87/robbohome-blog --body '<new>'` then trigger workflow |
| Rotate `ADMIN_API_TOKEN` | Same as above, but also update `~/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt` (it's a local copy, not a SecretRef) |
| Invalidate all admin sessions | Rotate `AUTH_SECRET` instead (HMAC cookie verification fails) |
| Restart the gateway | `launchctl kickstart -k gui/$UID/ai.openclaw.gateway` then wait ~30–60s for both channels `connected` |

## Visual / editorial conventions

- **Newsprint palette**: cream `#f4f1e4` / black `#0a0908` / no chrome accent. Warm ink amber (`#8a6d2b`) used only in SVG illustrations and log bullets.
- **Type system**: Source Serif 4 body, JetBrains Mono headings + wordmark + meta, Inter nav.
- **Editorial post layout**: uppercase mono H1, italic serif lede, mono small-caps meta line (date · reading time · tags), `⁂` asterism between major sections, opt-in drop caps via `class="has-dropcap"` on `<article>`.
- **Illustrations**: hand-drawn SVG schematics (`HomelabSketch`, `AgentFleet`, `RackMark` mark beside wordmark, `AuthorMark` monogram on About). Toggled per-post via `heroIllustration` frontmatter.

## What I should NOT do without Dan's say-so

- Auto-commit a draft to the blog repo (the `POST /api/ideas/:id/publish` endpoint isn't built yet — Dan reviews + ships manually from the admin UI).
- Post anything to LinkedIn. That's `dan-linkedin`'s job. The blog agent hands off via a brief — Dan moves it across channels.
- Touch the production tunnel config, the runner registration, or the Cloudflare DNS (that's `robbohome-admin` territory).

## Recovery if the agent goes silent

Probable causes (in order of likelihood):

1. **CPU saturation from stuck delivery retries** — check `openclaw channels status` for "event loop degraded" reasons. Fix: `find ~/.openclaw/delivery-queue -maxdepth 1 -type f -delete` (preserve `failed/`), then `launchctl kickstart -k gui/$UID/ai.openclaw.gateway`.
2. **Channel allowlist missing entry** — see the three-layer section above.
3. **`tools.message` not on the agent's tools list** — agent processes the message but can't reply.
4. **Discord WebSocket stuck in reconnect loop** — `grep "Gateway websocket closed: 1006" /tmp/openclaw/openclaw-*.log`. Fix: gateway kickstart.

## Related skills + memories

- `deploy-gym` — Pattern A reference, same shape as this blog deploy
- `register-runner` — adding a new self-hosted runner for a new robbohome-* repo
- Memory: `reference_openclaw_discord_agent_setup` — the full three-layer playbook
- Memory: `feedback_openclaw_attribution` — Dan did NOT write OpenClaw
- Memory: `feedback_public_writing_no_business` — 3D printing is a hobby in public copy
