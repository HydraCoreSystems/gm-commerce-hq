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

## 2026-08-01 — GM Commerce is a selection-driven pipeline, not a bulk importer

**Decision:** GM Commerce's `products` table only ever contains SKUs a
human has explicitly selected from Skrybix or the Product SKU Generator.
There is no bulk-import path and no screen anywhere that accepts a
manually-typed SKU.

**Reason:** The original GMCOM-002 issue described a read-only display
seeded by an import script; ChatGPT revised this mid-implementation
specifically to guarantee "GM Commerce must never require manual SKU
entry," and to model the real workflow (select → intake → review →
publish) rather than a passive mirror of both source systems' full
inventories. (Same principle as "Existing SKUs enter GM Commerce from two
source systems" above, recorded independently the same day.)

## 2026-08-01 — GM Commerce owns the downstream commerce workflow

**Decision:** After intake, GM Commerce owns photo-folder status, notes, AI processing state, listing drafts, review state, destination choices, marketplace relationships, and publishing state. Skrybix and Product SKU Generator remain authoritative only for their source identities and SKU creation.

**Reason:** This preserves clear source boundaries while giving the commerce workflow one durable home.

## 2026-08-01 — GM Commerce's Supabase project is a separate account

**Decision:** GM Commerce's database lives in a new Supabase account
(project ref `wcrcllhvgbhykbonopzx`) created under a co-owner's email, not
Phil's existing personal Supabase account that hosts Skrybix.

**Reason:** Phil's explicit choice, to keep GM Commerce's usage/billing
separate from his personal account.

## 2026-08-01 — GM Commerce creates its own photo folders for Skrybix-origin products

**Decision:** For plant products (which Skrybix doesn't organize into
photo folders at all), GM Commerce creates a folder named by SKU under the
same shared photo root the Product SKU Generator already writes into —
not a second, GM-Commerce-only photo root.

**Reason:** Avoids two parallel photo-folder locations for what should be
one conceptual "all product photos" root; the root is overridable via
`GM_COMMERCE_PHOTO_ROOT` if that ever needs to change.

## 2026-08-01 — One canonical Listing Package supports every sales option

**Decision:** GM Commerce is marketplace-agnostic. AI creates one canonical Listing Package for a SKU. Output adapters then use that package for Shopify, Etsy, both platforms, copy-and-paste sales sheets for Palm Street or Overgrown Oasis, and future sales channels.

**Reason:** Listing intelligence should be created once rather than rewritten independently for each channel. Some channels have APIs; others require a human-friendly document or copy-ready view. Both are output targets from the same package.

## 2026-08-01 — Human review remains before external publication

**Decision:** AI-generated copy and prepared photos enter a review queue. Phil or Crystal chooses the destination channel or channels and approves the result before publication during Version 1.

**Reason:** The system should remove repetitive work while preserving owner control over public listings and sales decisions.

## 2026-08-01 — Every source system uses the same commerce handoff pattern

**Decision:** Every authoritative source system must expose the same conceptual handoff to GM Commerce: a human selects an existing record, the source persists that selection, the source exposes only selected records through an authenticated interface, and GM Commerce consumes the handoff without creating or modifying the source identity.

**Reason:** Skrybix and Product SKU Generator manage different product types, but GM Commerce should receive both through one predictable intake pattern. A common handoff reduces duplicated logic, prevents manual SKU re-entry, and keeps each source authoritative for its own identities.

## 2026-08-01 — The interface must teach the workflow

**Decision:** GM Commerce must have a best-in-class, visually polished GUI that makes the next correct action obvious without requiring technical knowledge, written instructions, or institutional memory. The interface must be usable by Phil, Crystal, and future employees with minimal training.

**Reason:** GM Commerce is intended to become an operational system used repeatedly by different people. Its value depends not only on automation, but on reducing uncertainty, errors, and training time. The UI should communicate status, readiness, blockers, and the next action through layout, hierarchy, language, and feedback rather than expecting the user to understand the underlying architecture.

## 2026-08-02 — AI generation is reserved atomically before any paid call

**Decision:** `triggerListingGeneration` must atomically reserve a product
(a `generating` status, claimed via a Postgres function using `SELECT ...
FOR UPDATE`) before either paid provider call in the Listing Quality
Engine's pipeline is made. Reservation, not a post-hoc status update, is
the mechanism that prevents duplicate paid calls under concurrent Generate
clicks.

**Reason:** GMCOM-009's explicit acceptance criterion. The prior
implementation only flipped `products.status` at the very end of
generation, so two concurrent calls (a double-click, two tabs) could both
pass the eligibility check and both spend real provider calls before
either write landed.

## 2026-08-02 — Every regeneration is versioned, never silently overwritten

**Decision:** Any time a generation succeeds and a `listing_packages` row
already exists for that SKU, the row being replaced is archived into an
append-only `listing_package_versions` table (including its own
`source_facts` and `quality_summary`) inside the same transaction as the
new write. Application code never deletes or overwrites version history.

**Reason:** GMCOM-009's explicit requirement ("Add durable package
versioning to preserve prior versions during regeneration"). Regeneration
is real now (see the fix below) — losing the previous version silently on
regenerate would be a real data-loss risk once that path is actually used.

## 2026-08-02 — Regeneration requires the product to have reached review and been unlocked, not "ready_for_ai" again

**Decision:** A product is eligible for AI generation when either (a) its
status is `ready_for_ai` (first generation) or (b) its status is `review`
and its `listing_packages` row is unreviewed (`reviewed_at IS NULL` —
i.e., explicitly unlocked via `unlockForRegeneration`). Products never
return to `ready_for_ai` after reaching `review`.

**Reason:** The original GMCOM-007 guard only allowed generation from
`ready_for_ai`, which made `unlockForRegeneration`'s effect (clearing
`reviewed_at`) unreachable — nothing could ever follow it with an actual
regeneration. This closes that gap without any UI change: the eligibility
check is server-side only.

## 2026-08-02 — AI provider access is abstracted behind a swappable interface

**Decision:** GM Commerce never depends directly on any single AI vendor's SDK. A provider interface (`lib/ai/provider-types.ts`) sits between the Listing Generator and every vendor; the active provider (Mock, OpenAI, or Anthropic) is selected entirely through the `AI_PROVIDER` environment variable, never by editing code. Exactly one canonical prompt-building pipeline (`lib/ai/prompt-builder.ts`) and one Listing Package schema are shared by every provider — content integrity, tone, and business rules live there, never duplicated inside a provider implementation.

**Reason:** Phil's explicit architectural decision, made before any provider was implemented: GM Commerce must never be locked to one AI vendor. OpenAI was chosen as the first real provider because ChatGPT already serves as this project's coordinating PM, OpenAI's platform covers text/vision/structured-output/future capabilities in one place, and keeping Anthropic wired as a first-class option (not a fallback) gives a strong default today without a long-term vendor lock-in.

## 2026-08-02 — Photos are a first-class commerce asset

**Decision:** GM Commerce must prepare, standardize, optimize, and review product photos before any marketplace draft is created. Photo work is not a secondary attachment step; it is part of the canonical commerce package and must follow Gathering Moss photo standards.

The photo pipeline must preserve originals, create standardized derivatives, avoid altering the product truthfully represented in the image, support human review, and produce marketplace-ready assets with known dimensions, orientation, crop, file type, quality, order, and status.

**Reason:** Listing copy alone cannot produce strong sales. Product photos are often the first and most influential part of a listing, and inconsistent or poorly prepared images would undermine the quality of the entire GM Commerce workflow. Shopify and Etsy publishing should consume approved photo assets rather than raw uploads.
