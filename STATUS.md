# Current Project Status

_Last updated: 2026-08-12 (Phase I completed through PR #68; Phase 0 Slice 1 merged via PR #69)_

## Phase I — Legacy-to-Canonical Bridge (complete: I1–I5, CommercePackage, claims, drift, routing, and the three destination adapters all delivered and merged through PR #68)

Phase I bridges the real, working production pipeline (GMCOM-001–012) into the canonical model that Phases B–H built around. The authoritative slice plan is `phase-i-slice-plan.md` (this repo); **Phase I is now fully implemented and merged**. **The master completion map is `COMPLETION.md` (this repo)** — a plain-English answer to "what must be finished before GM Commerce is operationally complete," including the remaining launch/cutover work, the owner decisions that block it, and measurable exit criteria.

**Phase I is complete as implemented:** every slice in the plan is delivered and merged — the identity backbone (I1–I3), the approved-photo bridge (I4), the historical approved-photo backfill (I5, PR #58), canonical CommercePackage creation (PR #59), content assembly (PR #60), truthful claims mapping + field lineage (PR #61), drift monitoring/reconciliation hardening (PR #62), the review-shell display of assembled packages (PR #63), atomic canonical review decisions (PR #64), canonical destination routing/outbox (PR #65), and the three destination adapters — Listings Spreadsheet (PR #66), Shopify draft (PR #67), and Etsy draft (PR #68).

**"Implemented" is not the same as "launch-ready."** The adapters exist and are live-tested against Postgres, but the operational launch conditions are still open: automatic background processing is not yet complete (workers are manual server actions today), the current-version invariant/regeneration safety and destination-request deduplication findings from Phase 0 review are not yet fixed, Shopify draft-only end-to-end launch verification has not been executed, Etsy remains fail-closed until its token store and policy source are configured and verified, and legacy cutover has not begun. See `COMPLETION.md` and the Phase 0 section below.

`gm-commerce/main` is at `c6cf6c89c7a233a5c026c55e1e4a3fb89de5edfe` (merge of PR #69 / Phase 0 Slice 1).

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

## Phase 0 — Launch hardening (Slice 1 delivered and merged via PR #69; Slice 2 and 3 planned, not begun)

**Phase 0 Slice 1 — Environment and legacy-access hardening (PR #69, merge `c6cf6c8`, migration `20260812120000_phase0_slice1_environment_and_legacy_access_hardening.sql`):** removed every hardcoded/defaulted `'production'` from the legacy→canonical bridge and the legacy-writing RPCs; every such RPC now takes a **required** environment argument and sets the `app.gmcom_caller_environment` GUC transaction-locally, and the dual-write triggers fail closed when no environment is set (`gmcom_set_legacy_bridge_environment` / `gmcom_require_active_environment`). The unused 8-arg `gmcom_apply_legacy_correction` overload was dropped. The legacy tables (`products`, `listing_packages`, `listing_package_versions`, `photo_*`, `shopify_publications`, `commerce_details`) gained RLS with `anon`/`authenticated` revoked and explicit `service_role` grants. TS wrappers thread `resolveTrustedEnvironment()`. **Does not retire legacy functionality** and does not touch current-version logic, destination dedup, automation, batch, or photo architecture.

**Approved next sequence (owner-confirmed):**

- **Phase 0 Slice 2 — current-version invariant and regeneration safety** (review finding D1/High): an older APPROVED canonical package stays approved and routable after regeneration; the bridge supersedes only `status='draft'` and there is no "newer version exists" guard in the review-decision/enqueue RPCs. Verify and fix in a later Phase 0 slice.
- **Phase 0 Slice 3 — destination-request deduplication** (review finding D2/High): duplicate destination requests on double submit (no unique constraint on `(environment, package, destination)`, fresh correlation per submit, no UI pending guard). Fix before auto-processing.
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

- **50 ordered migrations** now apply cleanly from empty (CI `schema-from-empty` passes), and the consolidated `schema.sql` mirror builds an equivalent fresh database (verified on merged `main` at `c6cf6c8`).
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

- **Phase I is complete through PR #68** (I1–I5, CommercePackage creation/assembly/claims, drift, review decisions, routing outbox, and the Shopify/Etsy/Listings-Spreadsheet adapters). The review shell is populated by real canonical CommercePackages; the three destination choices are implemented. **Implemented ≠ launch-ready**: see the Phase 0 section for the launch conditions still open (auto-processing, D1/D2 fixes, Shopify launch verification, Etsy fail-closed configuration, legacy cutover).
- **Phase 0 Slice 1 merged (PR #69, `c6cf6c8`)** — environment and legacy-access hardening delivered and post-merge CI green (run `31565668557`, 24/24 jobs, with the documented deferred-drift skip for `schema-drift-deferred`).
- **Phase H is COMPLETE** (H1–H8 + Phase E/F/G prerequisite corrections, merged at `c65b023` per PR #53). No further Phase H slices are proposed.
- **Owner-confirmed output requirement (2026-08-09):** the review/publishing workflow must offer **Shopify, Etsy, and a permanent master Listings Spreadsheet** as **mandatory operational destination choices**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** Etsy is required for overall GM Commerce completion (ROADMAP Milestone 4, now classified as required). The Listings Spreadsheet is append-only and idempotent, written through a **durable export job/outbox with an immutable export ledger** (never claimed atomic with the database), configured from trusted server credentials only, and is **not** a per-listing downloadable file (CSV download is optional backup only). All three adapters are now **implemented** (PRs #66–#68); launch readiness is still pending (Etsy fail-closed until its token store and policy source are configured and verified; Shopify draft-only end-to-end launch verification not yet executed; automatic background processing not yet complete).
- **GitHub Issues**: Several tasks lack formal Issues. GitHub API access has been intermittent. Issue #46 remains open (see Schema State).
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Phase

**Phase 0 — Launch hardening.** Slice 1 (PR #69, `c6cf6c8`) is merged. **Phase 0 Slice 2 (current-version invariant and regeneration safety) and Phase 0 Slice 3 (destination-request deduplication) are approved to plan and implement next, in that order; Phase 0 Slice 2 has not begun.** Legacy cutover comes later, after required capabilities and dependencies are safely moved. **Phase 2 batch behavior is a mandatory owner-design checkpoint and must not be designed or implemented until Phil approves how it should work.** See `COMPLETION.md` for the finish-line map and the remaining launch conditions.

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | HQ status/plan documentation |
| Claude | Primary coordination + implementation + review | Available | Phase I complete (PRs #54–#68); Phase 0 Slice 1 (PR #69) delivered; Phase 0 Slice 2 pending authorization |
| GitHub Copilot | Implementation contributor | Quota-limited | Phase I bridge foundation (I1) |
| Phil | Product owner | Available as schedule permits | Phase 0 Slice 2 authorization; Phase 2 batch design decision |

Capacity status should be updated whenever a provider limit is reached or resets.
