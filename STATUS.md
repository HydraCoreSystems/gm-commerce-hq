# Current Project Status

_Last updated: 2026-08-08_

## Phase I — Legacy-to-Canonical Bridge (I1 delivered and merged; I2 pending design/owner authorization)

Phase I bridges the real, working production pipeline (GMCOM-001–012) into the canonical model that Phases B–H built around. The authoritative slice plan is `phase-i-slice-plan.md` (this repo).

**Status: I1 delivered and merged via PR #54 (`gm-commerce/main` at `2c26b61`).** I2 (ongoing `ready_for_ai` integration + durable bridge job) has **not begun** and remains pending design and owner authorization. No later slice (I3 backfill, photos, CommercePackage, claims, drift hardening) has begun.

`gm-commerce/main` is at `2c26b61dafae3ac79cdb54dced7160adab06a7fd` (merge of PR #54 / I1).

| Slice | Scope | PR | Status | Merge |
|---|---|---|---|---|
| I1 | Bridge foundation + atomic identity RPC (`canonical_legacy_entity_bridge` + `gmcom_bridge_product_identity`) | #54 | Merged | `2c26b61` |
| I2 | Ongoing `ready_for_ai` integration + durable bridge job | — | Planned — not begun | — |
| I3 | Existing identity backfill | — | Planned — after I2 | — |

I1 delivered the durable, environment-scoped `canonical_legacy_entity_bridge` substrate (`pending`/`processing`/`done`/`failed`/`mismatch`/`retry`/`excluded` statuses, `correlation_id`, `retry_count`, `error_context`) and the atomic `gmcom_bridge_product_identity` RPC that creates ProductConcept → SKU → SourceRecord → mapping in one transaction: operational-only, `owner_approval_state='pending'`, idempotent replay, drift/mismatch fail-visible, concurrent-winner convergence, atomic rollback, service_role-only execute, RLS fail-closed and env-scoped. Genuine two-session concurrency was tested and passed (both simultaneous calls return the same identity set; exactly one identity set and three bridge rows remain). The consolidated `schema.sql` mirror was updated in the same merge.

**I1 does not create CommercePackages and does not populate the Phase H queue.** I2 has not begun.

> Post-merge CI note: at the time of this update the `main` CI workflow for `2c26b61` (run `31258503603`) had not completed (queued); the Copilot workflow (`31258505200`) completed successfully. Verify the CI workflow independently before treating `main` as green.

Owner-approved decisions (2026-08-08, full list in `phase-i-slice-plan.md`): identity creation triggered at `products.status='ready_for_ai'`; a new `canonical_legacy_entity_bridge` table (not a widening of `canonical_legacy_field_bridge`); ProductConcept + SKU + SourceRecord + mapping created atomically by one RPC; legacy never rolls back on bridge failure; a durable bridge job/outbox (`pending/processing/done/failed/mismatch/retry` + `correlation_id` + error context); ongoing production bridging activated before backfill; archived rows initially skipped and recorded as excluded (no invented retention); no AI-content Claims until a defensible SourceCategory/evidence design is approved; **I1 does not populate the Phase H queue** (only a canonical CommercePackage does); CommercePackage timing is a later decision; the bridge architecture is approved as the next phase.

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

- **Phase I: I1 delivered and merged (PR #54, `gm-commerce/main` at `2c26b61`).** The authoritative multi-slice plan is `phase-i-slice-plan.md`. I1 established `canonical_legacy_entity_bridge` + the atomic `gmcom_bridge_product_identity` RPC; genuine two-session concurrency was tested and passed. **I2 (ongoing `ready_for_ai` integration + durable bridge job) has not begun** and remains pending design and owner authorization; a read-only I2 design inventory is under way/available for review. I1 creates no CommercePackages and populates nothing in the Phase H queue.
- **Phase H is COMPLETE** (H1–H8 + Phase E/F/G prerequisite corrections, merged at `c65b023` per PR #53). No further Phase H slices are proposed.
- **GitHub Issues**: Several tasks lack formal Issues. GitHub API access has been intermittent. Issue #46 remains open (see Schema State).
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Phase

**Phase I — Legacy-to-Canonical Bridge.** I1 is delivered and merged (PR #54, `gm-commerce/main` at `2c26b61`). **I2 (ongoing `ready_for_ai` integration + durable bridge job, deployed before backfill) is next and has not begun; it requires design and owner authorization.** Then I3 (existing identity backfill); then later slices (photos, CommercePackage, content assembly, claims mapping, drift hardening) that each require their own design and — for claims — an explicit SourceCategory/evidence decision.

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | Phase I plan/status documentation |
| Claude | Primary coordination + implementation + review | Available | Phase I: I1 delivered (PR #54, `main` `2c26b61`); I2 design inventory + status documentation |
| GitHub Copilot | Implementation contributor | Quota-limited | Phase I bridge foundation (I1) |
| Phil | Product owner | Available as schedule permits | I2 slice authorization |

Capacity status should be updated whenever a provider limit is reached or resets.
