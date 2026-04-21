---
description: GeekyThings 3D printing business context — Etsy/eBay, openclaw automation agents, and working principles.
---

# GeekyThings Business Context

Dan runs **GeekyThings** (geekythings.co.uk) — a 3D printing business.

## Business overview

- **Products:** LEGO-compatible parts, personalised card holders, articulated fidget toys — all original designs
- **Marketplaces:** Etsy (GeekyThingsUK) and eBay
- **Policy:** Compliant with Etsy's June 2025 3D printing policy (original designs only)
- **Goal:** Automate social media posting and keep Etsy/eBay listings aligned — all in-house via OpenClaw

## OpenClaw automation agents

| Agent | Workspace | Purpose |
|-------|-----------|---------|
| `geekythings-content-agent` | `~/.openclaw/workspaces/geekythings-content` | Social media content creation and posting |
| `geekythings-trends-agent` | `~/.openclaw/workspaces/geekythings-trends` | Trend research and product opportunity identification |

Both workspaces are created but not yet fully configured (as of April 2026).

## Working principle: OpenClaw first

All automation for GeekyThings must be designed around OpenClaw agents.
Do not recommend third-party SaaS tools (Nuelink, Outfy, Crosslist, etc.) unless there is a genuine capability gap that OpenClaw cannot fill.

## Product Manager app

The GeekyThings Product Catalogue app lives at https://geekythings.robbohome.com.
- Manages product listings, files (3MF), pricing, tags, colours, sizes
- API-driven — see `skills/geekythings-listings/SKILL.md` for workflow
- Deploy via `skills/deploy-geekythings/SKILL.md`

## Key contacts / accounts

- Etsy shop: GeekyThingsUK
- Primary email: 3dlabzuk@gmail.com
- GitHub: robinsondan87

## Related skills
- `skills/geekythings-listings/SKILL.md` — product listing workflow
- `skills/deploy-geekythings/SKILL.md` — deploying the product catalogue app
- `skills/bambu-integration/SKILL.md` — Bambu Studio integration findings (not possible from custom domain)
