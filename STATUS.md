# Current Project Status

_Last updated: 2026-08-13 (Phase I completed through PR #68; Phase 0 Slices 1–3 merged via PRs #69–#71; Recovery Foundation Slice 1 merged via PR #72)_

## Phase I — Legacy-to-Canonical Bridge (complete: I1–I5, CommercePackage, claims, drift, routing, and the three destination adapters all delivered and merged through PR #68)

Phase I bridges the real, working production pipeline (GMCOM-001–012) into the canonical model that Phases B–H built around. The authoritative slice plan is `phase-i-slice-plan.md` (this repo); **Phase I is now fully implemented and merged**. **The master completion map is `COMPLETION.md` (this repo)** — a plain-English answer to "what must be finished before GM Commerce is operationally complete," including the remaining launch/cutover work, the owner decisions that block it, and measurable exit criteria.

**Phase I is complete as implemented:** every slice in the plan is delivered and merged — the identity backbone (I1–I3), the approved-photo bridge (I4), the historical approved-photo backfill (I5, PR #58), canonical CommercePackage creation (PR #59), content assembly (PR #60), truthful claims mapping + field lineage (PR #61), drift monitoring/reconciliation hardening (PR #62), the review-shell display of assembled packages (PR #63), atomic canonical review decisions (PR #64), canonical destination routing/outbox (PR #65), and the three destination adapters — Listings Spreadsheet (PR #66), Shopify draft (PR #67), and Etsy draft (PR #68).

**"Implemented" is not the same as "launch-ready."** The adapters exist and are live-tested against Postgres, but the operational launch conditions are still open: automatic background processing is not yet complete (workers are manual server actions today), Shopify draft-only end-to-end launch verification has not been executed, Etsy remains fail-closed until its token store and policy source are configured and verified (its application-level pre-side-effect authorization is deferred to the Etsy activation phase), and legacy cutover has not begun. See `COMPLETION.md` and the Phase 0 section below.

`gm-commerce/main` is at `2f12de4935d9d7049ce01896f1adc4036bb577b4` (merge of PR #72 / Recovery Foundation Slice 1).

| Slice | Scope | PR | Status | Merge |
|---|---|---|---|---|
| I1 | Bridge foundation + atomic identity RPC (`canonical_legacy_entity_bridge` + `gmcom_bridge_product_identity`) | #54 | Merged | `2c26b61` |
| I2 | Ongoing `ready_for_ai` integration + durable bridge jobs | #55 | Merged | `bef5a5d` |
| I3 | Existing identity backfill (dry-run → single-use real-run, immutable outcome ledger) | #56 | Merged | `285a2a0` |
| I4 | Approved photo sets → canonical PhotoAsset | #57 | Merged | `e58766e` |
| I5 | Historical approved-photo backfill (dry-run promotion, I4 reuse, idempotency) | #58 | Merged | `cc19611` |
| CommercePackage creation | Canonical package at generation finalization | #59 | Merged | `a4db76f` |
| CommercePackage content assembly | Exact-version listing content, photo refs, lineage | #60 | Merged | `38519f` |
| Truthful claims mapping | AI-content claims + field lineage (`under_review`, no fabrication) | #61 | Merged | `ca613e2` |
| Drift monitoring | Bounded resumable scans, read-only, no repair | #62 | Merged | `8f60120` |
| Review-shell display | Assembled canonical package content in the review shell | #63 | Merged | `a0e426a` |
| Atomic canonical review decisions | Approve/reject atomicity, retry, rollback, fail-closed | #64 | Merged | `ee76dcd` |
| Canonical destination routing + outbox | Durable enqueue, lifecycle, ledger, idempotency, eligibility | #65 | Merged | `65be13b` |
| Listings Spreadsheet adapter | Durable export ledger, lease-protected claim, atomic completion | #66 | Merged | `3721fe` |
| Canonical Shopify draft adapter | Lease-protected claim, atomic marketplace success, immutable attempts | #67 | Merged | `19b562` |
| Canonical Etsy draft adapter | Create-intent-before-remote-call, `needs_confirmation` parking, single-use recreate | #68 | Merged | `f6e24bb` |

**I1–I5 and the CommercePackage/claims/drift work are all implemented and merged.** The identity backbone, approved-photo bridge, historical photo backfill, canonical CommercePackage creation at generation finalization, exact-version content assembly, truthful claims mapping (under_review only), drift monitoring (read-only, no repair), and the review-shell display of assembled package content are all delivered. The Phase H queue is now populated by **real canonical CommercePackages** created at generation finalization (PR #59), assembled with exact-version content and photo references (PR #60), and displayed in the review shell (PR #63) — the earlier "the review shell has no real CommercePackages" statement no longer holds.

The review/publishing workflow now offers the three confirmed operational destination choices (owner decisions 12–13): **Shopify** (PR #67), **Etsy** (PR #68), and the **permanent Listings Spreadsheet** (PR #66), each writing through the canonical destination outbox (PR #65) with lease-protected processing, durable/idempotent ledger behavior, and fail-closed dispatch. **Implemented does not mean launch-ready** — see the Phase 0 section and `COMPLETION.md` for the launch conditions that remain.

I4 details (delivered earlier and still accurate): an approved legacy photo set gains permanent canonical PhotoAsset records (one per original; no files copied/moved); derivatives remain linked `canonical_photo_asset_derivatives` representation rows; canonical approval stays `pending` (the legacy approved photo set remains the source of truth); durable per-photo mappings with provenance; replay validates full state; the processor dispatches fail-closed.

I2 delivered: every real `ready_for_ai` transition atomically enqueues a durable identity-bridge job; request-triggered processing + a bounded manual drain (owner/co-owner authorized via `lib/auth`), leases, retries, dead-lettering, correlation continuity, immutable attempt history.

I3 delivered: existing eligible legacy products can be backfilled into canonical identity through a required dry-run → single-use real-run workflow (DB-enforced single-use), explicit four-status eligibility, durable run + immutable per-run outcome ledger, keyset batching + resumability, run integrity/state machine. **Backfill remains identity-only** (I3); I5 added the historical approved-photo backfill using the I4 machinery and the I3 pattern.

> Post-merge CI for the completed Phase I and Phase 0 Slice 1 (merged `main` at `c6cf6c8`, run `31565668557`): **success, 24/24 jobs**. The `schema-drift-deferred` job completed successfully with its documented deferred-drift behavior (its two steps skip when `SUPABASE_DB_URL` is unset and the migration ledger is not bootstrapped); those steps are skipped by design, not "passed."

> Post-merge CI for Phase 0 Slice 2 (merged `main` at `9cc5f6a`, run `31605446557`): **success, 24/24 jobs** (0 failed, 0 cancelled, 0 skipped).

> Post-merge CI for Phase 0 Slice 3 (merged `main` at `851be71`, run `31615866708`): **success, 24/24 jobs** (0 failed, 0 cancelled, 0 skipped).

> Post-merge CI for Recovery Foundation Slice 1 (merged `main` at `2f12de4`, run `31662919875`): **success, 24/24 jobs** (0 failed, 0 cancelled, 0 skipped).

## Recovery Foundation Slice 1 — retry limits and queue health (PR #72, merge `2f12de4`, migration `20260818000000_recovery_foundation_slice1.sql`)

**Recovery Foundation Slice 1 is complete.** It is a **Recovery Foundation** slice, **not a Phase 0 slice** (Phase 0 ended with Slice 3). It is the database and observability foundation required before any automatic processing; **no scheduler, continuously running worker, notifications, or AI recovery are implemented here.**

- **Database-enforced maximum attempts:** `attempt_count` on a destination request counts processing **leases issued**. Five processing attempts are allowed; the **fifth attempt may run and succeed**. A request at the attempt limit is terminally failed with `max_attempts_exceeded` at the database claim boundary (`gmcom_claim_destination_requests`) and **a sixth lease is denied**.
- **Terminalization precedence:** `package_superseded` → `snapshot_mismatch` → `max_attempts_exceeded`. Currentness and snapshot revalidation run first, so their truthful error codes are never relabeled.
- **Retry/backoff:** default backoff is **60, 300, 900, and 3600 seconds, capped at 3600**; an explicit provider **`Retry-After` takes precedence** over the default backoff. `rate_limited` (retryable) and `max_attempts_exceeded` (terminal, written only by the claim boundary) are approved error codes.
- **`worker_runs`** provides the worker-run lifecycle audit ledger (begin/finalize; a finalized run cannot be rewritten; service-role only; environment-isolated).
- **`gmcom_destination_queue_health`** provides read-only, environment-scoped queue-health visibility.
- **Manual processing controls remain the operational fallback**; they use the same claim RPC and cannot bypass the database maximum.
- **Scheduled Worker Slice 2 has not begun.** The intended worker host is **Phil's UPS-backed Linux computer**, using a **dedicated systemd service/timer isolated from the self-hosted GitHub Actions runner**. **Vercel is not the selected host.** GitHub Actions / the self-hosted runner remain CI-only, not production workers.
- **Etsy remains implemented but inactive and fail-closed**; this slice does not activate it.
- **Phase 2 batch processing remains a mandatory Phil design/approval checkpoint.**

**Verification:** 2239 application tests passed locally; migrations from empty and upgrade passed; consolidated schema passed; full CI-order live sequence passed on one shared fresh PG15 database; recovery live/race/upgrade passed; Phase 0 Slice 2/Slice 3 regression races passed; PR CI passed; post-merge CI run `31662919875` passed 24/24 jobs.

## Phase 0 — Launch hardening (Slices 1–3 delivered and merged via PRs #69–#71; Phase 0 complete)

**Phase 0 Slice 1 — Environment and legacy-access hardening (PR #69, merge `c6cf6c8`, migration `20260812120000_phase0_slice1_environment_and_legacy_access_hardening.sql`):** removed every hardcoded/defaulted `'production'` from the legacy→canonical bridge and the legacy-writing RPCs; every such RPC now takes a **required** environment argument and sets the `app.gmcom_caller_environment` GUC transaction-locally, and the dual-write triggers fail closed when no environment is set (`gmcom_set_legacy_bridge_environment` / `gmcom_require_active_environment`). The unused 8-arg `gmcom_apply_legacy_correction` overload was dropped. The legacy tables (`products`, `listing_packages`, `listing_package_versions`, `photo_*`, `shopify_publications`, `commerce_details`) gained RLS with `anon`/`authenticated` revoked and explicit `service_role` grants. TS wrappers thread `resolveTrustedEnvironment()`. **Does not retire legacy functionality** and does not touch current-version logic, destination dedup, automation, batch, or photo architecture.

**Phase 0 Slice 2 — Current-version invariant and regeneration safety (PR #70, merge `9cc5f6a`, migration `20260816000000_phase0_slice2_current_version_safety.sql`):** one authoritative current canonical CommercePackage per environment and SKU, with deterministic highest-version convergence. Materialization is serialized per (environment, SKU) by locking the parent `canonical_skus` row (lock order `canonical_skus → canonical_commerce_packages → canonical_destination_requests`), and the partial unique index `canonical_commerce_packages_one_current_per_sku_env` is retained as defense in depth. Higher versions atomically supersede older current packages; lower versions remain historical/`superseded`; equal-version bridge replay is idempotent; the previous package stays authoritative until its replacement successfully materializes. Stale approval, enqueue, claim, retry, and success paths fail closed; nonterminal requests for superseded packages become terminal `failed`/`package_superseded`; historical packages, decisions, requests, and audit events are preserved. Shopify and Listings Spreadsheet now reauthorize immediately before external writes; the unavoidable database-to-external-API micro-window remains documented. Environment isolation is preserved.

**Verification:** 2204 application tests passed locally; migration from empty and upgrade passed; consolidated schema passed; materialization race verification passed; PR CI passed; post-merge CI run `31605446557` passed 24/24 jobs.

**Etsy deferral (Phase 0 Slice 2):** Etsy is implemented but **not** launch-active; it remains fail-closed. Shopify launches first. Phase 0 Slice 2's generic database protections cover Etsy requests at the database boundary. Etsy application-level pre-side-effect dispatch authorization is **intentionally deferred** to the Etsy activation phase — the missing protection is not optional. Resume it only when Phil explicitly begins and authorizes Etsy activation. The authoritative technical checklist is `gm-commerce/docs/canonical-etsy-draft.md` §10 (authorization immediately before `createDraftListing`, `updateListingInventory`, and every `uploadListingImage` inside the image loop, plus credentials/token store, shop identity, taxonomy, shipping/profile, inventory/variation, image, and listing-policy prerequisites, draft-only end-to-end verification, partial-external-side-effect recovery verification, and Phil's explicit activation authorization).

**Phase 0 Slice 3 — Destination-request deduplication (PR #71, merge `851be71`, migration `20260817000000_phase0_slice3_destination_dedup.sql`):** at most one ACTIVE operational destination request per delivery intent `(environment, commerce_package_id, destination, intended_external_destination, custom_destination_label)`, enforced by the partial unique index `canonical_destination_requests_one_active_per_delivery_intent`. Sequential and concurrent duplicate submissions — including different correlation IDs — converge to the existing request; active requests take precedence over terminal history; terminal-only history follows the established no-redelivery rule; exact-correlation replay remains idempotent and mismatched correlation reuse fails closed. The upgrade reconciliation retains the oldest ACTIVE request and records discarded duplicates as terminal `failed`/`duplicate_intent` (distinct from `package_superseded`, which remains reserved for actual package supersession), with the surviving request named in audit metadata. Existing same-row retry and Etsy recreate behavior are preserved; historical requests and events are preserved; superseded packages remain ineligible via Slice 2; deliberate redelivery is not implemented and requires a future explicit owner-authorized mechanism. Lock order unchanged: `canonical_skus → canonical_commerce_packages → canonical_destination_requests`. Etsy remains inactive and fail-closed. **Phase 0 Slices 1–3 are complete.**

**Verification:** 2211 application tests passed locally; migration from empty and upgrade passed; consolidated schema passed; dedup race (concurrent different-correlation and terminal-history scenarios) passed; dedup upgrade reconciliation passed; PR CI passed; post-merge CI run `31615866708` passed 24/24 jobs.

**Approved next sequence (owner-confirmed):**

- **Legacy cutover** comes later, after the required capabilities and dependencies are safely moved (see the safe cutover sequence in `DECISIONS.md` and `COMPLETION.md`).
- **Automatic background processing** (review finding C3/High — no automatic consumer; processors are manual server actions) is a launch requirement per owner decision 3, and is not yet implemented.
- **Batch processing (Phase 2) is REQUIRED before launch but is a mandatory owner-design checkpoint:** it must not be designed or implemented until Phil approves how it should work.

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

> Historical note: the earlier "Phase H remains in progress / next proposed work: H5" guidance is superseded. Phase H is fully complete (H1–H8 + the Phase E/F/G prerequisite corrections). The subsequent phase was Phase I (now also complete through PR #68).

**Open issue:** [#46 — Reconcile Phase E compliance gate functions and approval trigger into schema.sql](https://github.com/HydraCoreSystems/gm-commerce/issues/46) (open). Ordered migrations remain the deployment source of truth; this does not block Phase I or Phase 0.

## Phase G — Owner-Editable Policies and Learned-Rule Activation (complete; superseded "next task" guidance is historical)

Phase G enabled editing policies and confirming Phase F recommendations into standing learned rules. The authoritative slice plan is `phase-g-slice-plan.md` (this repo).

| Slice | Scope | PR | Status |
|---|---|---|---|
| G1 | Policy management — create, query, evaluate policies | #30 | Merged |
| G2 | Owner confirmation flow — recommendation → owner decision | #31 | Merged |
| G3 | Correction capture and scope inference | #32 | Merged |
| G4 | Rule activation engine — learned rules table, activate/revoke/query | #33 | Merged |
| G5 | RBAC enforcement — §15 role matrix at Repository command layer | Not started |

> Historical note: the earlier "Next Phase: Phase G" guidance is superseded. Phase G is complete through G4; G5 (RBAC enforcement) remains a later-phase item. The current next phase is Phase 0 launch hardening (above).

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

- **52 ordered migrations** now apply cleanly from empty (CI `schema-from-empty` passes), and the consolidated `schema.sql` mirror builds an equivalent fresh database (verified on merged `main` at `9cc5f6a`).
- I1 (PR #54, `2c26b61`): `canonical_legacy_entity_bridge` + `gmcom_bridge_product_identity`.
- I2 (PR #55, `bef5a5d`): `canonical_legacy_bridge_jobs` + immutable `canonical_legacy_bridge_job_attempts` + `gmcom_mark_product_ready_for_ai`/`gmcom_enqueue_legacy_bridge_job`/`gmcom_claim_legacy_bridge_job`/`gmcom_finish_legacy_bridge_job`.
- I3 (PR #56, `285a2a0`): redefined `gmcom_bridge_product_identity` (four-status allowlist) + `canonical_legacy_backfill_runs` + immutable `canonical_legacy_backfill_row_outcomes` + dry-run→real-run single-use linkage + backfill RPCs.
- I4 (PR #57, `e58766e`): `canonical_legacy_entity_bridge.legacy_table` widened to `photo_assets` + provenance columns; `canonical_photo_asset_derivatives`; `gmcom_mark_photo_set_approved` + `gmcom_bridge_photo_set`; fail-closed processor; CI workflow-size correction (`ci.yml` 529,068 → 414,483 bytes; `supabase/ci-workflow-size.test.ts` 450 KB guard).
- I5 (PR #58, `cc19611`): migration `20260809120000_phase_i_slice5_historical_approved_photo_backfill.sql` — historical approved-photo backfill (dry-run promotion, I4 reuse, idempotency, grants).
- CommercePackage creation (PR #59, `a4db76f`): `20260809180000_phase_i_commerce_package_creation.sql`.
- Content assembly (PR #60, `38519f`): `20260809210000_phase_i_commerce_package_content_assembly.sql`.
- Claims mapping (PR #61, `ca613e2`): `20260809220000_phase_i_claims_mapping.sql`.
- Drift monitor (PR #62, `8f60120`): `20260810000000_phase_i_drift_monitor.sql`.
- Atomic canonical review decisions (PR #64, `ee76dcd`): `20260811000000_phase_i_atomic_canonical_review_decisions.sql`.
- Destination routing outbox (PR #65, `65be13b`): `20260812000000_phase_i_canonical_destination_routing_outbox.sql`.
- Listings Spreadsheet adapter (PR #66, `3721fe`): `20260813000000_phase_i_listings_spreadsheet_adapter.sql`.
- Shopify draft adapter (PR #67, `19b562`): `20260814000000_phase_i_canonical_shopify_draft_adapter.sql`.
- Etsy draft adapter (PR #68, `f6e24bb`): `20260815000000_phase_i_canonical_etsy_draft_adapter.sql`.
- Phase 0 Slice 1 (PR #69, `c6cf6c8`): `20260812120000_phase0_slice1_environment_and_legacy_access_hardening.sql` — environment helpers, hardened dual-write triggers and legacy RPCs, legacy-table RLS/grants.
- Phase 0 Slice 2 (PR #70, `9cc5f6a`): `20260816000000_phase0_slice2_current_version_safety.sql` — current-version authority (parent `canonical_skus` serialization, partial unique index as defense in depth), supersede/reconcile on new current versions, terminal `failed`/`package_superseded` reconciliation, dispatch authorization (`gmcom_authorize_destination_dispatch`) for Shopify and Listings Spreadsheet, and regeneration authority-transfer safety.
- Phase 0 Slice 3 (PR #71, `851be71`): `20260817000000_phase0_slice3_destination_dedup.sql` — delivery-intent-based destination-request deduplication (partial unique index on `(environment, commerce_package_id, destination, intended_external_destination, custom_destination_label)` for ACTIVE operational requests), sequential/concurrent convergence, active-first selection, `failed`/`duplicate_intent` upgrade reconciliation naming the surviving request, and enqueue convergence preserving exact-correlation replay.
- Recovery Foundation Slice 1 (PR #72, `2f12de4`): `20260818000000_recovery_foundation_slice1.sql` — default backoff schedule (`gmcom_recovery_default_backoff_seconds`, 60/300/900/3600s capped), `rate_limited`/`max_attempts_exceeded` failure codes, claim-boundary maximum-attempt enforcement (7-arg `gmcom_claim_destination_requests`, `p_max_attempts` default 5; a sixth lease is denied), `worker_runs` lifecycle ledger with table-level guard trigger, and read-only environment-scoped `gmcom_destination_queue_health`.
- Migration gap resolved: `commerce_details` table and `listing_packages.seo_title`/`seo_description` columns have committed DDL at `20260802040000_commerce_readiness.sql`.
- `20260803000000_commerce_field_ownership.sql` (price/`content_provenance`) was deliberately retired as never-applied.
- `HY-LOB01-C04` test data verified absent from the live database (2026-08-05).
- **Issue #46 open:** consolidated `schema.sql` omits the Phase E compliance gate functions/trigger. Ordered migrations are the deployment source of truth; reconciliation is tracked, not blocking.

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

- **Phase I is complete through PR #68** (I1–I5, CommercePackage creation/assembly/claims, drift, review decisions, routing outbox, and the Shopify/Etsy/Listings-Spreadsheet adapters). The review shell is populated by real canonical CommercePackages; the three destination choices are implemented. **Implemented ≠ launch-ready**: see the Phase 0 section for the launch conditions still open (auto-processing, Shopify launch verification, Etsy fail-closed configuration + deferred pre-side-effect authorization, legacy cutover).
- **Phase 0 Slice 1 merged (PR #69, `c6cf6c8`)** — environment and legacy-access hardening delivered and post-merge CI green (run `31565668557`, 24/24 jobs, with the documented deferred-drift skip for `schema-drift-deferred`).
- **Phase 0 Slice 2 merged (PR #70, `9cc5f6a`)** — current-version invariant and regeneration safety delivered and post-merge CI green (run `31605446557`, 24/24 jobs). Etsy application-level pre-side-effect authorization is intentionally deferred to the Etsy activation phase.
- **Phase 0 Slice 3 merged (PR #71, `851be71`)** — destination-request deduplication delivered and post-merge CI green (run `31615866708`, 24/24 jobs).
- **Recovery Foundation Slice 1 merged (PR #72, `2f12de4`)** — retry limits and queue health delivered and post-merge CI green (run `31662919875`, 24/24 jobs). Recovery Foundation Slice 1 is complete; it is not a Phase 0 slice. Scheduled Worker Slice 2 has not begun.
- **Phase H is COMPLETE** (H1–H8 + Phase E/F/G prerequisite corrections, merged at `c65b023` per PR #53). No further Phase H slices are proposed.
- **Owner-confirmed output requirement (2026-08-09):** the review/publishing workflow must offer **Shopify, Etsy, and a permanent master Listings Spreadsheet** as **mandatory operational destination choices**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** Etsy is required for overall GM Commerce completion (ROADMAP Milestone 4, now classified as required). The Listings Spreadsheet is append-only and idempotent, written through a **durable export job/outbox with an immutable export ledger** (never claimed atomic with the database), configured from trusted server credentials only, and is **not** a per-listing downloadable file (CSV download is optional backup only). All three adapters are now **implemented** (PRs #66–#68); launch readiness is still pending (Etsy fail-closed until its token store and policy source are configured and verified; Shopify draft-only end-to-end launch verification not yet executed; automatic background processing not yet complete).
- **GitHub Issues**: Several tasks lack formal Issues. GitHub API access has been intermittent. Issue #46 remains open (see Schema State).
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Phase

**Phase 0 — Launch hardening.** Slices 1–3 (PRs #69–#71, `c6cf6c8`, `9cc5f6a`, `851be71`) are merged; **Phase 0 Slices 1–3 are complete.** **Recovery Foundation Slice 1** (PR #72, `2f12de4`) is also merged and complete; it is **not** a Phase 0 slice. The next implementation work is **Scheduled Worker Slice 2** — a production scheduler invoking the existing processors on a schedule — which **has not begun**. The intended worker host is **Phil's UPS-backed Linux computer** with a dedicated systemd service/timer isolated from the self-hosted GitHub Actions runner (Vercel is not the selected host). The remaining launch conditions in `COMPLETION.md` (automatic background processing, Shopify launch verification, Etsy configuration/authorization, photo architecture, batch processing, legacy cutover, and the recovery ladder) remain; **Phase 2 batch behavior is a mandatory owner-design checkpoint and must not be designed or implemented until Phil approves how it should work.**

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | HQ status/plan documentation |
| Claude | Primary coordination + implementation + review | Available | Phase I complete (PRs #54–#68); Phase 0 Slices 1–3 (PRs #69–#71); Recovery Foundation Slice 1 (PR #72) delivered; next implementation slice (Scheduled Worker Slice 2) pending authorization |
| GitHub Copilot | Implementation contributor | Quota-limited | Phase I bridge foundation (I1) |
| Phil | Product owner | Available as schedule permits | Next implementation slice authorization; Phase 2 batch design decision |

Capacity status should be updated whenever a provider limit is reached or resets.
