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
   - Agent-driven pricing changes use `preview_product_pricing_update` with the
     complete base and size-pricing object. Replay every size, cost, sale price
     and postage value, then use `apply_approved_action` only after Dan supplies
     the exact `APPROVE PRODUCT PRICING ...` phrase. The write refuses stale
     pricing, is atomic, verifies the saved values and sets `Completed: No`.
   - Catalogue-name corrections use `preview_product_rename` with the product id
     and title without its SKU prefix. Replay the exact old and new folder names,
     then use `apply_approved_action` only after Dan supplies the exact
     `APPROVE PRODUCT RENAME ...` phrase. The guarded rename preserves the SKU,
     live/draft state, files, pricing and marketplace links; updates the README
     heading and linked catalogue labels; and sets `Completed: No`.
   - A Product Manager rename never changes the customer-facing marketplace
     titles or descriptions.
   - To fill an empty catalogue media folder from an already linked Etsy
     listing, call `backfill_product_media_from_etsy`. It imports at most ten
     full-size images in Etsy rank order, never alters the marketplace, refuses
     Archived products and refuses to overwrite or append when the product
     already has images. A successful import verifies every stored image and
     sets `Completed: No` for Dan's Catalogue Review. If several active Etsy
     listings are linked, supply the exact listing id; never choose by title.

3. Move the product state if required.
   - Draft → Live: `POST /api/approve`
   - Live → Archive: `POST /api/archive`
   - Live → Draft: `POST /api/move_to_draft`
   - Draft → Personal / non-business: `POST /api/move_to_personal`
   - Personal or Archived → Draft: `POST /api/move_to_draft`
   - Personal is a lifecycle status, not a product category. Its files live at
     `Products/Categories/_Personal/<Category>/...`, preserving the original
     category and SKU while excluding the product from marketplace parity and
     the default Catalogue Review.
   - The Personal move refuses any product with an active structured
     marketplace listing. End or remove the marketplace listing first; never
     hide an actively sold product by classifying it as personal.
   - During Draft review, use the built-in Move to Personal or Archive actions;
     a successful classification advances directly to the next Draft item.

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

5. To create a new Etsy listing from a catalogue product, use the approval-gated
   private-draft workflow.
   - Call `preview_etsy_listing_draft` with the Product Manager product id and
     the complete proposed title, description, price, taxonomy, shipping and
     processing profile, tags, materials, colours, and exact image filenames.
   - Replay the exact `SKU - Product title`, public title and description,
     price, shared quantity, colour choices, tags, materials, and image list.
   - Call `apply_approved_etsy_listing_draft` only after Dan supplies the exact
     `APPROVE ETSY DRAFT ...` phrase from that preview in the current chat.
   - This flow creates a private Etsy draft and uploads only the SHA-bound
     approved Product Manager images. It never publishes the listing.
   - The write is resumable and idempotent. Treat success only as a fresh Etsy
     read verifying draft state, ownership/copy, images, fixed canonical SKU,
     shared quantity 20, and the approved inventory structure.
   - After verification it links the Etsy URL and price back to Product Manager
     and sets `Completed: No`, placing the touched product in Catalogue Review.
   - For colour choices, price and SKU do not vary by colour unless the preview
     explicitly captures a genuine existing price exception. Quantity never
     varies; the default shared quantity is 20.

5a. To update customer-facing copy on an existing live Etsy listing, use the
    approval-gated copy workflow.
   - Call `preview_etsy_listing_copy_update` with the listing id and any proposed
     title, description or complete tag list. Replay the exact complete current
     and proposed title, description and tags.
   - Customer-facing copy uses “designed and made in the UK” and must not mention
     3D printing. Remove legacy `3d printed` tags at the same time.
   - Call `apply_approved_etsy_listing_copy_update` only after Dan supplies the
     exact `APPROVE ETSY COPY ...` phrase in the current chat.
   - The approval is bound to hashes of all three copy fields. The apply refuses
     stale copy, is idempotent, re-reads Etsy for exact verification and returns
     the linked catalogue product to Catalogue Review.
   - This boundary cannot change price, inventory, variations, images, shipping
     or processing.

6. For live eBay SKU and stock corrections, use the separate approval-gated
   eBay workflow.
   - Call `preview_ebay_sku_update` with the exact eBay listing id, Product
     Manager product id, and available quantity (normally 20). It never writes.
     For a fixed-price listing it may also include one exact `price`, allowing
     Etsy price parity and stock restoration in the same approval. A single
     replacement price is forbidden for multi-variation listings.
   - Replay the resolved `SKU - Product title`, listing title/id, current and
     proposed SKU values, prices, sold counts, and available quantities.
   - For a fixed-price listing, use the canonical Product Manager SKU. For a
     multi-variation listing, every variation SKU must be unique: use the
     canonical SKU as a prefix plus deterministic variation values and a short
     collision-safe suffix.
   - eBay exposes lifetime total quantity on reads but accepts available stock
     on revisions. Set 20 available per active listing or variation for
     print-on-demand products and verify using total minus sold.
   - When lowering a fixed Buy It Now price, legacy Best Offer auto-accept or
     auto-decline thresholds can make eBay reject the revision. Price-changing
     revisions explicitly delete those two optional thresholds while leaving
     Best Offer itself enabled, then perform the normal fresh verification.
   - Call `apply_approved_ebay_sku_update` only after replaying the current and
     proposed price whenever price is included, and Dan supplies the exact
     `APPROVE EBAY INVENTORY ...` phrase from that preview in the current chat.
     The tool refuses stale previews, is idempotent, re-reads eBay, links a
     blank catalogue eBay URL, records any variant SKU aliases, and sets
     `Completed: No` for Catalogue Review.
   - Retiring a duplicate listing is a separate destructive boundary. First
     call `preview_ebay_end_listing`, replay the exact listing and sales data,
     and call `apply_approved_ebay_end_listing` only after Dan supplies its
     exact `APPROVE EBAY END ...` phrase. Never infer retirement approval from
     an inventory approval.

## Conventions to keep
- Keep product folders as `SKU - Product Title`; UI strips the SKU for display, but backend paths require the full folder name.
- Keep README as `README.md` under Draft/Live/Archived paths; the DB does not store README content unless passed to `/api/product_meta`.
- For a new multi-variation eBay listing, do not repeat a variation property such
  as `Colour` in its item specifics. eBay rejects that shape. Keep choice
  properties in variations and shared attributes, including finished dimensions,
  in item specifics. The preview must refuse duplicates before approval.
- A seller SKU and an EAN are different fields. When an eBay category requires a
  product identifier for a genuinely unbarcoded handmade item, set the exact EAN
  fallback `Does not apply` on every variation. Never invent an EAN or use the
  fallback when a manufacturer identifier exists. Keep the canonical Product
  Manager SKU as the basis of each unique deterministic variation SKU.
- Before issuing an `APPROVE EBAY LISTING ...` phrase, the preview must pass
  eBay's `VerifyAddFixedPriceItem` call. This catches category-specific required
  fields and invalid values without creating a listing. Never ask Dan to approve
  a locally validated payload that eBay has not accepted in verification.
- Before creating an eBay preview, resolve the catalogue product's existing
  structured `marketplace_listings` and `marketplace_exclusions`. If an active
  listing exists, update it instead; never create a second listing merely
  because a marketplace comparison failed to recognise the existing mapping.
  If eBay is excluded, omit the product from parity gaps and do not create a
  preview unless Dan first explicitly removes the exclusion. Use
  `set_marketplace_exclusion` for reversible catalogue exclusions; the reason
  is visible on the product page and setting one cancels pending create previews.
- Marketplace links are many-to-many. One catalogue product may have several
  listings, and one shared Etsy listing may represent several catalogue products
  through its variants. Persist the structured link for every resolved product;
  never move a shared listing from one product to another.
- To clean catalogue state after a parity audit, use
  `preview_unlisted_products_move_to_draft`. It snapshots every currently Live
  product without an active structured marketplace link and preflights all
  folders. Replay the full count and warning, then wait for the exact
  `APPROVE PRODUCT DRAFT ...` phrase before applying it. This changes Product
  Manager folders/status only and never Etsy or eBay.
- Approval previews are immutable snapshots, not background jobs. A pending
  preview cannot execute without its exact phrase. After reviewing old
  previews, mark superseded, abandoned and expired records Cancelled rather
  than deleting them so the audit history remains intact. Never reuse a stale
  marketplace preview; create a fresh hash-bound preview instead.
- Match the Etsy listing's UK postage when creating an eBay listing. Use the
  listing-level £1.59/£0.49 override only where Etsy charges it; retain free
  postage when Etsy is free.

## References (in GeekyThings project directory)
- `references/api.md` — full endpoint payloads and examples
- `references/listing-context.md` — marketplace rules and brand/social context

## Business context
- Sells on Etsy (GeekyThingsUK) and eBay
- Products: LEGO-compatible parts, personalised card holders, articulated fidget toys — all original designs
- Compliant with Etsy's June 2025 3D printing policy
- All automation via openclaw agents — no third-party SaaS tools
- See `skills/geekythings-business/SKILL.md` for full business context
