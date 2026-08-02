# Current Project Status

_Last updated: 2026-08-02_

## Current Milestone

**GMCOM-009, GMCOM-010, and GMCOM-011 are all complete on `main`.** The
GMCOM-009 migration has been applied by Phil and verified live (see
below); the GMCOM-011 photo pipeline migration has not yet been applied.

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

- **GMCOM-013/014 — Listing Audit Engine and read-only Commerce Audit workspace built** (`372d6f7`, `e50d95d`): local Shopify CSV parsing, transparent 0-100 scoring, ranking, score/category/status filtering, source-detail inspection, JSON reports, documentation, and test coverage. Real Gathering Moss export validation awaits the current CSV.

- **GMCOM-011 — Photo preparation and approval pipeline built and
  file-level verified.** Commit `9ee8eec` on `main`, pushed. 34 new tests
  (60 total), `npm run typecheck`/`npm run build` clean. Migration not
  yet applied to the live project — see
  `handoffs/2026-08-02-GMCOM-011-photo-pipeline.md`.
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

- **GMCOM-014 real-export smoke check attempted (2026-08-02) but blocked before load:** `products_export_1.csv` is not present in the GM Commerce worktree, any child directory, tracked branch file, or fetched remote branch. The requested 255-row / 42-product normalization, real-product grouping/filter results, and the `Plant Tags - set of 25` detail/rank therefore remain unverified until the exact export is provided. Static review confirms the audit path reads the selected file via `File.text()` into in-memory React state only; it has no upload, browser storage, Supabase, AI, Shopify, rewrite, or persistence path.

- **Phil to run the GMCOM-011 migration SQL against the live Supabase
  project** (copy-paste block in
  `handoffs/2026-08-02-GMCOM-011-photo-pipeline.md`) — the photo pipeline
  is blocked in the real app until this runs.
- Phil or Crystal to actually review `HY-LOB01-C04`'s real Listing
  Package in `/review` — edit, approve, or discard as they see fit (now
  on version 2, after the GMCOM-009 live verification regenerated it;
  version 1 is fully recoverable in `listing_package_versions`).
- Update GitHub Issues for GMCOM-007/008/009/010/011 — no GitHub API
  access available to Claude in this environment.
- Generate at least one real SKU through the Product SKU Generator's live
  app — its `sku-log.json` was still empty as of the last check.
- Real Supabase-backed verification of the GMCOM-011 photo pipeline (scan,
  generate, hero/order, alt text, approve) once the migration is applied —
  file-level processing is verified, database persistence and the guided
  review UI's real appearance are not yet.

## Current Blockers

- No GitHub API access for updating Issues directly.
- No live channel between Claude and ChatGPT exists yet; coordination
  currently depends on Phil relaying between them.
- AI provider usage limits (Anthropic/OpenAI/etc.) are not automatically
  visible to the project manager.

## Next Highest-Priority Task

**Apply and verify the GMCOM-011 migration, then build the first real
Shopify draft-publishing adapter.** Every reliability/completeness
prerequisite ChatGPT sequenced ahead of Shopify (GMCOM-009 → GMCOM-010 →
GMCOM-011, `handoffs/2026-08-02-photo-pipeline-priority.md`) is now built.
Both halves of a real listing — Listing Package copy and an approved
photo set — can now be independently reviewed and approved. The next real
gap: read from a *reviewed* Listing Package and an *approved* photo set
together, create a real Shopify draft product, and store the returned
Shopify product ID back on the record. Per the existing "Shopify before
Etsy" decision in `DECISIONS.md`.

## AI Capacity

| Contributor | Current role | Capacity status | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / reviewer / contributor | Available | Sequence Shopify draft-publishing issue |
| Claude | Primary hands-on builder | Available | GMCOM-009/010/011 complete; awaiting next assignment |
| GitHub Copilot | GitHub-native implementation contributor | Available | Completed GMCOM-003, GMCOM-005; GMCOM-010 (PR #1) |
| Phil | Product owner / workflow validator | Available as schedule permits | Run GMCOM-011 migration SQL; review `HY-LOB01-C04`'s real Listing Package |

Capacity status should be updated whenever a provider limit is reached or resets.
