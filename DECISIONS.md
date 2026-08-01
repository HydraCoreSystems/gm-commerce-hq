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

## 2026-08-01 — Existing SKUs enter GM Commerce from two source systems

**Decision:** GM Commerce never requires manual SKU re-entry. Plant records are selected in Skrybix; non-plant records are selected in Product SKU Generator. Each source sends the already-assigned SKU and source record into GM Commerce.

**Reason:** The SKU is the permanent business identity. Re-keying it would add labor and create avoidable inventory errors.

## 2026-08-01 — GM Commerce owns the downstream commerce workflow

**Decision:** After intake, GM Commerce owns photo-folder status, notes, AI processing state, listing drafts, review state, destination choices, marketplace relationships, and publishing state. Skrybix and Product SKU Generator remain authoritative only for their source identities and SKU creation.

**Reason:** This preserves clear source boundaries while giving the commerce workflow one durable home.

## 2026-08-01 — One canonical Listing Package supports every sales option

**Decision:** GM Commerce is marketplace-agnostic. AI creates one canonical Listing Package for a SKU. Output adapters then use that package for Shopify, Etsy, both platforms, copy-and-paste sales sheets for Palm Street or Overgrown Oasis, and future sales channels.

**Reason:** Listing intelligence should be created once rather than rewritten independently for each channel. Some channels have APIs; others require a human-friendly document or copy-ready view. Both are output targets from the same package.

## 2026-08-01 — Human review remains before external publication

**Decision:** AI-generated copy and prepared photos enter a review queue. Phil or Crystal chooses the destination channel or channels and approves the result before publication during Version 1.

**Reason:** The system should remove repetitive work while preserving owner control over public listings and sales decisions.

## 2026-08-01 — Every source system uses the same commerce handoff pattern

**Decision:** Every authoritative source system must expose the same conceptual handoff to GM Commerce: a human selects an existing record, the source persists that selection, the source exposes only selected records through an authenticated interface, and GM Commerce consumes the handoff without creating or modifying the source identity.

**Reason:** Skrybix and Product SKU Generator manage different product types, but GM Commerce should receive both through one predictable intake pattern. A common handoff reduces duplicated logic, prevents manual SKU re-entry, and keeps each source authoritative for its own identities.
