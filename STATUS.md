# Current Project Status

_Last updated: 2026-08-03_

## Phase A in progress — narrow schema-reconciliation scope only

Codex issued final judgment on the reset: `APPROVED FOR NARROW PHASE A
IMPLEMENTATION` (`PRODUCT_RESET_2026-08-03.md`, this repo) — strictly
schema reconciliation, the migration ledger, schema-from-empty CI, drift
detection, and the direct-SQL incident record. **Phase B0, Phase B, Etsy,
new AI features, and any implementation beyond this narrow scope remain
unauthorized and paused.** Inspecting HY-LOB01-C04's Shopify draft surfaced
real content defects; the in-flight fix
(GMCOM-015's Commerce Readiness Gate + a same-day follow-up moving price/
weight/collections/care-instructions ownership off Phil and Crystal) was
correct in direction but was being built piecemeal. Phil's instruction:
stop, produce the full reset package (vision, gap analysis, manual-handoff
inventory, autonomous intelligence architecture, persistent knowledge
design, provenance/confidence model, owner-authority model, Skrybix gap
analysis, completed-package review design, migration plan, test strategy,
revised roadmap), and do not resume implementation or begin Etsy work
until it's reviewed and approved. See that document for the full detail —
nothing below this notice should be read as current direction until the
reset is reconciled with it.

**Review history:** Copilot's review of Revision 2 returned `REQUIRES
REVISION 3 AND RE-REVIEW` (source: `gm-commerce` audit commit `bd650384`,
headquarters decision commit `da44c96`) — addressed in Revision 3.
Revision 3's first push (`b2feb51`) itself mischaracterized
`HY-LOB01-C04`'s test-session data as real; Phil caught and corrected this
same-day (`6455fc8`), adding the `RecordContext`/environment-isolation
model (§25) and database-change-control policy (§1.1) so this class of
mistake is structurally prevented going forward. **Codex then performed
the independent re-review of that corrected commit and returned `APPROVE
AFTER DOCUMENTED CORRECTIONS`** — twelve finite corrections, no broad
Revision 4 redesign required. All twelve were incorporated (`4cead48`,
§§5, 7–10, 14.1, 20, 22.2, 25). **Copilot is near a usage limit and did not
perform this or the following review; Codex performed both, per Phil's
relay.**

**Codex then performed the targeted verification of that commit (`8ecf2e4`)
and returned `APPROVE AFTER TWO TARGETED CORRECTIONS`:** all twelve
corrections were present and substantially satisfied the review; two
internal inconsistencies needed fixing — §13 artifact 7 still pointed to
the retired package-wide `evidenceRefs` instead of §7's `fieldLineage[]`,
and §10.1's verification-threshold rule required Phil's confirmation for
*every* price/weight/shipping/compliance/safety/marketplace-policy claim
regardless of evidence, which would have recreated the exact bottleneck
this reset's own governing completion test exists to eliminate. Both were
corrected: §13 now references `fieldLineage[]` directly, and §10.1
distinguishes authoritative-system-of-record facts and existing-policy
applications (auto-verifiable) from AI-generated recommendations
(evidence-backed, validated, approved once at completed-package review —
never as a separate per-field fact-check). **Codex then confirmed both
fixes and issued final judgment: `APPROVED FOR NARROW PHASE A
IMPLEMENTATION`** — strictly schema reconciliation, migration ledger,
schema-from-empty CI, drift detection, and the direct-SQL incident record.
No broader implementation sequence is authorized, and Etsy remains paused.

**Separately, Phil corrected a standing factual error**: `HY-LOB01-C04`
was never a real plant cutting or Gathering Moss inventory item —
synthetic test data only, created and used by Claude for pipeline testing
— and explicitly authorized deleting everything specifically tied to it.
That deletion is complete (database rows, the live Shopify draft, local
photo files); see the Active Work entry below and `PRODUCT_RESET_2026-08-03.md`
§1.2/§22.1 for the full disposition.

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

- **GMCOM-016 — Autonomous Commerce Intelligence Audit completed on `hydracoresystems-listing-audit-engine` (documentation only):** independent review finds that current GM Commerce is a safe draft-publishing pipeline, not yet an autonomous commerce operating system. The audit inventories manual work, maps data ownership, defines measurable autonomy evaluations, identifies missing research/vision/policy/pricing/learning capabilities, and documents audit-score compression from the real Shopify export (42 listings scored only 51-77). No production behavior changed. Claude's reset proposal was not yet published at review time; its required independent review gate is documented in `docs/autonomous-commerce-intelligence-audit.md`.

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

- **Product Reset Revision 3 — `APPROVED FOR NARROW PHASE A IMPLEMENTATION`.** Responds to all 15 finite corrections and 7 canonical decisions from the Revision 2 review (`docs/autonomous-commerce-intelligence-audit.md`), then to Codex's twelve corrections (`4cead48`) and two targeted fixes (`8ecf2e4`). Codex's final judgment, after confirming both fixes: narrow Phase A only — schema reconciliation, migration ledger, schema-from-empty CI, drift detection, the direct-SQL incident record (§1.1, §23 of the reset document). Phase B0, Phase B, Etsy, and any broader implementation remain unauthorized. **Separately, Phil corrected a standing factual error:** `HY-LOB01-C04` was never a real plant cutting or Gathering Moss inventory item — synthetic test data only — and explicitly authorized deleting everything tied to it. That deletion is complete: the `gm-commerce` database row (cascading to every child table), the live Shopify draft (`gid://shopify/Product/10220386648384`, via `productDelete`, no errors), and the local photo folders are all confirmed gone. **Every prior entry below this point that describes `HY-LOB01-C04` as a real cutting, real plant, or real photos reflects what was believed true at the time it was written, not the corrected, current understanding — treat this note as the standing correction rather than editing dozens of historical entries.** See `PRODUCT_RESET_2026-08-03.md` §1.2 and §22.1 for the full disposition, including the one item out of this project's control (a possible corresponding Skrybix-side record, not touched). **Phase A schema-reconciliation work is committed and pushed to `gm-commerce` (`77c8593`, then corrected at `f316ec2`):** the two previously-uncommitted migration files, an updated `supabase/schema.sql`, a `schema-from-empty` CI job. Typecheck, the full test suite (111 tests as of `f316ec2`), and the production build all pass locally.

**Correction (`f316ec2`): the initial migration-ledger characterization was factually wrong.** Phil attempted the originally-supplied ledger backfill and got `ERROR: 42P01: relation "supabase_migrations.schema_migrations" does not exist` — **this project's live database has never had a Supabase CLI migration ledger at all.** Every schema change in its history was applied by pasting SQL directly into the Supabase SQL Editor; this was not "two migrations need their ledger rows backfilled," it was a complete absence of migration history. Re-verifying all seven committed migrations (not just the two newest) against the live database found: 6 are genuinely applied (`20260801000000` through `20260802040000`); the 7th, `20260803000000_commerce_field_ownership.sql`, is **not** — `commerce_details.price` and `listing_packages.content_provenance` both return Postgres error `42703` ("column does not exist") on direct probe, despite the migration being committed to git.

`gm-commerce` now has `supabase/ledger-bootstrap.sql` — a single, idempotent, reviewed SQL file that creates the ledger from scratch and records exactly the 6 verified migrations (deliberately excluding the unapplied 7th, rather than fabricating it as applied) — generated by `supabase/generate-ledger-bootstrap.js` and structurally tested by `supabase/ledger-bootstrap.test.ts` (6 tests, passing; could not be run against a real live Postgres — no working Docker daemon or local Postgres was available in the environment that built it, disclosed rather than silently skipped). The CI drift job is renamed `schema-drift-deferred` and its messaging corrected so a skip is never described or perceived as passing protection; the Supabase CLI pin moved from `latest` to `2.111.0`, matching the exact version the ledger's table structure was extracted from (its own compiled binary, not documentation or memory).

**One action remains, for Phil only:** open `supabase/ledger-bootstrap.sql` on GitHub, copy its complete contents, paste into the already-open Supabase SQL Editor, click Run, and report back the result of its final verification query (expected: exactly 6 rows, `20260801000000` through `20260802040000`, in order). No password, connection string, or credential management needed. Phase A closes once that result is confirmed.

The broader uncommitted GMCOM-015/016 working tree on `gm-commerce` (Commerce Readiness Gate, the legacy `/commerce` UI, AI care-research changes, etc.) remains deliberately uncommitted — out of this narrow Phase A scope, per the reset document's disposition table and Codex's authorization.

- **GMCOM-014 real-export smoke check passed (2026-08-02) using `products_export_1_revised_plant_tags.csv`:** the production parser normalized 255 Shopify product rows into 42 unique handles with no parsing error; repeated image/variant rows grouped correctly (including `fidget-animals` with 64 unique images). Status filters match the export: 37 active, 2 draft, 1 archived, 2 with no status. Category filters expose Accessories (12), Plant Accessories (2), Plant Additive (1), Planter (5), Plants (14), and Uncategorized (8). All 42 listings are in the 50-79 Commerce Score filter; the 0-49 and 80-100 filters correctly return none. `plant-tags-set-of-25` is active, Uncategorized, rank 41/42 (77/100): actual title `3D Printed Plant Tags – Set of 25 Reusable Garden & Houseplant Labels`; its raw exported description, SEO title/description, and nine tags are retained in the detail view. Its findings are one-image count, missing Product Type, and no internal link. Commerce Audit loaded the real report with no parsing/rendering error. Input remains browser-local: `File.text()` into in-memory React state only, with no upload, browser storage, Supabase, AI, Shopify, rewrite, or persistence path.

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
