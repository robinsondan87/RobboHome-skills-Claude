---
name: geekythings-listings
description: Create or update GeekyThings product listings in the Product Manager app via API workflows, including tags/colours/sizes/readme/pricing/media, syncing marketplace listings, or moving drafts live.
---

# GeekyThings Listings

## Overview
Use this skill to create or update GeekyThings product listings in the Product Manager.
Prefer API calls over editing CSV/files directly, and keep folder conventions intact.

All API calls go to https://geekythings.robbohome.com (Cloudflare Zero Trust protected — must be authenticated).

## Workflow decision tree

1. Identify the product target.
   - Create a new product: `POST /api/add_product`
   - Update an existing product: look up by `category` + `product_folder` (use `/api/rows` to locate by SKU if needed)

2. Update listing content.
   - Tags/colours/sizes/readme: `POST /api/product_meta` (preferred) or `POST /api/readme` for README-only edits
   - Pricing: `POST /api/pricing`
   - Media/3MF: `POST /api/upload`, then use `/api/rename_file` or `/api/delete_file` as needed

3. Move the product state if required.
   - Draft → Live: `POST /api/approve`
   - Live → Archive: `POST /api/archive`
   - Live → Draft: `POST /api/move_to_draft`

4. For live Etsy variant SKUs, use the GeekyThings MCP approval boundary.
   - When every Etsy variant belongs to one catalogue product, call
     `preview_etsy_fixed_sku_update`. It assigns the same canonical SKU to all
     variants, turns SKU and quantity variation off, sets shared quantity to
     20, and keeps only variation properties that genuinely explain different
     prices. It also supports listings with no variations.
   - Call `preview_etsy_sku_update` with the listing id, one complete Etsy
     variation property, and Product Manager product ids for every value only
     when one Etsy listing genuinely represents multiple catalogue products.
   - Replay every resolved `SKU - Product title` mapping and the affected count.
   - Call `apply_approved_etsy_sku_update` only after Dan supplies the exact
     phrase from that preview in the current chat.
   - Treat success only as a fresh Etsy inventory read matching the approved
     hash. The apply tool refuses stale previews and is idempotent.
   - Etsy requires compatible `*_on_property` sets. If existing price or
     quantity variation spans every property, a SKU-only update may also need
     every property. To make price and SKU vary only by the mapping property,
     use `normalise_variation_settings=true`; `global_quantity` defaults to 20.
     The same approved payload then sets price/SKU to the mapping property and
     turns quantity/processing-profile variation off. Replay those setting and
     quantity changes explicitly because they are not SKU-only.
   - Quantity must never vary for print-on-demand products. Use one shared
     quantity, normally 20. Do not make price vary by colour when all colours
     have the same price. The tool detects and preserves genuine existing
     colour exceptions such as a cheaper `Random` option and includes them in
     the approval preview.
   - Etsy requires every retained `*_on_property` id to stay in the same order
     used by the inventory products. Never sort property ids numerically when
     constructing an inventory update; preserve Etsy's API order.
   - Etsy can regenerate internal variation `value_ids` after accepting an
     inventory PUT. Verification hashes must use the customer-visible property
     id/name/value, SKU, price, quantity and enabled state, not `value_ids`.
     Otherwise a successful multi-variant write is falsely reported as failed.

## Conventions to keep
- Keep product folders as `SKU - Product Title`; UI strips the SKU for display, but backend paths require the full folder name.
- Keep README as `README.md` under Draft/Live/Archived paths; the DB does not store README content unless passed to `/api/product_meta`.

## References (in GeekyThings project directory)
- `references/api.md` — full endpoint payloads and examples
- `references/listing-context.md` — marketplace rules and brand/social context

## Business context
- Sells on Etsy (GeekyThingsUK) and eBay
- Products: LEGO-compatible parts, personalised card holders, articulated fidget toys — all original designs
- Compliant with Etsy's June 2025 3D printing policy
- All automation via openclaw agents — no third-party SaaS tools
- See `skills/geekythings-business/SKILL.md` for full business context
