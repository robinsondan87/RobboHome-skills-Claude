---
name: geekythings-business
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

The clean four-desk model was created and smoke-tested on 5 September 2026.
Each agent uses GPT-5.6 with medium thinking, has its own Discord channel, and
shares the `geekythings` MCP backed by Product Manager structured state.

| Discord channel | Agent | Purpose |
|-------|-----------|---------|
| `#geekythings-product-manager` | `geekythings-products` | Catalogue, pricing, files, stock, production and marketplace alignment. Product writes require preview, exact approval and verification. |
| `#geekythings-ideas` | `geekythings-ideas` | Research and the idea → shortlist → prototype → product draft → ready → live pipeline. |
| `#geekythings-customer-service` | `geekythings-customer-service` | Private Etsy order/review reads and concise customer-reply drafts. Dan sends replies in Etsy because Open API v3 has no conversations API. |
| `#geekythings-social` | `geekythings-social` | Exact-media social drafts, approval and internal scheduling. Official Instagram and Facebook adapters are configured, but the global publishing switch remains off. |

The old `geekythings-content-agent` and `geekythings-trends-agent` names are
retired. Their Discord channels were retained as `zlegacy-*`; do not bind new
agents to them. The old Upload-Post social integration must not be revived: it
selected the wrong image at least once and did not verify live publication.

The current MCP exposes catalogue, idea, customer-draft, social-draft and
approval-gated product tools. Etsy listing, receipt and review reads are live
through the approved seller app with read-only scopes. Etsy conversations are
still unavailable, so customer replies remain drafts that Dan sends in Etsy.
Instagram and Facebook official adapters are configured but public publishing
remains globally disabled. Any controlled publishing phase must preserve exact
media/caption/time approval plus live URL verification.

## Working principle: OpenClaw first

All automation for GeekyThings must be designed around OpenClaw agents.
Do not recommend third-party SaaS tools (Nuelink, Outfy, Crosslist, etc.) unless there is a genuine capability gap that OpenClaw cannot fill.

## Product Manager app

The GeekyThings Product Catalogue app lives at https://geekythings.robbohome.com.
- Manages product listings, files (3MF), pricing, tags, colours, sizes
- API-driven — see `skills/geekythings-listings/SKILL.md` for workflow
- Deploy via `skills/deploy-geekythings/SKILL.md`

The scoped agent API is exposed through the local launcher
`~/.local/bin/geekythings-mcp`. The shared Discord/Codex MCP process itself runs
as a MetaMCP stdio child on `svr001`. The Etsy callback is
`https://geekythings.robbohome.com/api/etsy/oauth/callback`; connect it with
`~/Projects/GeekyThingsProductCatalogue/scripts/connect-etsy` only after the
API key and shared secret are stored in SOPS, then deploy the resulting token
file to the `svr001` MetaMCP runtime path described in the deploy skill.

Etsy inventory prices may be changed only through the dedicated MCP approval
flow. Preview first, show Dan the listing title, current and proposed Stand
prices and affected variant count, then wait for the exact `APPROVE ETSY ...`
phrase. Apply refuses stale inventory, preserves all non-price variant fields,
is idempotent, and verifies the final inventory from Etsy. The connector needs
`listings_w`; general agreement is never sufficient approval for a live write.

## Key contacts / accounts

- Etsy shop: GeekyThingsUK
- Primary email: 3dlabzuk@gmail.com
- GitHub: robinsondan87

## Related skills
- `skills/geekythings-listings/SKILL.md` — product listing workflow
- `skills/deploy-geekythings/SKILL.md` — deploying the product catalogue app
- `skills/bambu-integration/SKILL.md` — Bambu Studio integration findings (not possible from custom domain)
