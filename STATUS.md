# Current Project Status

_Last updated: 2026-08-09 (Phase I Slice 4 merged via PR #57)_

## Phase I — Legacy-to-Canonical Bridge (I1, I2, I3, and I4 delivered and merged; next slice pending design/owner authorization)

Phase I bridges the real, working production pipeline (GMCOM-001–012) into the canonical model that Phases B–H built around. The authoritative slice plan is `phase-i-slice-plan.md` (this repo). **The master completion map is `COMPLETION.md` (this repo)** — a plain-English answer to "what must be finished before GM Commerce is operationally complete," including the remaining Phase I slices in dependency order, the owner decisions that block them, and measurable exit criteria.

**Status: I1, I2, I3, and I4 delivered and merged.** I1 via PR #54 (`2c26b61`); I2 via PR #55 (`bef5a5d`); I3 via PR #56 (`285a2a0`); **I4 via PR #57 (`gm-commerce/main` at `e58766e`)**. The identity backbone and the approved-photo bridge are complete. The likely next slice (historical approved-photo backfill using the I4 bridge machinery) is a **design recommendation only** (see `phase-i-slice-plan.md`) and has not begun. No later slice (CommercePackage, claims, drift hardening) has begun.

`gm-commerce/main` is at `e58766e67e5877d8e34187846d63476b7a63c4f9` (merge of PR #57 / I4).

| Slice | Scope | PR | Status | Merge |
|---|---|---|---|---|
| I1 | Bridge foundation + atomic identity RPC (`canonical_legacy_entity_bridge` + `gmcom_bridge_product_identity`) | #54 | Merged | `2c26b61` |
| I2 | Ongoing `ready_for_ai` integration + durable bridge jobs (`canonical_legacy_bridge_jobs` + `gmcom_mark_product_ready_for_ai`/enqueue/claim/finish) | #55 | Merged | `bef5a5d` |
| I3 | Existing identity backfill (explicit four-status allowlist; durable run + immutable per-run outcome ledger; dry-run→single-use real-run workflow) | #56 | Merged | `285a2a0` |
| I4 | Approved photo sets → canonical PhotoAsset (attached assets only; atomic approval+enqueue; durable per-photo mappings; replay-validating bridge RPC; fail-closed dispatch) | #57 | Merged | `e58766e` |

I2 delivered: **every real `ready_for_ai` transition now atomically enqueues a durable identity-bridge job** (`gmcom_mark_product_ready_for_ai` transitions `products.status` and enqueues/resolves the job in one transaction; the transition never rolls back on later canonical failure). Request-triggered processing (best-effort after transition + a bounded manual drain), leases (`FOR UPDATE SKIP LOCKED`, 300s, lease-expiry recovery), retries (exponential backoff 60s→1h), dead-lettering, end-to-end correlation continuity, immutable attempt history, and **owner/co-owner-only authorization for the manual drain** (via `lib/auth` `resolvePrincipal`/`requireRole`; staff/service rejected, no new permission invented) are all delivered and live-tested.

I3 delivered: **existing eligible legacy products can be backfilled into canonical ProductConcept/SKU/SourceRecord identity** through a required dry-run → single-use real-run workflow (an exact completed `dry_run` authorizes at most one real run, DB-enforced by FK + guard trigger + partial unique index). Explicit four-status eligibility (`ready_for_ai`/`generating`/`review`/`published`); durable `canonical_legacy_backfill_runs` + immutable per-run outcome ledger; keyset batching + resumability; owner/co-owner authorization; run identity/source immutability + a strict run state machine (a failed run accepts no outcomes until explicitly resumed; forged completed runs and falsified `completed_at` are rejected); finish/record serialization (`FOR UPDATE`). **Backfill remains identity-only.**

I2 and I3 **create ProductConcept/SKU/SourceRecord identities only** — no CommercePackages, Claims, photos/evidence records, Phase H queue entries, publishing changes, or later Phase I work were created.

I4 delivered: **an approved legacy photo set now gains permanent canonical PhotoAsset records**, and the bridge is fully durable and replay-safe. What shipped (verified against the merged migration `20260808200000_phase_i_slice4_approved_photo_bridge.sql`, `lib/canonical/bridge/processor.ts`, `app/photos/actions.ts`):

- **Approval and durable photo-job enqueue are atomic** — `gmcom_mark_photo_set_approved` performs the guarded `needs_review → approved` transition and enqueues one durable `photo_sets` bridge job in a single transaction (concurrent-change guard; idempotent enqueue). **A later processing failure never reverses the legacy approval.**
- **One canonical PhotoAsset per original legacy photo** (`canonical_photo_assets`), preserving `storage_ref` (the original path) and `original_content_hash`; **no files are copied or moved** — only paths/hashes are recorded.
- **Derivatives remain linked representations** in the new `canonical_photo_asset_derivatives` table (one row per legacy `photo_derivatives` row, environment-scoped RLS), not separate identities.
- **Canonical approval remains `pending`** — the generic `createEntity` cannot fabricate `owner_approval_state='genuine'`, so the legacy approved photo set stays the legacy source of truth.
- **Permanent per-photo mappings exist** — `canonical_legacy_entity_bridge.legacy_table` was additively widened to allow `photo_assets`, one bridge row per canonical PhotoAsset with `legacy_approved_at`/`legacy_approved_by` provenance; the run/outcome ledger stays separate operational history.
- **Replay validates the full state** — `gmcom_bridge_photo_set` is idempotent and concurrency-safe (deterministic lock order; two-session race proven), and replay verifies `record_purpose`, approval state, linkage, storage/hash drift, full derivative state (missing/extra/duplicate), and **one shared correlation** across assets, mappings, and derivatives; partial mappings are never silently repaired (mismatch fails visible).
- **Unsupported bridge jobs fail closed** — the processor routes `products` → the identity RPC and `photo_sets` → the photo RPC, and any other `legacy_table` is never routed and finishes as a permanent `failed`/`bridge_unsupported` job (unit-tested for `listing_packages`, `commerce_details`, and unknown tables).

**I4 exclusions (all hold):** no historical photo backfill; no Claims or Evidence; no CommercePackage; no Phase H queue population; no publishing changes; no later Phase I work.

> **CI workflow-size incident and permanent correction.** PR #57's branch CI produced no runs because `.github/workflows/ci.yml` had grown to 529,068 bytes — above GitHub's 512,000-byte per-workflow file limit, which rejects the workflow **before any run is created** (silent; the runner and repo were fine). Fix: the large Phase I live-Postgres SQL heredocs and race scripts were extracted from `ci.yml` into checked-in `supabase/live/` files (invoked via short `psql -f` / `bash` steps, preserving exact order/assertions/env), and a structural test (`supabase/ci-workflow-size.test.ts`) fails any workflow YAML file at or above the project's 450 KB guard. `ci.yml` is now **414,483 bytes**; the fix was verified end to end before merge.

> Post-merge CI for `e58766e` (I4) verified green: CI workflow run `31307655608` = success (independently pulled on 2026-08-09; Copilot run `31307656060` also success). Prior post-merge runs: `285a2a0` → `31281957101` (CI) success; `bef5a5d` → `31262417220` (CI) / `31262419177` (Copilot) success; `2c26b61` → `31258503603` (CI) success.

Owner-approved decisions (2026-08-08, full list in `phase-i-slice-plan.md`): identity creation triggered at `products.status='ready_for_ai'`; a new `canonical_legacy_entity_bridge` table (not a widening of `canonical_legacy_field_bridge`); ProductConcept + SKU + SourceRecord + mapping created atomically by one RPC; legacy never rolls back on bridge failure; a durable bridge job/outbox (`pending/processing/done/failed/mismatch/retry` + `correlation_id` + error context); ongoing production bridging activated before backfill; archived rows initially skipped and recorded as excluded (no invented retention); no AI-content Claims until a defensible SourceCategory/evidence design is approved; **I1/I2/I3/I4 do not populate the Phase H queue** (only a canonical CommercePackage does); CommercePackage timing is a later decision; the bridge architecture is approved as the next phase.

## Phase H — Read-Only Completed-Review Refinement (complete)

Phase H surfaced what Phases B–G actually produced as read-only context in the review shell. The authoritative slice plan is `phase-h-slice-plan.md` (this repo).

`gm-commerce/main` is at `c65b0232d834c200195b38ce992d1e42272954d1` (merge of PR #53 / H8).

| Slice | Scope | PR | Status | Merge |
|---|---|---|---|---|
| H1 | Canonical commerce → read-only `ReviewPackage` adapter | #42 | Merged | `f44ceb2` |
| H2 | Canonical loader + precedence/contradiction preservation + detail routing | #43 | Merged | `9540a1a` |
| H3 | Operational-only commerce queue discovery | #44 | Merged | `d3cba35` |
| Phase E prerequisite correction | Operational-only compliance current-state/gate derivation | #45 | Merged | `1472660` |
| H4 | Read-only Phase E compliance context on commerce detail | #47 | Merged | `606c5a5` |
| H5 | Read-only Phase C evidence-library context | #48 | Merged | `4b5ff6a` |
| Phase F prerequisite correction | Operational-only recommendation current-state derivation | #49 | Merged | `372ee23` |
| H6 | Read-only Phase F recommendation + §18 SelectionTrace context | #50 | Merged | `905d658` |
| H7 | Read-only Phase D vision-analysis context | #51 | Merged | `7f5a153` |
| Phase G prerequisite correction | Operational-only policy/rule read paths | #52 | Merged | `23c4e39` |
| H8 | Read-only Phase G policy + learned-rule context | #53 | Merged | `c65b023` |

All commerce capabilities remain `false`; Phase H is strictly read-only.

> Historical note: the earlier "Phase H remains in progress / next proposed work: H5" guidance is superseded. Phase H is fully complete (H1–H8 + the Phase E/F/G prerequisite corrections). The current next phase is Phase I (above).

**Open issue:** [#46 — Reconcile Phase E compliance gate functions and approval trigger into schema.sql](https://github.com/HydraCoreSystems/gm-commerce/issues/46) (open). Ordered migrations remain the deployment source of truth; this does not block Phase I.

## Phase G — Owner-Editable Policies and Learned-Rule Activation (complete; superseded "next task" guidance is historical)

Phase G enabled editing policies and confirming Phase F recommendations into standing learned rules. The authoritative slice plan is `phase-g-slice-plan.md` (this repo).

| Slice | Scope | PR | Status |
|---|---|---|---|
| G1 | Policy management — create, query, evaluate policies | #30 | Merged |
| G2 | Owner confirmation flow — recommendation → owner decision | #31 | Merged |
| G3 | Correction capture and scope inference | #32 | Merged |
| G4 | Rule activation engine — learned rules table, activate/revoke/query | #33 | Merged |
| G5 | RBAC enforcement — §15 role matrix at Repository command layer | Not started |

> Historical note: the earlier "Next Phase: Phase G" guidance is superseded. Phase G is complete through G4; G5 (RBAC enforcement) remains a later-phase item. The current next phase is Phase H (above).

## Phase F (complete)

All seven core recommendation services merged in PRs #23–#29. Full closeout in `phase-f-closeout-report.md`.

| Slice | PR | Service | Kind |
|---|---|---|---|
| F1 | #23 | SelectionTrace foundation | infrastructure |
| F2 | #24 | Price | `price` |
| F3 | #25 | Taxonomy + Collections | `taxonomy` / `collections` |
| F4 | #26 | Marketplace suitability | `marketplace_suitability` |
| F5 | #27 | Photography | `photography` |
| F6 | #28 | SEO | `seo` |
| F7 | #29 | Merchandising | `merchandising` |

See `phase-f-slice-plan.md` for per-slice scope, acceptance criteria, and exclusions.

## Product Reset

`PRODUCT_RESET_2026-08-03.md` is at Revision 3. Copilot's re-review returned `REQUIRES REVISION 3 AND RE-REVIEW`; Revision 3 incorporated the corrections; Codex subsequently returned `APPROVE AFTER DOCUMENTED CORRECTIONS`. Per §24, the review step is complete.

## Schema State

- I1 merged (PR #54, `2c26b61`): `canonical_legacy_entity_bridge` table + `gmcom_bridge_product_identity` RPC added, with the consolidated `schema.sql` mirror updated in the same merge (applies clean from empty on PG15; 36 ordered migrations).
- I2 merged (PR #55, `bef5a5d`): 37th migration adds `canonical_legacy_bridge_jobs` (durable per-`(environment, legacy_table, legacy_key)` job queue with lease/backoff/dead-letter machinery) + immutable `canonical_legacy_bridge_job_attempts` ledger + `gmcom_mark_product_ready_for_ai`/`gmcom_enqueue_legacy_bridge_job`/`gmcom_claim_legacy_bridge_job`/`gmcom_finish_legacy_bridge_job` RPCs; consolidated `schema.sql` mirror updated; live PG15 coverage added to the `phase-b0-slice1-review-shell-live` CI job.
- I3 merged (PR #56, `285a2a0`): 38th migration redefines `gmcom_bridge_product_identity` with the explicit four-status eligibility allowlist and adds `canonical_legacy_backfill_runs` + immutable per-run `canonical_legacy_backfill_row_outcomes` + `source_dry_run_id` single-use linkage (FK + guard trigger + partial unique index) + `gmcom_legacy_backfill_inspect`/`start_run`/`resume_run`/`record_outcome`/`finish_run` RPCs + run integrity/state-machine guard; consolidated `schema.sql` mirror updated; live PG15 + two-session race coverage added to `phase-b0-slice1-review-shell-live`.
- I4 merged (PR #57, `e58766e`): 39th migration additively widens `canonical_legacy_entity_bridge.legacy_table` CHECK to allow `photo_assets`, adds `original_content_hash` to `canonical_photo_assets`, creates `canonical_photo_asset_derivatives` (linked representations, RLS env-scoped), adds `legacy_approved_at`/`legacy_approved_by` provenance columns to the bridge, and adds `gmcom_mark_photo_set_approved` (atomic guarded approval + durable enqueue) + `gmcom_bridge_photo_set` (deterministic lock order; one PhotoAsset per original; full replay validation of purpose/approval/storage/linkage/derivatives/correlation); `lib/canonical/bridge/processor.ts` generalized to fail-closed dispatch; consolidated `schema.sql` mirror updated; live PG15 (20 checks) + two-session race coverage added to the `phase-b0-slice1-review-shell-live` CI job (I4 steps run within that job, alongside I1–I3). Also in this merge: the CI workflow-size correction (large Phase I live SQL/race bodies extracted to `supabase/live/`; `ci.yml` 529,068 → 414,483 bytes; `supabase/ci-workflow-size.test.ts` 450 KB guard).
- Migration gap resolved: `commerce_details` table and `listing_packages.seo_title`/`seo_description` columns have committed DDL at `20260802040000_commerce_readiness.sql`.
- `20260803000000_commerce_field_ownership.sql` (price/`content_provenance`) was deliberately retired as never-applied.
- CI schema-from-empty passes from committed migrations.
- `HY-LOB01-C04` test data verified absent from the live database (2026-08-05).
- **Issue #46 open:** consolidated `schema.sql` omits the Phase E compliance gate functions/trigger (`gmcom_compliance_check_is_stale`, `gmcom_current_compliance_check_status`, `gmcom_compliance_gate_outcome`, `gmcom_compliance_checks_with_status`, `gmcom_guard_commerce_package_approval` + trigger and dependencies). Ordered migrations are the deployment source of truth; reconciliation is tracked, not blocking.

## Completed (pre-Phase F)

- **GMCOM-012** — Shopify Draft Publisher, verified against a real store (`gid://shopify/Product/10220386648384`). Full detail in `handoffs/2026-08-02-GMCOM-012-shopify-draft-publisher.md`.
- **GMCOM-013/014** — Listing Audit Engine and Commerce Audit workspace on `hydracoresystems-listing-audit-engine`. Real-export validation pending Phil providing a current Shopify CSV.
- **GMCOM-016** — Autonomous Commerce Intelligence Audit (documentation only) on `hydracoresystems-listing-audit-engine`.
- **GMCOM-011** — Photo preparation and approval pipeline. Migration applied, verified live via GMCOM-012.
- **GMCOM-009/010** — AI generation hardened (atomic locking, versioning, runtime validation); migrations, CI, operational baseline.
- **GMCOM-008** — Listing Quality Engine (multi-stage pipeline, quality summaries).
- **GMCOM-007** — AI Listing Package generator + Provider abstraction.
- **GMCOM-006** — Guided intake experience.
- **GMCOM-004** — Verified end-to-end with real cutting (`HY-LOB01-C04`).
- **GMCOM-003** — Skrybix commerce selection handoff, production-deployed.
- **GMCOM-002** — GM Commerce application foundation.
- **GMCOM-001** — Product SKU Generator baseline.
- **Phase C Slice 5** — Freshness, Revalidation, and Promotion Gates (PR #14, merged). _Historical — superseded by later phases; retained as a completed record._
- **Phase D Slice 1** — Vision-provider contract and authorized-media-access boundary (built, on branch `agent/phase-d-slice-1-vision-provider`, not yet merged).
- **Phase E Slices 1–4** — ComplianceCheck, deterministic validation, fail-closed gate, review surface (PRs #19–#22, all merged).

Supabase project: `wcrcllhvgbhykbonopzx` (separate co-owner account).

## Active Work

- **Phase I: I1, I2, I3, and I4 delivered and merged (PR #54 `2c26b61`, PR #55 `bef5a5d`, PR #56 `285a2a0`, PR #57 `e58766e`).** The authoritative multi-slice plan is `phase-i-slice-plan.md`; the master completion map is `COMPLETION.md`. I1 established `canonical_legacy_entity_bridge` + the atomic `gmcom_bridge_product_identity` RPC; I2 wired every real `ready_for_ai` transition to a durable identity-bridge job; I3 delivered existing-identity backfill (dry-run → single-use real-run, immutable outcome ledger, run integrity/state machine); I4 delivered the approved-photo bridge (atomic approval+enqueue; one canonical PhotoAsset per original; derivatives as linked representations; canonical approval stays `pending`; durable per-photo mappings with provenance; full replay validation including one shared correlation; fail-closed processor dispatch; no files copied/moved) and the CI workflow-size correction (extracted Phase I live tests to `supabase/live/`; `ci.yml` 414,483 bytes; 450 KB guard test). **The likely next slice (I5: historical approved-photo backfill using the I4 bridge machinery) is a design recommendation only and is NOT authorized** (see `phase-i-slice-plan.md`). All of I1–I4 create ProductConcept/SKU/SourceRecord identities and canonical PhotoAssets only — no CommercePackages, Claims, evidence records, Phase H queue entries, or publishing changes.
- **Phase H is COMPLETE** (H1–H8 + Phase E/F/G prerequisite corrections, merged at `c65b023` per PR #53). No further Phase H slices are proposed.
- **Owner-confirmed output requirement (2026-08-09):** the review/publishing workflow must offer **Shopify, Etsy, and a permanent master Listings Spreadsheet** as **mandatory operational destination choices**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** Etsy remains scheduled under **ROADMAP Milestone 4, now classified as required for overall GM Commerce completion** (not optional). The Listings Spreadsheet is append-only and idempotent, written through a **durable export job/outbox with an immutable export ledger** (never claimed atomic with the database), configured from trusted server credentials only, and is **not** a per-listing downloadable file — CSV download is optional backup only. Overall completion requires a **proven working path for each of the three choices**, demonstrated by completion testing that routes an **approved canonical CommercePackage through Shopify, through Etsy, and through the Listings Spreadsheet without Phil or Crystal rewriting or reformatting the approved content**. Recorded as decisions 12–13 in `phase-i-slice-plan.md` and in `COMPLETION.md`; **not implemented and not authorized to implement yet**.
- **GitHub Issues**: Several tasks lack formal Issues. GitHub API access has been intermittent. Issue #46 remains open (see Schema State).
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Phase

**Phase I — Legacy-to-Canonical Bridge.** I1 (PR #54, `2c26b61`), I2 (PR #55, `bef5a5d`), I3 (PR #56, `285a2a0`), and I4 (PR #57, `e58766e`) are delivered and merged. **The proposed next slice (I5: historical approved-photo backfill using the I4 bridge machinery) is a design recommendation only and is NOT authorized.** Then the later Phase I slices (CommercePackage creation, content assembly, claims mapping, drift hardening) each require their own design and — for claims — an explicit SourceCategory/evidence owner decision. See `COMPLETION.md` for the full finish-line map, remaining-slice sequence, and exit criteria.

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | Phase I plan/status documentation |
| Claude | Primary coordination + implementation + review | Available | Phase I: I1 (PR #54) + I2 (PR #55) + I3 (PR #56) + I4 (PR #57) delivered; I5 design recommendation + master completion map documented |
| GitHub Copilot | Implementation contributor | Quota-limited | Phase I bridge foundation (I1) |
| Phil | Product owner | Available as schedule permits | Next Phase I slice authorization |

Capacity status should be updated whenever a provider limit is reached or resets.
