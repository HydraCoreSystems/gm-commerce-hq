# Current Project Status

_Last updated: 2026-08-04_

## Current authoritative state — Phase C Slice 5 merged (2026-08-04)

**Phase C (OneDrive evidence library and secure ingestion) is now five
slices deep, all merged to `gm-commerce` `main`.** This document's prior
"Phase B complete, no later phase has started" framing below is stale —
Phase C was separately authorized and slices 1–4 (OneDrive connector
boundary, durable ingestion ledger, evidence revision ingestion, secure
extraction/adversarial validation — `gm-commerce` PRs #9–#13) landed
without an intermediate update here. Recording that gap plainly rather
than editing it out: this section is the catch-up, not a claim that
nothing happened in between.

**Slice 5 (freshness, revalidation, source-family assessment,
contradiction routing, promotion gates)** merged as `gm-commerce` PR #14,
merge commit `b205ea79b325e96b301054c96f941a462b41ad10`, authorized by
Phil. This slice's path to merge is worth recording precisely, since it's
a direct example of the multi-agent review discipline this project now
runs on:

- The implementing agent's own report claimed CI was green before an
  independent review (Claude, standing in for Codex, who was at a usage
  limit) found it wasn't: `gmcom_guard_freshness_policy_activate` silently
  rewrote a correctly-linked new policy version to `superseded` instead of
  promoting it — since the table is append-only, this meant a freshness
  policy could never actually be changed after its first creation, with no
  error ever raised, and the CI test asserted the broken behavior as
  correct. Reproduced live against Postgres before reporting it.
- Two more rounds followed the same pattern — a fix reported as complete,
  independently re-verified rather than trusted, a real gap found each
  time (a second report claimed "functionally equivalent," which a live
  reproduction disproved; a third fix for the real structural bug
  introduced a CI test-isolation regression in an unrelated step, caught
  by checking the actual GitHub Actions run rather than trusting a
  "10/10 green" claim).
- The structural fix that survived: `canonical_freshness_policies`' current
  version is now derived as the head of its append-only `supersedes_policy_id`
  chain (the row nothing else references yet) via a dedicated
  `gmcom_current_freshness_policy()` function, never a mutable flag frozen
  at insert time; the insert trigger locks the predecessor row and rejects
  (raises) branching, orphaned, or non-increasing supersession attempts
  instead of silently rewriting them.
- Final verification, independently reproduced by the reviewer (not taken
  from any agent's report): typecheck clean, **587/587 tests pass**,
  production build succeeds, all GitHub Actions jobs green on the actual
  merged SHA, and the entire `phase-c-slice5-live` CI job replayed by hand
  against a fresh local Postgres 16 instance end-to-end with zero errors.

**Phase C Slice 6 (if any) is not authorized and has not started.** Etsy
and marketplace-publishing expansion remain out of scope until separately
authorized. This section supersedes every older status statement below
it, including the "Phase B complete, no later phase has started" section
immediately following.

## Current authoritative state — Phase B complete (2026-08-04)

**Phase A, Phase B0, and all four Phase B slices are complete and merged.**
The newest application merge is `gm-commerce` PR #8 at merge commit
`2f47ea6059eee3993dfa54507fa2a1cec501c9e0`. Slice 4 closes the Phase B0
migration window with an idempotent, concurrency-safe
`LegacyCorrectionEvent` to canonical `Correction` migration: only
production, operational, genuinely owner-approved, active events can
migrate; test/non-genuine/unmapped events remain retained and ineligible;
temporarily unresolved canonical SKUs remain deferred and replayable.

Final verification: **473/473 tests pass**, typecheck and production build
pass, and GitHub CI run 78 passes ordered migrations and the consolidated
schema against fresh PostgreSQL, including eligibility, deferral,
reclassification, later identity resolution, duplicate prevention, exact
lineage, and immutable audit emission. `origin/main` was confirmed at the
exact merge commit above.

**Phase B is complete. No later phase has started.** Etsy, marketplace
publishing expansion, and Phase C research/knowledge ingestion remain out
of scope until separately authorized. This section supersedes every older
status statement below it.

## Current authoritative state (2026-08-04)

**Phase A, Phase B0, and Phase B Slices 1–3 are complete and merged.**
The newest application merge is `gm-commerce` PR #7 at merge commit
`2fe5831d0ffb24c9758402b0637fa7655003578e` (Phase B Slice 3: the
`IntelligenceRepositoryV1` contract foundation, explicit capability and
authorization/audit policy inventory, load-bearing canonical-mutation audit
stream, additive legacy `listing_packages`/`commerce_details` dual-write,
idempotent backfill, supersession, reconstruction ledger, and drift
validation). The final adversarial review found and corrected the original
contract test's failure to prove §9's command-audit requirement. Final
verification: **467/467 tests pass**, typecheck and production build pass,
and GitHub CI run 73 passes both ordered migrations and the consolidated
schema against fresh PostgreSQL, including real Claim insert/supersession
audit events. `origin/main` was confirmed at the exact merge commit above.

**Phase B Slice 4 is next and has not started.** Its bounded scope is the
eligible `LegacyCorrectionEvent` → canonical `Correction` migration job
from PRODUCT_RESET_2026-08-03.md §14.1. Etsy and later publishing work remain
out of scope until separately authorized. Historical status entries below
remain unchanged as a record of what was true when written; this section is
the controlling current state.

## Correction to this document's own prior state (2026-08-03, this update)

**The "Phase A in progress... Phase B0 remains unauthorized and paused"
framing directly below this notice is STALE and describes an earlier point
in the same day, not the current state.** Re-verified directly against
`gm-commerce` git log rather than trusted from memory: **Phase A is
complete** (unchanged from below), and **Phase B0 is also complete and
merged** — `gm-commerce` PR #3 (`55a89b2`, completed-package review shell +
`LegacyCorrectionEvent`, PRODUCT_RESET_2026-08-03.md §19/§14.1) and PR #4
(`46d364d`, function search-path hardening on Phase B0's two new
functions) are both merged to `main`. `main` is currently at `46d364d`.
Live Supabase Security Advisor: 0 errors, 7 pre-existing warnings, none
introduced by Phase B0. **Phase B, slice 1 of 4 (canonical entity
foundation — §5's 18 entity types, the RecordContext envelope per §25, and
the minimal entity-level Repository slice per §9) has been built on branch
`phase-b-slice-1-canonical-entities` and is open as `gm-commerce` PR #5,
NOT merged** — see the Active Work entry below for exact commits, test
counts, and the full slice 2-4 breakdown. Below this notice, leave the
original Phase A narrative as written (it was accurate for its own point
in time); this notice is the correction, not an edit to history.

## Phase A — narrow schema-reconciliation scope (complete; historical framing below, see correction above)

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

**Closeout (`gm-commerce` `4e40906`): Phil ran the bootstrap and it succeeded.** The live ledger returned exactly the six expected rows (`20260801000000` through `20260802040000`); `20260803000000` correctly absent. That migration is now **retired**, not merely left pending — per the reset document's reconciliation, `content_provenance` is superseded by the canonical claim/evidence model and field-level `FieldLineage`, and the price-ownership decision's approved home is `CommercePackage`/`VariantOffer`, not this column. Its file moved to `docs/incidents/2026-08-03-retired-migration-commerce_field_ownership.sql.txt` (non-executable extension) and its two columns were removed from `supabase/schema.sql`; `supabase/migrations/` now contains exactly the six active, applied, ledgered migrations. Live schema, migration ledger, and committed migration sequence all agree. Full suite (111 tests), typecheck, and build pass. **Phase A is complete.**

**Correction (2026-08-03, later the same day): Phase B0 is complete, not "not started."** The line that previously stood here — "Next approved phase awaiting Phil's authorization: Phase B0 ... not started" — was stale by the time this was re-verified. `gm-commerce` PR #3 (`55a89b2`, merged) built the completed-package review shell's data layer: `legacy_correction_events` (§14.1's `LegacyCorrectionEvent`, full `RecordContext` envelope, structural constraints preventing a test/demo/fixture-purpose event from ever being marked `migrated`, or a non-genuine approval state from carrying a true eligibility flag) and `owner_effort_events` (§20's instrumentation: review/approval_rejection/correction/escalation_resolution/navigation_data_entry/physical_labor categories, explicit start/stop pairs, an `owner_effort_unresolved` view for abandoned starts). PR #4 (`46d364d`, merged) pinned `search_path` on both new Postgres functions (`gmcom_record_instant_owner_effort`, `gmcom_apply_legacy_correction`) as a same-day hardening follow-up. Both are on `main`.

**Phase B, slice 1 of 4 — `MERGED`.** `gm-commerce` PR **#5** (https://github.com/HydraCoreSystems/gm-commerce/pull/5), branch `phase-b-slice-1-canonical-entities`, merged into `main` at commit **`b1af27aa737bbf6f792ffa22d01190efe8ce1859`** (merge commit; `origin/main` and local `main` both confirmed aligned at this SHA). Delivers the canonical entity foundation (PRODUCT_RESET_2026-08-03.md §5/§9/§25): all 18 canonical entity tables with real relational structure (not a relabeling of `listing_packages`/`commerce_details`), a full `RecordContext` envelope on every one, and the minimal `createEntity`/`getEntity`/`listEntities` slice of `IntelligenceRepositoryV1`.

Went through three rounds of Codex review before approval, each finding real structural gaps and each fixed for real rather than papered over — worth recording precisely since this is exactly the discipline this reset exists to enforce:
- **Round 1 (Codex):** `createEntity` allowed caller-supplied fields to overwrite protected `RecordContext`/identity columns (spread-order bug); `callerEnvironment` was accepted per-call rather than bound per-repository-instance, so any call site could request `production` directly; foreign keys didn't enforce matching environment across parent/child rows, allowing a cross-environment reference. Fixed with an explicit protected-column reject list, a repository bound to one environment at construction, and composite `(id, environment)` foreign keys, live-verified against real Postgres in CI.
- **Round 2 (Codex):** the environment-binding fix wasn't yet structural — the repository constructor was still public and callable with an arbitrary environment; `createEntity` could still be passed `ownerApproval.approvalState: "genuine"` and `eligibility: true` with no real `OwnerDecision` behind it, fabricating owner approval; all 18 tables defaulted `environment` to `production`, so an insert omitting it silently became production data. Fixed by narrowing the creation-context type so privileged approval/eligibility values don't type-check (plus a runtime backstop), and by dropping the production default so a missing `environment` now fails `NOT NULL` — both live-verified in CI.
- **Round 3 (Codex):** the constructor lockdown from round 2 was a naming/import-location convention (an audit test scanning only `app/`/`components/`), not an actual construction boundary — any `lib/`, script, or future source directory could still import the internal class directly. Fixed with a private constructor plus a module-private runtime token that only the trusted factory holds (a forged `Symbol()` with a matching description still fails — JS symbol equality is identity-only, not description-based), so arbitrary-environment construction is now impossible from any import location, not just discouraged from the ones an audit happened to check. The test-only environment-accepting factory (`lib/canonical/testing.ts`) was deleted; tests now go through the same trusted factory, controlling `process.env.GMCOM_ENVIRONMENT` for the duration of each test.

**Final verification (commit `3dbd00e`, the state actually merged):** `npm run typecheck` clean; `npm test` **279 passed / 0 failed** (up from 111 at Phase A close); `npm run build` succeeds; CI `schema-from-empty` green against a real Postgres 15 service container, including live checks that a mismatched-environment foreign-key insert fails, an `environment`-omitting insert fails `NOT NULL`, and every constructor-bypass attempt throws. Codex's final review, after all three rounds: **approved**. Phil merged PR #5 into `main` directly (not via `gh pr merge` by the AI system) — confirmed via `gh pr view` (`state: MERGED`, `mergeCommit.oid: b1af27a...`) and via `git fetch`/`git log` showing the merge commit live on `origin/main`.

**Full slice 2-4 breakdown** (from the PR, unchanged): slice 2 = Evidence & Claim model (§6) — sources/revisions/anchors, claims, precedence, contradictions; slice 3 = Repository contract completion + `content_provenance`/`commerce_details` dual-write bridge and backfill; slice 4 = `LegacyCorrectionEvent` → canonical `Correction` migration job (§14.1).

**Honest limitation carried forward, not yet closed:** `service_role` has Postgres `BYPASSRLS`, so RLS is not load-bearing against the app's actual caller today — the real enforcement is the TypeScript repository layer (bound environment, protected columns, private-constructor lockdown). RLS remains correct, live-tested defense-in-depth for a future non-service-role caller. Documented in `DECISIONS.md`.

**Phase B slice 2 is not authorized yet — awaiting Phil's separate go-ahead**, per his explicit instruction not to begin it automatically after merge.

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
