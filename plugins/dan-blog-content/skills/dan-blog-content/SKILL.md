---
description: Reference for Dan's personal site — Production Shaped at productionshaped.com (editorial publication brand). Covers the admin UI, the JSON ideas API, and the dan-blog-content OpenClaw agent. Use when helping Dan with anything blog-shaped (capture, draft, publish, voice, agent troubleshooting).
---

# Dan's blog — admin, API, and agent

A small companion reference so future-Claude knows the blog stack exists without Dan re-explaining.

## Brand state

The site is **Production Shaped** — *"Field notes on running things like production — at work, at home, and in the gap between them."* The brand replaced the original **RobboHome** homelab framing; the editorial publication treatment (masthead with rules, italic-serif strapline, monospace meta line, colophon footer, drop-caps on essays) is the live look.

The `productionshaped.com` domain is acquired and live (INFRA-28 closed 2026-05-19). The Cloudflare zone is `d3a29862bd5a814c9ce1d7957df8f0aa`; apex + `www` are both CNAME-flattened, proxied, and route through the existing Cloudflare tunnel to `svr002:3003`. The legacy `blog.robbohome.com` hostname was fully retired (tunnel ingress + DNS removed 2026-05-20) — Dan only had two external links pointing at it and updated them by hand. There are no host-level redirects; the only hostname serving the site is `productionshaped.com` (with `www.productionshaped.com` as an apex alias that's not 301'd).

The 2026-05 brand-variant experiment (parallel `productionshaped` + `direction-a` + `direction-b` branches, three extra subdomains, a "Working notebook" variant) is fully retired — branches deleted, tunnel ingress + DNS removed, GHCR tags cleaned up, staging containers destroyed.

## Site

| Item | Value |
|---|---|
| Canonical URL | https://productionshaped.com |
| Apex alias | https://www.productionshaped.com (serves the same content; not 301'd) |
| Branch | `main` (only branch) |
| Container on svr002 | `robbohome-blog` (Docker, port 3003) |
| Data volume | `/home/robbohomebot/data/robbohome-blog/data` (SQLite admin DB) |
| Repo | `robinsondan87/robbohome-blog` (private) |
| Project path | `/Users/robbohomebot/Projects/openclaw-dan-blog` |
| Stack | Astro 5, Node standalone adapter, nginx-removed in v0.7.0 |
| Deploy | Pattern A on a self-hosted GitHub Actions runner — `cat VERSION` drives the image tag, deploy step writes `.env`, runs `docker compose pull && up -d` |

**Cloudflare zones in use**: `productionshaped.com` (`d3a29862bd5a814c9ce1d7957df8f0aa`) — the only zone holding ingress for the blog. The `robbohome.com` zone (`93e554d66c0ed530fbd1387ce14a62a5`) is still in the account for other services but no longer carries any blog hostname.

**Agent endpoint**: the OpenClaw `dan-blog-content` agent and any reflective-capture session post to `https://productionshaped.com/api/ideas` with the bearer token at `~/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt`. (Repointed from the retired `blog.robbohome.com` hostname on 2026-05-20.)

## Routes

**Public, prerendered:**
- `/` — home (hero bullets + Lately stream + Pinned + Projects preview)
- `/writing` — combined chronological stream of essays + notes
- `/blog/<slug>` — individual essay
- `/blog/tag/<tag>` — tag index
- `/notes/<slug>` — individual short-form note
- `/log`, `/projects`, `/projects/<slug>`, `/about`, `/now`, `/contact`, `/404`, `/rss.xml`, `/sitemap-index.xml`
- `/og/default.png`, `/og/blog/<slug>.png` — OG card images (build-time)

**Permanent redirects** (set in `astro.config.mjs`):
- `/blog` → `/writing` (the old essay index)
- `/notes` → `/writing` (the old notes index)
- `/start-here` → `/` (the home page already does the start-here job)

`/blog/<slug>` and `/notes/<slug>` slug pages still resolve directly — only the bare index URLs redirect.

**Header nav** (5 items + theme toggle, deliberately KISS):

```
writing · projects · about · log · now
```

**SSR (require auth):**
- `/admin/login` — password form (open)
- `/admin/ideas` — capture + list + state transitions + inline draft editor with image drop zone + publish-to-blog/notes dialog (cookie-gated)
- `/admin/posts` — directory of every file in `src/content/blog/` and `src/content/notes/`, pulled live via the GitHub Contents API
- `/admin/posts/<collection>/<slug>` — edit page for an existing post: full-file textarea (frontmatter + body), image drop zone, Save (commits with if-match SHA → 409 on concurrent edits), Unpublish (toggles `draft: true`), Delete (git rm)
- `/admin/logout` — POST clears cookie
- `/api/health` — bearer-gated ping
- `/api/ideas[?state=]` — list / create
- `/api/ideas/:id` — read / patch / delete
- `/api/ideas/:id/publish` — assemble frontmatter + draft body, commit to `src/content/<collection>/<slug>.md`, mark idea done
- `/api/uploads` — multipart image upload, commits via the GitHub Contents API. **Accepts cookie OR bearer**. Pass either `idea_id` (draft context, assets → `src/assets/images/<id>/`) OR `post_collection` + `post_slug` (existing-post context, assets → `src/assets/images/posts/<slug>/`). Other `/api/*` routes stay bearer-only.

### Mobile-friendly image upload

Both the draft editor on `/admin/ideas` and the post editor on `/admin/posts/<collection>/<slug>` have a tap-to-upload drop zone (label: *"Tap to choose, drop, or paste an image"*). A hidden `<input type="file" accept="image/*" multiple>` is wired so the same widget handles:

- **Tap/click** → opens iOS Photos picker / system file dialog
- **Drag-and-drop** → desktop drag from Finder/Explorer
- **Clipboard paste** → desktop screenshots/copies into the textarea
- **Enter / Space** keypress on the focused zone — keyboard a11y

Inserts `![](relativePath)` at the cursor on success (alt is empty by default — type it in afterwards).

## Auth

| Surface | Mechanism | Where to find |
|---|---|---|
| `/admin/*` UI | `APP_PASSWORD` env var + HMAC-signed cookie (`AUTH_SECRET`) | GitHub secret on the repo; stash in 1Password |
| `/api/*` JSON | `Authorization: Bearer $ADMIN_API_TOKEN` | GitHub secret on the repo |
| `/api/uploads` | Cookie OR bearer (special-cased in `src/middleware.ts`) | Same as above |
| Agent's API token | Plain text file (mode 0600) | `/Users/robbohomebot/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt` |
| GitHub Contents API (commits from `/api/uploads`) | `BLOG_REPO_PAT` env var — fine-grained PAT, `contents:write` on `robinsondan87/robbohome-blog` only | GitHub secret on the repo (name: `BLOG_REPO_PAT`) |

`security.checkOrigin` is disabled in `astro.config.mjs` — the HMAC cookie does the CSRF job.

## Ideas API quick reference

```bash
TOKEN=$(cat ~/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt)
BASE=https://productionshaped.com/api

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

## Image placeholders in drafts

Every draft that needs images uses an HTML comment block per image plus a deliberately-broken placeholder line. The comment preserves the prompt + alt for future regeneration; the broken `![]()` makes it impossible to publish accidentally with images missing.

```markdown
<!--
IMAGE: hero
ASPECT: 16:9
ALT: A short sentence describing the image for screen readers.
PROMPT: The full image-generator prompt, anything until the closing arrow.
-->
![TODO — replace JUST this line with the uploaded image; leave the comment above as a record](TODO-IMAGE-hero)
```

Rules when drafting:

- One placeholder per image, in the position it'll occupy in the rendered post.
- `IMAGE:` id is unique within the post — `hero`, `inline-1`, `inline-2`, etc.
- `ASPECT:` follows the visual rhythm: `16:9` for the hero, `4:3` or `1:1` for inline sections.
- `ALT:` is real alt text, not placeholder text. Screen readers will read it.
- `PROMPT:` is tuned to match the newsprint aesthetic — see *Visual / editorial conventions* below for the palette/style cues.

When Dan uploads the real image via the admin drop zone, **only the `![TODO ...](TODO-IMAGE-<id>)` line gets replaced** with the inserted `![<alt>](relativePath)`. The comment block stays above it — it's the receipt of how the image was generated and lets future-Dan (or a different image model) regenerate without losing context.

Don't generate the image yourself unless Dan asks. Hand him the prompt in the comment, leave the placeholder broken, and let him drive the image-gen tool.

## What NOT to put in a draft

These are easy to add as helpful "future-Dan" notes but actively cause damage at publish time:

- **No leading `---...---` "Suggested frontmatter" preamble.** The publish modal on `/admin/ideas` now supplies real frontmatter from the form. A preamble bordered by `---` markers in the draft body renders as a stray `<hr>` plus a code block on the live page. `publishDraft()` strips it defensively, but don't write it in the first place — keep the draft as pure post content.
- **No `pubDate` or other frontmatter keys inline in the body.** Same reason — the form supplies these.
- **No "TL;DR for Dan" / "Voice notes:" lines mixed into the body.** Put guidance in the idea's `notes` field instead. The body should be the published copy verbatim.

The Bambu URL-schemes post was the canary for this — it shipped with a visible preamble before the strip + skill rules landed.

## Prompt for reflective capture sessions

Use this when Dan asks a long-running session or agent to "look back at our work together and propose blog ideas". Paste the block below into the session verbatim — it's self-contained, embeds the voice rules, and tells the session to be conservative (capture zero or one idea per session, three is suspicious).

The audience for the prompt is a different session, not future-you. Don't trim it on the assumption the reader knows context — they don't.

```
Goal: capture blog-idea drafts based on YOUR experience working with Dan in
this session (and any earlier session context you have). The inbox at
https://productionshaped.com/admin/ideas holds Dan's existing curated ideas;
you're adding only the noteworthy moments from your shared work.

If you have the `dan-blog-content` skill installed, load it now — it has the
full API reference and voice rules. Then return here for the source-material
guidance, which is specific to this task.

The blog you're contributing to
- Publication name: "Production Shaped" (migrating from RobboHome).
- Strapline: field notes on running things like production — at work, at home,
  and in the gap between them.
- The genre: every post traces a discipline from work-production to home-
  production and reports honestly on the friction. Not how-to-do-SRE.
  What travels and what doesn't.

Source material — what to look at
- This conversation: what did Dan ask, what did you build, what broke, what
  was the friction, what surprised either of you.
- Patterns ACROSS the work, not single tasks: three slips of the same kind of
  bug; a scoping decision Dan made twice; a tool that only works because Dan
  treats his home like production; a tool that doesn't because he can't.
- Things Dan said in passing that sounded like a one-liner waiting for a
  600-word post around them.

The work/home gap is the brand
The strongest captures point at the tension where production-thinking from
work-Dan meets the four humans, no rota, no SLA realities of home-Dan. If
a candidate idea doesn't have that moment, it's probably not for this blog.

Filter — almost nothing you did together belongs on the blog
The hard test: would a future reader who doesn't know Dan find this honest,
specific, and slightly painful in a useful way?

Do NOT capture:
- "We built feature X." Dan isn't running a changelog.
- "Look how productive we were." He isn't running a marketing blog.
- Tutorials that just describe what got built. The how is in the code.
- Anything without a moment of friction, walked-back decision, or surprise.

DO consider capturing:
- A bug that taught a real lesson (e.g. a CSS-variable typo that broke a
  modal; a draft preamble that escaped into a live post).
- A scoping decision Dan made twice in different contexts.
- A point where doing it the production way at home felt absurd, AND doing
  it the home way at home felt sloppy. The tension itself is the post.
- A meta observation about the pacing — vertical slices that each shipped,
  the discipline of stopping a slice when you could keep going.
- An honest "the LLM in the loop did X well / fluffed Y" reflection — but
  only if specific. Generic "AI is helpful" doesn't earn its place.

The hook test
Before you POST anything: write the one-sentence hook out loud. If it sounds
like a press release or a tutorial intro, don't ship. If it sounds like
something Dan would say to a friend at a pub, you've got one.

API (skip this section if the skill is loaded)
- POST https://productionshaped.com/api/ideas
- Bearer token: /Users/robbohomebot/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt
- Headers:
    Authorization: Bearer <token>
    Content-Type: application/json
    User-Agent: curl/8.4.0          # Python urllib default UA is 1010-blocked
- Body: { "title": "≤200 chars, imperative, sentence-case", "notes": "<brief>" }
- GET /api/ideas first; dedupe by case-insensitive title match.

What to put in `notes`
- Why (the hook + the tension that earns it)
- Where it'd live: /blog (essay) or /notes (paragraph + link)
- 1–3 suggested tags from: sre-at-home, agents, homelab, observability,
  walked-back, postmortem, runbook, reflection, field-notes, vertical-slice,
  craft, 3d-printing, home-assistant
- Rough length (300–500 for /notes, 800–1500 for /blog)
- Voice guardrails specific to this idea (e.g. "don't name family",
  "OpenClaw is adopted not built", "don't claim 3D printing is a business")
- Image placeholders (per the skill's convention) if the piece needs them

Voice rules (always)
1. OpenClaw is NOT Dan's — adopted from openclaw.ai. Frame as "the runtime
   I run my fleet on", never "the runtime I built."
2. 3D printing / GeekyThings is a HOBBY in public copy — never "side
   business", "marketplaces", "customers", "selling". Bet365 employment
   terms.
3. No identifying child / school / clinic / address detail. The son's
   existence is acknowledged; specifics are not.
4. No Bet365 internal specifics. Day job: "Site Reliability Tech Lead at
   a large UK tech company."
5. British spelling. First-person. No marketing voice — no "elevate",
   "leverage", "unlock", no exclamation marks, no "drop a comment".
6. Don't claim authorship of anything Dan only runs/configured (Claude
   Code, OpenClaw, Home Assistant, New Relic — all someone else's work).

Stopping rule
Capture zero or one idea per session. Two is unusual. Three is suspicious —
it usually means you're capturing process not story. Re-read your shortlist
and cut. Better to skip than dilute the inbox.

Report back
- "OK <id> — <title>" per capture
- "SKIP (<one-line reason>) — <would-have-been-title>" per skip
- One honest closing sentence on why you skipped what you skipped, OR why
  the session genuinely had nothing worth capturing (that's the right
  answer most of the time).
```

## Common operations

| Task | How |
|---|---|
| Capture an idea from Discord | Post in `#dan-blog`, agent calls `POST /api/ideas` |
| Capture from anywhere | Curl `POST $BASE/ideas` with bearer token |
| See current inbox | https://productionshaped.com/admin/ideas or `GET /api/ideas?state=idea` |
| Develop an idea into a draft | Ask agent in `#dan-blog`; agent PATCHes `draft_markdown` + state=drafting. Include image placeholders for every image the post needs (see *Image placeholders in drafts* above). |
| Read or edit a draft | Admin UI: drafting-state ideas render an inline editor with a drop zone for images. Save with the form's Save button. Non-drafting states still show the read-only `<details>` panel. Or `GET /api/ideas/:id`. |
| Add images to a draft | Open the draft in `/admin/ideas`, drop or paste the image into the editor. Uploads to `src/assets/images/<idea-id>/<filename>` via `POST /api/uploads`, inserts `![](relativePath)` at the cursor. Replace only the `![TODO ...](TODO-IMAGE-<id>)` placeholder line; leave the comment block above as a record. Fill in `ALT:` from the comment into the `![alt](...)` brackets before publishing. |
| Publish a post | `/admin/ideas` → click "publish…" on a drafting-state idea, pick collection + frontmatter, hit Publish. One commit on `main`, Pattern A redeploys. The idea moves to Done with the live URL on the row. |
| Edit an existing post | `/admin/posts` → click the slug → edit the file in the textarea. Save commits with sha conflict detection — concurrent local edits return 409 with a "reload" message instead of silently overwriting. |
| Add images to a post (live or draft) | Drop or paste into the editor's drop zone. Drafts use `idea_id`; published posts use `post_collection` + `post_slug`. Assets land under the right path, markdown `![]()` reference inserted at the cursor. |
| Unpublish a post | On the edit page, hit "Unpublish (set draft: true)". Toggles `draft:` in frontmatter; the file stays in git but stops rendering on the public site. |
| Delete a post | On the edit page, hit "Delete (git rm)" — confirm modal, then a commit on `main` removes the file. |
| Rotate `APP_PASSWORD` | `gh secret set APP_PASSWORD --repo robinsondan87/robbohome-blog --body '<new>'` then trigger workflow |
| Rotate `ADMIN_API_TOKEN` | Same as above, but also update `~/.openclaw/workspace/agents/dan-blog-content/state/api-token.txt` (it's a local copy, not a SecretRef) |
| Invalidate all admin sessions | Rotate `AUTH_SECRET` instead (HMAC cookie verification fails) |
| Restart the gateway | `launchctl kickstart -k gui/$UID/ai.openclaw.gateway` then wait ~30–60s for both channels `connected` |

## Visual / editorial conventions

- **Newsprint palette**: cream `#f4f1e4` / black `#0a0908` / no chrome accent. Warm ink amber (`#8a6d2b`) used only in pen-and-ink illustrations and log bullets. CSS vars: `--bg`, `--fg`, `--ink-accent`, `--code-bg`, `--rule`, `--rule-strong`, `--muted`, `--faint`.
- **Type system**: Source Serif 4 body, JetBrains Mono headings + wordmark + meta, Inter nav.
- **Code blocks**: Shiki `css-variables` theme driven by `--code-bg` + `--fg` + `--ink-accent` so blocks sit on cream, not slammed white-on-cream from `github-light`.
- **Reading-time meta line**: `<date> · <X min read> · #tag #tag` rendered uppercase via `.meta { text-transform: uppercase }`.
- **Drop cap**: `<article class="has-dropcap">` is applied by default in `PostLayout.astro` for every essay.
- **`.visually-hidden`** screen-reader-only utility for pages where the masthead carries the visible page title but a semantic h1 is still wanted in the DOM (used on the home page).
- **Illustrations**: hand-drawn pen-and-ink, single warm-amber line art on cream, New Yorker spot-illustration style. See *Image placeholders in drafts* for the prompt backbone. The original SVG schematics (`HomelabSketch`, `AgentFleet`) and the `heroIllustration` frontmatter enum are **removed** — every hero is a raster illustration in `src/assets/images/posts/<slug>/`.
- **Soft-edge raster filter**: `article figure img, article > p img, .note-body img` get two `var(--bg)`-coloured drop-shadow halos so dark-edged illustrations feather into the cream page, plus a faint dark elevation shadow.

**Editorial masthead** (`.editorial-mast` in `Header.astro`): double rule top, centred wordmark in JetBrains Mono caps (no DR mark — that lives on the favicon only), italic-serif strapline below, monospace meta line (`VOL. 1 · <MONTH YEAR> · SHEFFIELD`), single rule below, then the 5-item nav. Home hero has no visible h1/subhead (the masthead does it) — a `.visually-hidden` h1 keeps the page semantic; the three pillar bullets lead the visible content.

**Section headings** (`.editorial-mast ~ main .section-heading`): strong rule above + faint rule below, centred — the print-newspaper section-divider treatment.

**Colophon footer** (`.editorial-colophon` in `Footer.astro`): typography credit, publication line (italic `Production Shaped` published by Dan Robinson from Sheffield, since 2026), link row (RSS · Contact · GitHub · LinkedIn), © line.

## What I should NOT do without Dan's say-so

- Publish an idea or edit an existing post **without explicit "yes, ship it"** from Dan. The endpoints exist (`POST /api/ideas/:id/publish` and the `/admin/posts/<collection>/<slug>` save/delete flow) and the agent has a bearer token that can call them — but each is a real commit on `main` that triggers Pattern A. Wait for Dan to drive the publish/edit himself in the admin UI, unless he explicitly delegates a specific action.
- Delete a post. Even with sha-conflict detection, a `git rm` commit is destructive — confirm in chat before doing it.
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
