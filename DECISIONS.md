# Decision Log

## 2026-07-31 — GitHub is the coordination hub

**Decision:** Use `HydraCoreSystems/gm-commerce-hq` as the permanent project-management headquarters and source of truth for GM Commerce.

**Reason:** AI conversations and usage windows are temporary. GitHub provides persistent documentation, issues, branches, commits, and handoffs that multiple contributors can use.

## 2026-07-31 — Repository remains private

**Decision:** Keep the headquarters repository private.

**Reason:** It may contain internal business processes, architecture, future strategy, operational weaknesses, and integration details. Claude, Copilot, and ChatGPT can use authenticated GitHub access rather than requiring public visibility.

## 2026-07-31 — Minimal human intervention is a core requirement

**Decision:** GM Commerce will optimize not only for speed and inventory accuracy but also for the fewest responsible human touches.

**Reason:** Gathering Moss needs leverage comparable to a larger company without adding a large staff.

## 2026-07-31 — Skrybix remains separate and authoritative for plants

**Decision:** Plant identities remain in Skrybix. The Product SKU Generator handles non-plant products during the initial phase.

**Reason:** Skrybix already has Mother_ID and Cutting_ID generation. Replacing or duplicating it would delay useful delivery and introduce inventory risk.

## 2026-07-31 — Product SKU Generator local JSON is interim

**Decision:** Accept local JSON counters and history for the single-user starter utility, but do not treat them as the permanent central commerce database.

**Reason:** The approach is appropriately simple now, but GM Commerce will eventually need a durable shared source of truth for products, listing state, and marketplace identifiers.

## 2026-07-31 — Shopify comes before Etsy

**Decision:** Prove the shared listing-draft model and Shopify draft-publishing workflow before implementing Etsy publication.

**Reason:** This keeps the first end-to-end workflow smaller and avoids designing two marketplace integrations before the central model is validated.
