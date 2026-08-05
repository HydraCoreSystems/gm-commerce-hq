# Backlog

GitHub Issues are the executable work queue. This document holds ideas and work that are not yet ready to become active implementation issues.

## Version 1 Candidates

- Central product and listing schema.
- Product SKU Generator connection or migration path.
- SKU photo-folder readiness detection.
- AI-assisted product listing drafts.
- Owner review and approval workflow.
- Shopify draft publishing.
- Marketplace identifier storage.
- Etsy publishing after Shopify workflow validation.
- Sold-state and duplicate-listing protection.

## Version 2

- Marketing calendar and campaign assistance.
- Listing performance analysis.
- Pricing recommendations based on costs and market conditions.
- Automated stale-listing identification and refresh suggestions.
- Deeper GM Money integration.

## Flagged architectural risks (not yet actionable — no code changed)

- **GM Commerce's Product SKU Generator integration is local-filesystem-only,
  which conflicts with its own stated Vercel deployment target.**
  `lib/sources/sku-generator.ts` and `lib/photo-root.ts` (plus the photo-
  folder creation in `app/pipeline/actions.ts`) read/write the SKU
  Generator's local files (`data/sku-log.json`, `data/config.json`) and
  the local OneDrive photo root directly via `fs`. This works today
  because GM Commerce only runs on `localhost` on the same machine as the
  SKU Generator. The moment GM Commerce is deployed to Vercel (its own
  intended stack) this breaks completely — a Vercel serverless function
  has no access to a local Windows filesystem at all. Concretely: this
  also means nobody but Phil, at his own desktop, can ever select or
  process a non-plant product — Crystal couldn't do this from her phone
  the way she already does in GM Money, even after Version 1 ships.
  Skrybix's own pattern (a small authenticated HTTP endpoint,
  `sku-registry`) is the proven fix once this needs solving — the Product
  SKU Generator would need an equivalent tiny local API instead of GM
  Commerce reading its files directly. Not urgent while GM Commerce is
  local-only; becomes a real blocker the moment real deployment or
  phone-based use is wanted.
- **GM Commerce has no authentication at all.** Fine while it only runs on
  `localhost`; becomes a real exposure (a public URL with unauthenticated
  write access to real product data and local photo-folder creation) the
  moment it's deployed. Skrybix's shared-password-plus-signed-cookie
  pattern is the nearest proven precedent, though the eventual direction
  for this business is real per-person accounts (see `gm-money-webapp`'s
  `CLAUDE.md`), not another shared password.

## Phase F deferred

- **Photography recommendation service: migrate photo-set signal derivation to canonical photo claims when the GMCOM-011-to-canonical bridge lands.** Currently reads `photo_sets`/`photo_assets`/`photo_derivatives` legacy tables directly. Once product reset Phase B creates canonical photo claim predicates, the photography service's data-reading layer should consume those claims rather than the legacy tables. The recommendation value shape and §18 SelectionTrace contract remain unchanged — this is a data-source migration, not a contract change. Tracked in `DECISIONS.md` (2026-08-05).
- **Phase F Slice 6 (SEO) and Slice 7 (merchandising)** — planned in `phase-f-slice-plan.md`, not yet started. No briefs exist yet.

## Future

- More autonomous merchandising and marketing execution.
- Additional sales channels.
- Advanced forecasting and business intelligence.

## Explicitly Outside GM Commerce

- HydraCore.
- HydraCloud.
- General-purpose cultivation automation unrelated to selling Gathering Moss products.
