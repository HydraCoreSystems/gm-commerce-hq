# Current Project Status

_Last updated: 2026-08-03_

## PAUSED — awaiting Phil's review of the product/architecture reset

Implementation is paused pending Phil's sign-off on
`PRODUCT_RESET_2026-08-03.md` (this repo). Inspecting HY-LOB01-C04's real
Shopify draft surfaced real content defects; the in-flight fix
(GMCOM-015's Commerce Readiness Gate + a same-day follow-up moving price/
weight/collections/care-instructions ownership off Phil and Crystal) was
correct in direction but was being built piecemeal. Phil's instruction:
stop, produce the full reset package (vision, gap analysis, manual-handoff
inventory, autonomous intelligence architecture, persistent knowledge
design, provenance/confidence model, owner-authority model, Skrybix gap
analysis, completed-package review design, migration plan, test strategy,
revised roadmap), and do not resume implementation or begin Etsy work
until he's reviewed it. See that document for the full detail — nothing
below this notice should be read as current direction until the reset is
reconciled with it.

## Current Milestone (pre-pause)

**GMCOM-012 — the Shopify Draft Publisher — is complete and verified
against a real Shopify store.** The first complete production workflow
now genuinely works end to end: `HY-LOB01-C04`'s real photos were
scanned, standardized, and approved; its Listing Package was reviewed;
`publishToShopify` created a real Shopify draft product
(`gid://shopify/Product/10220386648384`). Verified field-by-field against
Shopify's own response (title/description/vendor/product type/tags,
all 4 images in the correct order with correct alt text, status
confirmed `DRAFT`) and for manual-edit resilience (a direct Shopify-side
edit was cleanly overwritten by a GM Commerce republish, same product ID,
no duplicate). Full detail in
`handoffs/2026-08-02-GMCOM-012-shopify-draft-publisher.md`.

Getting real credentials working took real troubleshooting worth knowing
about: this store's Shopify plan doesn't offer the legacy static-token
"custom app" flow at all, only a Dev-Dashboard app's Client ID/Secret
exchanged for a short-lived token — `lib/shopify/real-client.ts` now
supports both, with automatic token refresh for the second mode. Also
learned that changing an app's requested scopes doesn't retroactively
grant them to an already-installed instance (uninstall/reinstall
required) — cost real time to discover, now documented.

One deliberate, one-time exception: Claude marked `HY-LOB01-C04`'s
Listing Package reviewed and its photo set approved itself, rather than
Phil or Crystal, specifically to make this real verification possible —
explicitly authorized by Phil in chat, with ChatGPT's concurring
recommendation that it be treated as a development-only exception. The
standing "Phil or Crystal approves real content" rule is unchanged going
forward.

**GMCOM-009, GMCOM-010, and GMCOM-011 are all complete on `main` and
their migrations are applied.** GMCOM-011's photo pipeline migration
(previously pending) has since been applied and is now confirmed working
via the GMCOM-012 real run above.

**GMCOM-011 — the photo preparation and approval pipeline is built.** Raw
uploads (JPEG/JPG, PNG, WebP, and HEIC/HEIF with no manual conversion
required) become standardized, human-approved commerce assets: a
provider-neutral `ImageProcessor` interface (same pattern as the AI
Provider abstraction), deterministic inspection (blur/exposure/near-
duplicate signals, no ML model), a `square_marketplace` derivative padded
onto a configurable brand-color mat (never a real background swap — kept
truthful per the issue's own guardrails) plus a conditional
`detail_uncropped` derivative, alt text generated via the existing AI
Provider abstraction, and an approval gate requiring a hero image and
alt text on every photo. New `/photos` and `/photos/[sku]` pages, 34 new
tests (60 total), verified against real files: `HY-LOB01-C04`'s two real
plant photos and a real HEIC source, both processed correctly with
originals confirmed byte-for-byte unchanged. **Migration not yet
applied** — full detail and the copy-paste SQL block in
`handoffs/2026-08-02-GMCOM-011-photo-pipeline.md`.

**GMCOM-009 verified live.** Before starting GMCOM-011, Claude ran a real
generation against `HY-LOB01-C04` (real Supabase, real OpenAI call):
reservation, generation, and finalization all succeeded, version
incremented 1→2 with the old version correctly archived, and no lock
remained afterward. One unrelated operational finding: running two
`next dev` processes against the same checked-out folder at once corrupts
both (shared `.next` build directory) — not a code bug, just worth
knowing.

Both AI calls' output is now runtime-checked (zod) against the canonical
schemas before persistence; concurrent Generate clicks for the same SKU
can produce at most one paid provider call (atomic reserve-before-call,
backed by three Postgres functions); every regeneration archives the
package it replaces into `listing_package_versions`; `markReviewed`/
`unlockForRegeneration` are guarded against acting while a generation is
in flight; and the pre-existing bug where regeneration was unreachable
(a product never returned to `ready_for_ai` once reviewed) is fixed.

**GMCOM-010** (Issue #9 — migrations, CI, operational baseline) was
completed via a separate PR, reconciled with GMCOM-009, and merged —
versioned Supabase migrations replacing reliance on `create table if not
exists` for upgrades, a CI workflow (typecheck/test/build on every push),
a pinned Node version (`.nvmrc`), structured server logging, and
server-action input validation (including SKU-derived photo-path
traversal protection, now reused by GMCOM-011's image-serving route).

**GMCOM-013/014 — Shopify Listing Audit Engine and Commerce Audit workspace are built on `hydracoresystems-listing-audit-engine` (`372d6f7` + `e50d95d`).** `/audit` is a strictly read-only, browser-local Shopify CSV workspace: deterministic 0-100 Commerce Scores, ranked improvement queue, score/category/status filters, and expandable source snapshots (title, description, SEO fields, tags, category, and findings). It does not upload the export, call AI, mutate Shopify, or rewrite copy. The reusable audit library is documented and covered by tests. **A current Gathering Moss Shopify CSV was not present in the application workspace, so real-export validation remains pending Phil providing/exporting that file.**
## Previous Milestone

**GMCOM-008 complete and verified — the Listing Quality Engine is live.**
Listing generation is now a multi-stage pipeline (Gather Facts → Prompt
Builder → AI Generation → Self Review + Revision → Final Listing
Package), still behind the same AI Provider abstraction. Verified against
a real product: `HY-LOB01-C04` ("Hoya lobbii") now has a real, genuinely
useful Listing Package + Quality Summary sitting in `/review`, awaiting
Phil or Crystal's actual review.

## Completed

- **GMCOM-012 — Shopify Draft Publisher, built and verified against a
  real store.** Commits `572a858`/`f2a073d` on `main`, pushed. 19 new
  tests (79 total). Real Shopify draft created and field-verified for
  `HY-LOB01-C04`; see handoff for full detail.
- **GMCOM-013/014 — Listing Audit Engine and read-only Commerce Audit workspace built** (`372d6f7`, `e50d95d`): local Shopify CSV parsing, transparent 0-100 scoring, ranking, score/category/status filtering, source-detail inspection, JSON reports, documentation, and test coverage. Real Gathering Moss export validation awaits the current CSV.

- **GMCOM-011 — Photo preparation and approval pipeline built and now
  fully verified (file-level, then real database-backed via GMCOM-012's
  run).** Commit `9ee8eec` on `main`, pushed. 34 new tests (60 total,
  since grown to 79 with GMCOM-012). Migration applied and confirmed
  working live — see `handoffs/2026-08-02-GMCOM-011-photo-pipeline.md`
  and the GMCOM-012 handoff.
- **GMCOM-010 — migrations, CI, operational baseline.** Reconciled with
  GMCOM-009 and merged (merge commit `9dc82d8`) — see Phil's PR #1.
- **GMCOM-009 — AI generation hardened: atomic (reserve/finalize/release
  locking, ≤1 paid call under concurrent Generate clicks), versioned
  (`listing_package_versions`, append-only), and runtime-validated (zod
  schemas reject malformed/invalid provider output before persistence).
  Also fixed regeneration being unreachable in the old code.** Migration
  applied by Phil and verified live: a real generation against
  `HY-LOB01-C04` succeeded end-to-end (reserve → real OpenAI call →
  finalize → version 1→2, old version archived, lock released).
- GM Commerce Headquarters repository created.
- Core mission and Version 1 direction established.
- Existing Skrybix repository identified: `HydraCoreSystems/skrybix-webapp`.
- Product SKU Generator built by Claude and documented.
- Product SKU Generator repository location recorded and pushed:
  `HydraCoreSystems/product-sku-generator`.
- GMCOM-001 baseline documented. Milestone 0 real-world validation
  completed by Phil.
- **GMCOM-002 — GM Commerce application foundation built.**
- New Supabase project provisioned for GM Commerce under a co-owner's
  separate account (project ref `wcrcllhvgbhykbonopzx`).
- **GMCOM-003 — merged.** Skrybix commerce selection handoff, production-
  deployed.
- **GMCOM-004 — verified end-to-end with a real cutting** (`HY-LOB01-C04`).
- **GMCOM-006 — guided intake experience built and verified.**
  `HY-LOB01-C04` has had its photo folder and photos confirmed since this
  milestone.
- **GMCOM-007 — canonical AI Listing Package generator + Provider
  abstraction, fully verified including a real OpenAI call.** Vendor-
  agnostic architecture: `GM Commerce → Listing Generator → Prompt
  Builder → AI Provider interface → {OpenAI, Anthropic, Mock}`. Anthropic
  provider complete but intentionally untested per instruction.
- **GMCOM-008 — Listing Quality Engine (Phase 1), commit `827a3ac` on
  `main`, pushed.** Single-prompt generation replaced with a multi-stage
  pipeline: Gather Facts (new — now includes real photo-folder metadata,
  `photoCount`) → Prompt Builder → AI Generation → Self Review + Revision
  (combined into one API call for cost efficiency, per explicit
  instruction — 2 total AI calls per generation, not 3) → Final Listing
  Package. New `listing_packages.quality_summary` column (confidence,
  unsupported claims caught, missing information, recommendations),
  surfaced directly to the human reviewer in `/review`. Verified twice:
  pipeline logic with the Mock provider (updated to return matching
  shapes per pipeline stage) against a scratch record, then a real
  generation against `HY-LOB01-C04` via OpenAI — confidence came back
  honestly "medium," zero unsupported claims, and specific actionable
  guidance (price, dimensions, rooted status, "use the available photos
  to confirm condition"). Real record deliberately left at "Needs
  review," not approved by Claude. Full detail in
  `handoffs/2026-08-02-GMCOM-008-listing-quality-engine.md`.
- **Found and fixed a real bug in this repository** (prior session):
  `DECISIONS.md` had unresolved git conflict markers from an incomplete
  stash-pop; merged cleanly.

## Active Work

- **GMCOM-014 real-export smoke check passed (2026-08-02) using `products_export_1_revised_plant_tags.csv`:** the production parser normalized 255 Shopify product rows into 42 unique handles with no parsing error; repeated image/variant rows grouped correctly (including `fidget-animals` with 64 unique images). Status filters match the export: 37 active, 2 draft, 1 archived, 2 with no status. Category filters expose Accessories (12), Plant Accessories (2), Plant Additive (1), Planter (5), Plants (14), and Uncategorized (8). All 42 listings are in the 50-79 Commerce Score filter; the 0-49 and 80-100 filters correctly return none. `plant-tags-set-of-25` is active, Uncategorized, rank 41/42 (77/100): actual title `3D Printed Plant Tags – Set of 25 Reusable Garden & Houseplant Labels`; its raw exported description, SEO title/description, and nine tags are retained in the detail view. Its findings are one-image count, missing Product Type, and no internal link. Commerce Audit loaded the real report with no parsing/rendering error. Input remains browser-local: `File.text()` into in-memory React state only, with no upload, browser storage, Supabase, AI, Shopify, rewrite, or persistence path.

- **`HY-LOB01-C04` now has a real Shopify draft product**
  (`gid://shopify/Product/10220386648384`) created under GMCOM-012's
  one-time verification exception — Phil or Crystal should look at it in
  Shopify admin and decide what to do with it (further edits, publish it
  live themselves when genuinely ready, or discard). GM Commerce itself
  has no path to make a product live; that isn't a gap, it's by design.
- Phil or Crystal to actually review `HY-LOB01-C04`'s real Listing
  Package in `/review` going forward under the normal (non-exception)
  process — it's currently reviewed under GMCOM-012's one-time exception,
  not by an owner's own judgment.
- Update GitHub Issues for GMCOM-007/008/009/010/011/012 — no GitHub API
  access available to Claude in this environment. GMCOM-012 has no Issue
  at all yet (assigned directly in chat) — recommend creating one
  retroactively for the record.
- Generate at least one real SKU through the Product SKU Generator's live
  app — its `sku-log.json` was still empty as of the last check.

## Current Blockers

- No GitHub API access for updating Issues directly.
- No live channel between Claude and ChatGPT exists yet; coordination
  currently depends on Phil relaying between them.
- AI provider usage limits (Anthropic/OpenAI/etc.) are not automatically
  visible to the project manager.

## Next Highest-Priority Task

**Etsy publishing, reusing the pattern GMCOM-012 just proved end-to-end**
— or, alternatively, closing out GMCOM-013/014's real-export validation
now that a real Shopify store and real product data actually exist to
test the audit engine against. Per the existing "Shopify before Etsy"
decision in `DECISIONS.md`, Shopify is now genuinely proven (not just
built) end-to-end, so Etsy is the natural next marketplace using the same
provider-neutral client pattern, publication-tracking table shape, and
never-overwrite-live logic. Needs a ready GitHub Issue before starting,
per this repo's own operating rules — none exists yet for either option.

## AI Capacity

| Contributor | Current role | Capacity status | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / reviewer / contributor | Available | Sequence Etsy publishing or GMCOM-014 real-export validation |
| Claude | Primary hands-on builder | Available | GMCOM-009/010/011/012 complete; awaiting next assignment |
| GitHub Copilot | GitHub-native implementation contributor | Available | Completed GMCOM-003, GMCOM-005; GMCOM-010 (PR #1); GMCOM-013/014 |
| Phil | Product owner / workflow validator | Available as schedule permits | Review the real Shopify draft product; provide a real Shopify CSV export for GMCOM-014 |

Capacity status should be updated whenever a provider limit is reached or resets.
