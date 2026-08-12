# Phase I Slice Plan — Legacy-to-Canonical Bridge

_Historical/phase plan for the legacy-to-canonical bridge. Based on the read-only design inventory (see `handoffs/2026-08-08-phase-h-complete-and-legacy-canonical-bridge-handoff.md` and its superseding addenda). The master completion map is `COMPLETION.md` (this repo). **For current status, see `STATUS.md`; this document is retained as the historical record of the plan and its execution.** Last updated: 2026-08-12._

> **Slice status: Phase I is COMPLETE as implemented.** **I1–I5, CommercePackage creation, content assembly, truthful claims mapping, drift monitoring, review-shell display, atomic review decisions, canonical destination routing/outbox, and the three destination adapters (Listings Spreadsheet, Shopify, Etsy) are all delivered and merged** — PR #54 (`2c26b61`), PR #55 (`bef5a5d`), PR #56 (`285a2a0`), PR #57 (`e58766e`), PR #58 (`cc19611`), PR #59 (`a4db76f`), PR #60 (`38519f`), PR #61 (`ca613e2`), PR #62 (`8f60120`), PR #63 (`a0e426a`), PR #64 (`ee76dcd`), PR #65 (`65be13b`), PR #66 (`3721fe`), PR #67 (`19b562`), PR #68 (`f6e24bb`) on `gm-commerce/main`. "Implemented" is not "launch-ready": see `COMPLETION.md` for the launch conditions that remain. No further Phase I slice is proposed. Phase 0 Slices 1–3 were completed through application PRs #69–#71 (see `STATUS.md` for the authoritative current status).

## Purpose

Bridge the real, working production pipeline (GMCOM-001–012: Skrybix / Product-SKU-Generator selection → SKU-named photo folder → human photo confirmation → AI listing generation → Shopify publish), which today writes **only legacy tables** (`products`, `listing_packages`, `photo_sets`, `commerce_details`), into the canonical model (`canonical_product_concepts`, `canonical_skus`, `canonical_source_records`, `canonical_commerce_packages`, `canonical_claims`, …) that Phases B–H were built around.

The **architecture itself is approved by the owner as the next phase** (owner decision 11). Slices are independently testable and mergeable. **All Phase I slices are delivered and merged** (PRs #54–#68; see the status note at the top of this document).

## Critical finding this phase addresses

The canonical identity spine has **no live population path**: code search confirms zero non-framework writers to `canonical_product_concepts` / `canonical_skus` / `canonical_commerce_packages` / `canonical_claims` from the real intake pipeline, and the owner confirmed no person/script/seed process creates canonical production records manually. Phase H's entire review surface therefore has nothing real to show in production until this bridge exists.

## Dependency

Phase I depends on:

- **Phase B** (merged): the 18 canonical entity tables, `RecordContext` (§25), and the generic `CanonicalEntityRepository` (`createEntity`/`getEntity`/`listEntities`, environment-bound).
- **Phases C–H** (merged): claims/evidence, compliance, recommendations, vision, policy/rules, and the read-only review surface.
- **GMCOM-001–012** (verified): the legacy production pipeline this phase bridges from.

## Approved owner decisions (2026-08-08 — supersede any earlier proposed-I1 language)

1. Canonical identity creation is triggered when a legacy `products` row reaches `status='ready_for_ai'`.
2. Create a new `canonical_legacy_entity_bridge` table. Do **not** widen or repurpose `canonical_legacy_field_bridge`.
3. ProductConcept + SKU + SourceRecord + entity-bridge mapping must be created **atomically by one database RPC**.
4. The legacy pipeline must **not roll back** because canonical bridging fails.
5. A durable bridge job/outbox must record `pending`, `processing`, `done`, `failed`, `mismatch`, `retry` information, `correlation_id`, and error context.
6. **Activate ongoing production bridging before backfilling** existing rows.
7. Initially **skip archived rows and record them as excluded**; do not invent retention behavior.
8. Do **not** create canonical Claims for AI-generated listing content until a defensible SourceCategory/evidence design is explicitly approved.
9. **I1 does not make anything visible in the Phase H queue.** That requires a canonical CommercePackage in a later slice.
10. CommercePackage creation timing remains a later design decision (likely generation finalization / entry into review), but is **not** authorized as an I1 assumption.
11. The legacy-to-canonical bridge architecture itself is approved as the next phase.
12. **Review/publishing output destinations (2026-08-09, owner-confirmed — replaces any proposed "downloadable-file" third-output framing):** the review/publishing workflow must offer **three mandatory operational destination choices — Shopify, Etsy, and Listings Spreadsheet**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** The Listings Spreadsheet is **one permanent master Google Sheet**; selecting it **appends one new row to the same configured sheet** and must **not create a new spreadsheet per listing**. The user can identify the **intended external destination** (e.g. **Facebook, Palm Street, Whatnot, auction, website, email, or other**). The sheet is **append-only operational output** — existing rows are never overwritten. Each row preserves the canonical **CommercePackage ID and version**, **export ID**, **timestamp**, **trusted actor**, **intended destination**, **approval state**, **listing fields**, **price recommendation**, and **photo references**. **Idempotent**: retries/double-clicks create no duplicate rows; a later **intentional** export of a newer package version may create a new row. The **Google Sheet ID and credentials come from trusted server configuration** and never reach the browser. Because a Google Sheets write **cannot share a database transaction**, implementation must use a **durable export job/outbox** with retry, idempotency, failure reporting, and an **immutable export ledger** — the database update and the Google Sheet update are **never one atomic transaction** and that must not be claimed. **CSV download is optional backup functionality only, not the primary requirement.** Recorded as a confirmed requirement; **not implemented and not authorized to implement yet** (do not claim the DB and Google Sheet update are one atomic transaction).
13. **Etsy is required for overall completion (2026-08-09, owner-confirmed):** Etsy remains scheduled under **ROADMAP Milestone 4**, but M4 is now explicitly classified as **required for overall GM Commerce operational completion** — not optional future work. Overall completion requires a **proven working path for each of the three mandatory destination choices (Shopify, Etsy, and the Listings Spreadsheet)**; completion testing must demonstrate that an **approved canonical CommercePackage can be routed successfully through Shopify, through Etsy, and through the Listings Spreadsheet without Phil or Crystal rewriting or reformatting the approved content**. A listing is **not** required to be sent to all three destinations simultaneously; for each approved listing Phil or Crystal chooses the intended route.

## Invariants (hold across every slice)

- **Legacy remains the working source pipeline** during bridging; bridging never rolls back or mutates the legacy transition that succeeded.
- Canonical writes are **operational-purpose** (`record_purpose='operational'`) and **environment-bound** (the bound `GMCOM_ENVIRONMENT`; never request-derived, never cross-environment).
- **No fabricated** approval, eligibility, evidence, provenance, claims, or authority (`createEntity` rejects privileged context; `rejectPrivilegedCreationContext` blocks genuine approval / true eligibility at the repository boundary).
- **No cross-environment correlation** (composite `(id, environment)` FKs; bridge mapping is environment-scoped).
- **No reliance on SKU text as the sole permanent mapping** — a permanent `canonical_legacy_entity_bridge` mapping row is the durable correlation; `sku_code` is a convenience index, not the source of truth.
- **No commerce mutations or publishing work** is pulled into the Phase I identity slices.
- **No AI-content Claim mapping** without a later explicit owner decision.

## Correction of an earlier design-report statement

An earlier draft stated ProductConcept + SKU creation "makes Phase H's review surface show something real." That is **false**: the Phase H queue (`/review-shell`) lists canonical **CommercePackages**, not SKUs. **Only a canonical CommercePackage can enter the Phase H queue.** Identity (ProductConcept/SKU) is required groundwork, but I1 alone makes nothing visible in Phase H.

## Slice inventory

| Slice | Scope | Status | Base SHA |
|---|---|---|---|
| I1 | Bridge foundation + atomic identity RPC (ProductConcept + SKU + SourceRecord + mapping) | **Merged — PR #54, `2c26b61`** | On `main` |
| I2 | Ongoing `ready_for_ai` integration + durable bridge jobs | **Merged — PR #55, `bef5a5d`** | On `main` |
| I3 | Existing identity backfill (re-runnable, no duplicates, excluded-row recording) | **Merged — PR #56, `285a2a0`** | On `main` |
| I4 | Approved photo sets → canonical PhotoAsset (attached assets only; atomic approval+enqueue; durable per-photo mappings; replay-validating bridge RPC; fail-closed dispatch) | **Merged — PR #57, `e58766e`** | On `main` |
| I5 | Historical approved-photo backfill (reuse I4 bridge machinery + I3 backfill pattern) | **Merged — PR #58, `cc19611`** | On `main` |
| CommercePackage creation | Canonical CommercePackage at generation finalization | **Merged — PR #59, `a4db76f`** | On `main` |
| CommercePackage content assembly | Exact-version listing content, photo refs, field lineage | **Merged — PR #60, `38519f`** | On `main` |
| Truthful claims mapping | AI-content claims + field lineage (`under_review`, no fabrication) | **Merged — PR #61, `ca613e2`** | On `main` |
| Drift monitoring | Bounded resumable read-only scans, no repair | **Merged — PR #62, `8f60120`** | On `main` |
| Review-shell display | Assembled canonical package content in the review shell | **Merged — PR #63, `a0e426a`** | On `main` |
| Atomic canonical review decisions | Approve/reject atomicity, retry, rollback, fail-closed | **Merged — PR #64, `ee76dcd`** | On `main` |
| Canonical destination routing + outbox | Durable enqueue, lifecycle, ledger, idempotency, eligibility | **Merged — PR #65, `65be13b`** | On `main` |
| Listings Spreadsheet adapter | Durable export ledger, lease-protected claim, atomic completion | **Merged — PR #66, `3721fe`** | On `main` |
| Canonical Shopify draft adapter | Lease-protected claim, atomic marketplace success, immutable attempts | **Merged — PR #67, `19b562`** | On `main` |
| Canonical Etsy draft adapter | Create-intent-before-remote-call, `needs_confirmation` parking, single-use recreate | **Merged — PR #68, `f6e24bb`** | On `main` |

**All Phase I slices are merged.** No further Phase I slice is proposed. Phase 0 Slices 1–3 were completed through application PRs #69–#71 (see `STATUS.md` for the authoritative current status); the next implementation work is automatic background processing and the remaining launch conditions, followed later by dependency-safe legacy cutover.

---

## I1 — Bridge foundation and atomic identity RPC

> **Status: delivered and merged.** PR #54, merge SHA `2c26b61` on `gm-commerce/main`. The sections below are the authoritative spec of what shipped. Implementation facts (verified against the merged migration `20260808090000_phase_i_slice1_identity_bridge_foundation.sql`): the RPC is `gmcom_bridge_product_identity(p_environment, p_sku, p_correlation_id)` returning `(concept_id, sku_id, source_record_id, status)`; the bridge table uses statuses `pending/processing/done/failed/mismatch/retry/excluded` with `correlation_id`, `retry_count`, `error_context`, `first_bridged_at`, `last_bridged_at`, and a UNIQUE `(environment, legacy_table, legacy_key, canonical_entity_type)`; a two-session concurrency test was added to the `phase-b0-slice1-review-shell-live` CI job and passes (both simultaneous calls return the same identity set; exactly one identity set and three bridge rows remain). **I1 still does not create CommercePackages and does not populate the Phase H queue.** Post-merge `main` CI for `2c26b61` was still queued at the time of the status update (verify independently).

**Objective.** Establish the bridge substrate and the single atomic operation that turns an eligible legacy product into canonical identity (ProductConcept + SKU + SourceRecord) with a permanent mapping — no production action integration, no backfill, nothing visible in the Phase H queue.

**Authoritative trigger.** The existence of a legacy `products` row with `status='ready_for_ai'` passed to the I1 RPC. I1 itself is not yet wired to any production action; callers (initially tests; later I2/I3) invoke it.

**Tables / RPCs / application paths affected.**
- New table `canonical_legacy_entity_bridge` (environment, legacy_table, legacy_key, canonical_entity_type, canonical_entity_id, status, retry_count, error_context, correlation_id, first_bridged_at, last_bridged_at; UNIQUE on the correlation tuple; RLS + grants + env-isolation policy mirroring canonical conventions; a durable job/outbox flavor per owner decision 5).
- New RPC (e.g. `gmcom_bridge_product_identity`) that **atomically** creates, in dependency order, `canonical_product_concepts` → `canonical_skus` → `canonical_source_records` → the `canonical_legacy_entity_bridge` mapping row. Environment-bound (`p_environment` from trusted config pattern), operational-only.
- No change to any legacy writer, no `gm-commerce` app code change in I1 beyond the migration + RPC + tests.

**Transaction boundaries.** One database transaction for the four-row canonical write + bridge mapping. The legacy `products` row is **read**, never written. On any failure, the entire canonical transaction rolls back and the bridge job records `failed` (owner decision 4: legacy never rolls back — here there is no legacy write at all).

**Idempotency and retry rules.** Get-before-create on `canonical_skus` UNIQUE `(sku_code, environment)` and on the bridge mapping UNIQUE; a unique violation means "already bridged" → reconcile (return existing, mark `done`) rather than duplicate. `retry_count` and `error_context` are recorded; a re-run of the RPC for the same legacy key is a no-op success when already `done`.

**Security / environment / record_purpose rules.** RPC runs as `service_role`; writes `record_purpose='operational'` on the four canonical rows (bridge job row may use `'operational'`/`'migration'` per its bookkeeping role); environment = the bound trusted environment only; no cross-environment lookups; no fabricated approval/eligibility (via `rejectPrivilegedCreationContext` semantics inside the RPC or an equivalent guard).

**Test and live-Postgres requirements.** Unit tests on the RPC's guards and idempotency with the fake-Supabase pattern; live-Postgres job (extend an existing Phase B live job, do not create a redundant one): fresh eligible `products` row → RPC → concept+sku+source_record+bridge row exist, `record_purpose='operational'`, concept-before-sku FK satisfied; re-run → no duplicates; ineligible row (not `ready_for_ai`) → rejected; cross-environment id never resolved; failure path → `failed` bridge row + full canonical transaction rolled back.

**Entry criteria.** Owner approved I1 (this slice spec). **Exit criteria.** The atomic RPC exists, is idempotent, environment-bound, operational-only, and passes live-Postgres; the `canonical_legacy_entity_bridge` substrate is in place; **Phase H queue is unchanged** (no packages appear).

**Explicit exclusions.** Any production action wiring; backfill; photo/claim/CommercePackage creation; retention behavior (archived handling is recorded, not acted on); commerce mutations or publishing.

**Dependencies.** None (needs only Phase B canonical substrate + the new migration).

**Unresolved decisions blocking I1** (none block I1; recorded for I2/I3): exact RPC signature / where the RPC lives; whether the bridge job row's `record_purpose` is `'operational'` or a bookkeeping value; whether `canonical_source_records` should be created in I1 or deferred (recommended: include it, per owner decision 3).

---

## I2 — Ongoing `ready_for_ai` integration

> **Status: delivered and merged.** PR #55, merge SHA `bef5a5d` on `gm-commerce/main`. The sections below are the authoritative spec of what shipped. Implementation facts (verified against the merged migration `20260808120000_phase_i_slice2_ready_for_ai_bridge_jobs.sql`): a SEPARATE durable `canonical_legacy_bridge_jobs` queue (per `(environment, legacy_table, legacy_key)`, statuses `pending/processing/done/failed/mismatch/retry/dead_letter`, lease columns, `attempt_count`/`max_attempts`, `available_at` backoff, `correlation_id`, `error_context`, status-shape CHECK) + immutable `canonical_legacy_bridge_job_attempts` ledger; `gmcom_mark_product_ready_for_ai` (guarded transition + enqueue in one transaction; enqueue failure rolls back the transition, later canonical failure never reverts it; no-op replay) + idempotent `gmcom_enqueue_legacy_bridge_job` + `gmcom_claim_legacy_bridge_job` (`FOR UPDATE SKIP LOCKED`, 300s lease, lease-expiry recovery, retry re-claim, dead-letter exhaustion) + `gmcom_finish_legacy_bridge_job` (lease-token validated, 60s→1h exponential backoff). Request-triggered processing via `app/actions.ts` `markReadyForAI` (best-effort, failure-isolated) and the manual drain `drainBridgeJobs` server action, which is **authorized owner/co_owner-only** via `lib/auth` `resolvePrincipal`/`requireRole` (staff/service rejected; no new permission invented; button hidden on `/` when not authorized). Post-merge `main` CI for `bef5a5d` verified green (run `31262417220`, success).

**Objective.** Make the legacy `ready_for_ai` transition and a durable bridge-job enqueue **atomic on the legacy side**, then process the bridge job using the I1 machinery — without rolling back the successful legacy transition when bridging fails (owner decision 4).

**Authoritative trigger.** `markReadyForAI` (or an equivalent path) successfully flips `products.status` to `ready_for_ai`.

**Tables / RPCs / application paths affected.** `app/actions.ts` `markReadyForAI` (or a new RPC that atomically performs the legacy update + bridge-job insert); the durable bridge outbox (I1 substrate, consumed by a processor); the I1 identity RPC (invoked by the processor); `canonical_legacy_entity_bridge`.

**Transaction boundaries.** Legacy update + outbox enqueue in **one** transaction. The outbox processor runs separately: reads a `pending` job → `processing` → invokes the identity RPC → `done` (or `failed`/`retry` with `error_context`, `retry_count`, `correlation_id`). A failed bridge job never rolls back the legacy transition.

**Idempotency and retry rules.** Outbox rows are idempotent (a `done` row is not reprocessed; a `failed` row is retried with backoff up to a cap, then requires reconciliation). Processing reuses I1's get-before-create, so a retry after a partial/failed run converges.

**Security / environment / record_purpose rules.** Processor runs as `service_role`, environment-bound, operational-only; same invariants as I1.

**Test and live-Postgres requirements.** Unit tests on the outbox state machine (pending/processing/done/failed/retry/mismatch) and the atomic legacy+enqueue; live-Postgres: `ready_for_ai` transition creates the outbox row; the processor creates canonical identity; an injected canonical failure leaves `products.status='ready_for_ai'` intact and the outbox row `failed`/retryable.

**Entry criteria.** I1 merged; owner approves the `ready_for_ai` integration wiring. **Exit criteria.** Every new `ready_for_ai` product automatically gains canonical identity via the outbox; legacy pipeline continues to work through bridge failures; `done` jobs are idempotent.

**Explicit exclusions.** Backfill of existing rows (I3); claims/photos/CommercePackage; changing the legacy transition's own behavior beyond adding the enqueue.

**Dependencies.** I1.

**Unresolved decisions blocking I2 but not I1.** Whether `markReadyForAI` is modified directly or a new RPC wraps it; outbox consumption trigger (poll vs a `pg_notify`/webhook) and retry schedule; whether the enqueue is a DB RPC or a TS-side insert in the same request.

---

## I3 — Existing identity backfill

> **Status: delivered and merged.** PR #56, merge SHA `285a2a0` on `gm-commerce/main`. The sections below are the authoritative spec of what shipped. Implementation facts (verified against the merged migration `20260808150000_phase_i_slice3_identity_backfill.sql` and `lib/canonical/bridge/backfill.ts`): `gmcom_bridge_product_identity` is redefined with the explicit four-status eligibility allowlist (`ready_for_ai`/`generating`/`review`/`published`, fail-closed otherwise); `canonical_legacy_backfill_runs` + immutable per-run `canonical_legacy_backfill_row_outcomes` (UNIQUE `(run_id, environment, legacy_table, legacy_key)`, append-only trigger, SQL-constrained outcome/reason vocabulary); `source_dry_run_id` single-use dry-run→real-run linkage (NULL-safe CHECK, composite FK for same-environment, guard trigger requiring a completed dry_run, partial unique index = one real run per dry run, no freshness rule); run identity/source immutability + a strict run state machine (INSERT must be `running`/no `completed_at`/no cursor/empty counters; running/failed have `completed_at NULL`, completed has `completed_at SET`; a failed run accepts no outcomes until explicitly resumed; `completed` is terminal); `record_outcome`/`finish_run` serialize via `FOR UPDATE`; owner/co-owner authorization via `runLegacyBackfillDryRun`/`promoteLegacyBackfill(dryRunId)`; keyset batching + per-row cursor persistence + resumability. Post-merge `main` CI for `285a2a0` verified green (run `31281957101`, success). **Backfill is identity-only** — no CommercePackages, Claims, photos/evidence, or Phase H queue work.

**Objective.** Re-runnable backfill of existing eligible `products` rows (at/after `ready_for_ai`) using the same I1 bridge machinery — no duplicates, archived/ineligible/malformed rows skipped and recorded as excluded.

**Authoritative trigger.** An explicit operator-invoked backfill run (script + RPC), run **after** I2's ongoing path is live (owner decision 6).

**Tables / RPCs / application paths affected.** `canonical_legacy_entity_bridge`; the I1 identity RPC; a backfill driver (RPC or script) that pages eligible `products` rows.

**Transaction boundaries.** Each row bridged by the I1 atomic RPC; the backfill driver records per-row outcome (`done` / `excluded` / `mismatch`). Rows are read-only; legacy is never mutated.

**Idempotency and retry rules.** Re-runnable: already-`done` rows no-op; a partial run resumes cleanly; the bridge UNIQUE prevents duplicates. Excluded rows are recorded once (e.g. `status='excluded'` with reason) and skipped on re-runs.

**Security / environment / record_purpose rules.** Same as I1/I2; backfill may use `record_purpose='migration'` on the bridge bookkeeping row while the canonical entity rows are `'operational'` (drift/mismatch rows keep `mismatch` status).

**Test and live-Postgres requirements.** Live-Postgres against a seeded legacy dataset: eligible rows bridged; `intake`/no-photo rows skipped; `archived` rows recorded as `excluded` (owner decision 7 — no invented retention); malformed rows recorded; no duplicates across two runs; drift/mismatch reporting (a legacy value that changed after bridging → `mismatch`).

**Entry criteria.** I2 merged; owner approves backfill. **Exit criteria.** All eligible existing products have canonical identity; every skipped/malformed row is recorded; re-runs are clean no-ops.

**Explicit exclusions.** Photo/claim/CommercePackage backfill; retention actions on archived rows (recorded only); any legacy mutation.

**Dependencies.** I2 (ongoing path live before backfill, per owner decision 6).

**Unresolved decisions blocking I3 but not I1/I2.** Excluded-row retention semantics beyond recording (owner decision 7 explicitly defers retention behavior); malformed-row policy (skip-and-record vs halt); drift threshold / alerting channel.

---

## I4 — Approved photo sets → canonical PhotoAsset (attached assets only)

> **Status: delivered and merged.** PR #57, merge SHA `e58766e` on `gm-commerce/main`. The sections below are the authoritative spec of what shipped. Implementation facts (verified against the merged migration `20260808200000_phase_i_slice4_approved_photo_bridge.sql`, `lib/canonical/bridge/processor.ts`, and `app/photos/actions.ts`): `gmcom_mark_photo_set_approved` performs the guarded `needs_review → approved` transition and enqueues one durable `photo_sets` bridge job in a single transaction (concurrent-change guard; idempotent enqueue) — **approval and durable enqueue are atomic, and a later processing failure never reverses the legacy approval**. `gmcom_bridge_photo_set` is idempotent and concurrency-safe (deterministic lock order `photo_sets → photo_assets → photo_derivatives`; two-session race proven), creates **one canonical PhotoAsset per original legacy `photo_assets` row** (preserving `storage_ref` = original path and the new `original_content_hash`; **no files are copied or moved**), one `canonical_photo_asset_derivatives` row per legacy `photo_derivatives` row (**derivatives remain linked representations, not identities**), and one permanent `canonical_legacy_entity_bridge` row per PhotoAsset (legacy_table `'photo_assets'`, additively widened CHECK; provenance columns `legacy_approved_at`/`legacy_approved_by`). Canonical PhotoAssets are inserted with **`owner_approval_state='pending'`** and `record_purpose='operational'` — the generic `createEntity` cannot fabricate `'genuine'`, so the legacy approved photo set stays the legacy source of truth. **Replay validates the full state**: `record_purpose` drift, approval-state drift, storage/hash/linkage drift, partial mappings (never silently repaired — mismatch fails visible), full derivative state (missing/extra/duplicate derivative types), and **one shared correlation** across assets, mappings, and derivatives. The processor (`lib/canonical/bridge/processor.ts`) dispatches **fail-closed**: `products` → identity RPC, `photo_sets` → photo RPC, any other `legacy_table` is never routed and finishes `failed`/`bridge_unsupported` (unit-tested). The trigger path is `approvePhotoSet` (uses `gmcom_mark_photo_set_approved` + best-effort `processBridgeJobs`). Post-merge `main` CI for `e58766e` verified green (run `31307655608`, success; Copilot run `31307656060`).

**CI workflow-size incident and permanent correction (part of this merge).** PR #57's branch CI produced no runs because `.github/workflows/ci.yml` was 529,068 bytes — above GitHub's 512,000-byte per-workflow file limit, which rejects the workflow before any run is created (silent; the repo permissions and runner were fine). Fix: the large Phase I live-Postgres SQL heredocs and race scripts were extracted from `ci.yml` into checked-in `supabase/live/` files (`i1-identity-bridge-live.sql`, `i1-identity-bridge-race.sh`, `i2-ready-for-ai-bridge-jobs-live.sql`, `i3-identity-backfill-live.sql`, `i3-backfill-race.sh`, `i4-approved-photo-live.sql`, `i4-photo-race.sh`), invoked via short `psql -f`/`bash` steps preserving exact order, assertions, env, and failure behavior; and `supabase/ci-workflow-size.test.ts` fails any workflow YAML file at or above the project's 450 KB guard (GitHub's hard limit is 500 KB). `ci.yml` is now **414,483 bytes**; the extracted tests were run exactly as CI invokes them (I1 12 → I2 18 → I3 15 → I4 20 checks + all three race scripts) on one fresh PG15 before merge, and passed in CI on the merged `main`.

**Objective.** Make the truthfully-approved legacy event (`photo_sets.status='approved'`, the guarded terminal state that gates `publishToShopify`) produce permanent canonical PhotoAsset records with durable per-photo mappings — attached/commerce assets only, no Claims, no Evidence, no CommercePackage.

**Authoritative trigger.** `photo_sets.status` reaching `'approved'` through `approvePhotoSet` → `gmcom_mark_photo_set_approved` (atomic guarded transition + enqueue). `products.photos_confirmed_at` is a distinct, earlier intake attestation, not photo approval.

**Tables / RPCs / application paths affected.** `canonical_photo_assets` (additive `original_content_hash`); new `canonical_photo_asset_derivatives` (RLS env-scoped); `canonical_legacy_entity_bridge` (additive `legacy_table` CHECK widening for `photo_assets` + `legacy_approved_at`/`legacy_approved_by` provenance); new RPCs `gmcom_mark_photo_set_approved` and `gmcom_bridge_photo_set`; generalized `lib/canonical/bridge/processor.ts` (fail-closed dispatch); `app/photos/actions.ts` `approvePhotoSet`.

**Transaction boundaries.** Approval + enqueue in one transaction (legacy never rolls back on later canonical failure). `gmcom_bridge_photo_set` creates the full canonical PhotoAsset + derivatives + mapping set atomically per original; deterministic lock order prevents deadlocks.

**Idempotency and retry rules.** Enqueue is idempotent (one durable `photo_sets` job per `(environment, legacy_table, legacy_key)`). `gmcom_bridge_photo_set` is idempotent: replay resolves the existing mapping, validates the complete state, and returns `done`; mismatch (partial mappings, drift, divergent correlations) fails visible and is never silently repaired.

**Security / environment / record_purpose rules.** RPCs run as `service_role` only; environment-bound; `record_purpose='operational'`; no fabricated approval (PhotoAssets stay `'pending'`); RLS + grants mirror canonical conventions.

**Test and live-Postgres requirements.** Unit tests on fail-closed dispatch (products/photo_sets routed; `listing_packages`/`commerce_details`/unknown never routed → `bridge_unsupported`) and on the approval RPC; live PG15: approval → enqueue → bridge creates one PhotoAsset per original + derivatives + mapping, `owner_approval_state='pending'`, replay returns same correlation with full derivative/purpose/approval/linkage validation, mismatch paths fail visible, two-session concurrency converges (one identity set, no duplicates).

**Entry criteria.** I3 merged; owner approves I4. **Exit criteria.** Every newly approved photo set gains permanent canonical PhotoAssets with durable mappings; canonical approval stays `pending`; the bridge is idempotent, replay-safe, and fail-closed; legacy approval is never reversed by a later canonical failure.

**Explicit exclusions.** No historical photo backfill (proposed I5, not authorized); no Claims or Evidence; no CommercePackage; no Phase H queue population; no publishing changes; no later Phase I work.

**Dependencies.** I1 (canonical identity must exist for the SKU).

**I4 decisions resolved as implemented** (were the seven owner-gated inventory decisions; now resolved by the delivered, merged implementation): (1) photos are attached commerce assets, not Claims or Evidence; (2) identity unit = one canonical PhotoAsset per stable legacy **original** photo asset; (3) derivative paths/metadata are preserved as linked `canonical_photo_asset_derivatives` representation rows (additive table), not separate identities; (4) canonical PhotoAssets remain `owner_approval_state='pending'`; the legacy approved photo set remains the legacy source of truth; (5) `storage_ref` = the legacy `original_path`, with `original_content_hash` preserved (no binary copy/move); (6) durable per-photo bridge mapping via additive `legacy_table` CHECK widening to `photo_assets`, one permanent bridge row per canonical PhotoAsset, run/outcome ledger kept separate for operational history; (7) ongoing approval-event integration (`gmcom_mark_photo_set_approved`) shipped before any photo backfill, consistent with owner decision 6.

---

## Later slices — delivered and merged (historical)

### I5 — Historical approved-photo backfill (delivered and merged — PR #58, `cc19611`)

Delivered per its original objective: use the I4 bridge machinery (`gmcom_bridge_photo_set`, durable per-photo mappings, fail-closed processor) and the I3 backfill pattern (durable run + immutable per-run outcome ledger, required dry-run → single-use real-run workflow, keyset batching/resumability) to backfill photo sets that were already `approved` before I4 went live. Migration `20260809120000_phase_i_slice5_historical_approved_photo_backfill.sql`; dry-run promotion, I4 reuse, idempotency, and grants covered by live tests.

### CommercePackage creation (truthful lifecycle boundary) — delivered and merged (PR #59, `a4db76f`)

Delivered at generation finalization: a canonical CommercePackage is created when a generation finalizes. This is the slice that made real packages appear in the review-shell queue. Migration `20260809180000_phase_i_commerce_package_creation.sql`.

### CommercePackage content assembly — delivered and merged (PR #60, `38519f`)

Delivered: package content/`offers[]`/`fieldLineage[]` populated with exact-version listing content and photo references. Migration `20260809210000_phase_i_commerce_package_content_assembly.sql`. Review-shell display of the assembled content followed in PR #63 (`a0e426a`).

> **Confirmed owner requirement that constrains the output work (decisions 12–13, 2026-08-09):** the review/publishing workflow must offer **Shopify, Etsy, and a permanent master Listings Spreadsheet** as **mandatory operational destination choices** — for each approved listing, Phil or Crystal chooses the intended route (a listing is not required to be sent to all three) — and Etsy remains scheduled under **ROADMAP Milestone 4, now required for overall completion**. The Listings Spreadsheet path is append-only and idempotent, writes via a **durable export job/outbox + immutable export ledger** (never claimed atomic with the database), uses trusted-server-config credentials only, and is **not** a per-listing downloadable file (CSV download is optional backup only). Overall completion requires a **proven working path for each of the three choices**, demonstrated by completion testing that routes an **approved canonical CommercePackage through Shopify, through Etsy, and through the Listings Spreadsheet without Phil or Crystal rewriting or reformatting the approved content**. See decisions 12–13 and `COMPLETION.md`. Not designed or authorized yet.

### Claims mapping (AI-generated listing content) — delivered and merged (PR #61, `ca613e2`)

Delivered: AI-content listing claims are bridged to canonical Claims **as `under_review` only, with no fabrication** (consistent with owner decision 8 — no AI-content Claim is created as verified/genuine without a defensible SourceCategory/evidence design). Migration `20260809220000_phase_i_claims_mapping.sql`; §7 field lineage covered by live tests.

### Drift monitoring and operational reconciliation hardening — delivered and merged (PR #62, `8f60120`)

Delivered: bounded, resumable, read-only drift detection scans (legacy value hash vs bridged value) with an immutable drift ledger and **no repair** (drift is surfaced, not silently corrected). Migration `20260810000000_phase_i_drift_monitor.sql`.

### Canonical destination routing, outbox, and the three adapters — delivered and merged (PRs #65–#68)

Delivered: canonical destination routing + durable outbox (PR #65, `65be13b`), the Listings Spreadsheet adapter (PR #66, `3721fe`), the canonical Shopify draft adapter (PR #67, `19b562`), and the canonical Etsy draft adapter (PR #68, `f6e24bb`). Implemented ≠ launch-ready: see `COMPLETION.md` and `STATUS.md` for the remaining launch conditions (automatic background processing, Shopify draft-only launch verification, Etsy fail-closed configuration).

---

## Exclusions (Phase I as a whole)

No commerce mutations or publishing. No CommercePackage content fabrication. No AI-content Claims without owner decision. No retention behavior beyond recording archived rows as excluded. No cross-environment access. No changes to the working legacy pipeline's success path beyond additive bridge steps.

## Unresolved decisions by earliest blocking slice

- **Block I2 (not I1):** resolved by the delivered I2 implementation (see the I2 status note above).
- **Block I3 (not I1/I2):** resolved by the delivered I3 implementation (see the I3 status note above) — excluded-row retention, malformed-row policy, drift alerting, dry-run requirement, batch/restart behavior, terminal-job treatment, and manual owner/co-owner initiation were all decided as implemented.
- **Block I4:** **resolved by the delivered I4 implementation** (see the I4 section above). The seven inventory decisions are recorded as resolved-as-implemented: photos as attached commerce assets; one canonical PhotoAsset per legacy **original**; derivatives as linked representation rows (additive table); canonical PhotoAssets stay `owner_approval_state='pending'` with the legacy approved photo set as source of truth; `storage_ref` = legacy `original_path` + preserved `original_content_hash` (no copy/move); durable per-photo bridge mapping via additive `legacy_table` CHECK widening; ongoing approval-event integration shipped before any photo backfill.
- **I5:** **delivered and merged (PR #58, `cc19611`)** — historical approved-photo backfill using the I4 bridge machinery and the I3 backfill pattern. No longer a design recommendation only.
- **Later Phase I slices:** **all delivered and merged through PR #68** (CommercePackage creation at generation finalization, content assembly, truthful claims mapping, drift monitoring, review-shell display, atomic review decisions, destination routing/outbox, and the Listings Spreadsheet / Shopify / Etsy adapters). No further Phase I slice is proposed. Remaining launch conditions are tracked in `COMPLETION.md`; Phase 0 Slices 1–3 were completed through application PRs #69–#71 per `STATUS.md` (authoritative current status).
- **Potential future (not part of Phase I):** **image-region evidence** — an EvidenceAnchor whose `ImageRegion` locator references a canonical PhotoAsset; **SourceCategory/evidence design for AI-content Claims** (owner decision 8 — claims are bridged as `under_review` today, with no fabricated verified/genuine claims); any new SourceCategory (deliberate, never invented).
