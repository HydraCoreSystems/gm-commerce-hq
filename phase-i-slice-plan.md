# Phase I Slice Plan — Legacy-to-Canonical Bridge

_Authoritative plan for the legacy-to-canonical bridge. Based on the read-only design inventory (see `handoffs/2026-08-08-phase-h-complete-and-legacy-canonical-bridge-handoff.md` and its superseding addendum). Last updated: 2026-08-08._

> **Slice status:** **I1 is delivered and merged** — PR #54, merge SHA `2c26b61` on `gm-commerce/main`. **I2 has not begun** and remains pending design and owner authorization; its design section below is the current authoritative spec. I3 and later slices are not begun.

## Purpose

Bridge the real, working production pipeline (GMCOM-001–012: Skrybix / Product-SKU-Generator selection → SKU-named photo folder → human photo confirmation → AI listing generation → Shopify publish), which today writes **only legacy tables** (`products`, `listing_packages`, `photo_sets`, `commerce_details`), into the canonical model (`canonical_product_concepts`, `canonical_skus`, `canonical_source_records`, `canonical_commerce_packages`, `canonical_claims`, …) that Phases B–H were built around.

The **architecture itself is approved by the owner as the next phase** (owner decision 11). Slices are independently testable and mergeable. **I1 is delivered and merged (PR #54, `2c26b61`); no later Phase I slice has been implemented.**

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
| I2 | Ongoing `ready_for_ai` integration + durable bridge job | Planned — design below; not begun | After I1 (`2c26b61`) |
| I3 | Existing identity backfill (re-runnable, no duplicates, excluded-row recording) | Planned — design below | After I2 |
| Later | PhotoAsset/evidence after approved photo sets | Design required before implementation | After I3 |
| Later | CommercePackage creation at a truthful lifecycle boundary | Design required before implementation | After I3 |
| Later | CommercePackage content assembly | Design required before implementation | TBD |
| Later | Claims mapping (only after SourceCategory/evidence authorization) | Blocked on owner decision; design required | TBD |
| Later | Drift monitoring + operational reconciliation hardening | Design required before implementation | After I3 |

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

> **Status: not begun.** Pending design and owner authorization. The design sections below are the current authoritative spec. A read-only I2 design inventory (file-level evidence, architecture options, transaction boundaries, idempotency/retry/concurrency behavior, proposed files and tests, and owner decisions) is in progress and will be available for owner review; it has not been adopted into this plan yet.

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

## Later slices — design required before implementation

### PhotoAsset / evidence bridge (after approved photo sets)
Objective: create `canonical_photo_assets` (and approval evidence) once a `photo_sets` row is `status='approved'`. Blocked considerations: `canonical_photo_assets` has **no `status`/`approved_at` columns** (verified); approval must be represented via a claim or RecordContext — design decision. Requires I1 (SKU must exist) and a photo-approval evidence design. Not authorized yet.

### CommercePackage creation (truthful lifecycle boundary)
Objective: create a `draft` `canonical_commerce_packages` row at a truthful boundary. Owner decision 10: timing is a later design decision (likely generation finalization / entry into review) and is **not** an I1 assumption. This is the slice that makes a package appear in the Phase H queue (see the correction above). Requires I1.

### CommercePackage content assembly
Objective: populate package content/`offers[]`/`fieldLineage[]` (deferred in the contract — `getCompletedPackage` is `deferred`). Requires CommercePackage creation + a content-availability decision.

### Claims mapping (AI-generated listing content)
Objective: map `listing_packages` AI content to canonical Claims. **Blocked on an explicit owner decision** on a defensible SourceCategory/evidence design (owner decision 8). No slice may create AI-content Claims before that. The closed 6-category taxonomy and the evidence-anchor requirements mean this cannot be bridged "for free."

### Drift monitoring and operational reconciliation hardening
Objective: scheduled drift detection (legacy value hash vs bridged value), alerting, reconciliation. Requires I3's drift reporting as a base.

---

## Exclusions (Phase I as a whole)

No commerce mutations or publishing. No CommercePackage content fabrication. No AI-content Claims without owner decision. No retention behavior beyond recording archived rows as excluded. No cross-environment access. No changes to the working legacy pipeline's success path beyond additive bridge steps.

## Unresolved decisions by earliest blocking slice

- **Block I2 (not I1):** exact wiring of `markReadyForAI` (modify vs wrap RPC); outbox consumption trigger and retry schedule; enqueue mechanism.
- **Block I3 (not I1/I2):** excluded-row retention semantics; malformed-row policy; drift threshold/alerting.
- **Block later slices (not I1/I2/I3):** photo-approval evidence representation (claim vs RecordContext); CommercePackage creation timing (owner decision 10) and content-availability; **SourceCategory/evidence design for AI-content Claims (owner decision 8 — the only hard owner-gated blocker)**; any new SourceCategory (deliberate, never invented).
