# Current Project Status

_Last updated: 2026-08-07_

## Phase H — Read-Only Completed-Review Refinement (current)

Phase H surfaces what Phases B–G actually produced as read-only context in the review shell. The authoritative slice plan is `phase-h-slice-plan.md` (this repo).

`gm-commerce/main` is at `606c5a5301605e05ee71e470cabaf129ec28e589`.

| Slice | Scope | PR | Status | Merge |
|---|---|---|---|---|
| H1 | Canonical commerce → read-only `ReviewPackage` adapter | #42 | Merged | `f44ceb2` |
| H2 | Canonical loader + precedence/contradiction preservation + detail routing | #43 | Merged | `9540a1a` |
| H3 | Operational-only commerce queue discovery | #44 | Merged | `d3cba35` |
| Phase E prerequisite correction | Operational-only compliance current-state/gate derivation | #45 | Merged | `1472660` |
| H4 | Read-only Phase E compliance context on commerce detail | #47 | Merged | `606c5a5` |

All commerce capabilities remain `false`; Phase H is strictly read-only.

**Phase H remains in progress** because the C/D/F/G read-only context is not yet surfaced: Phase C evidence-library context, Phase D vision-analysis context, Phase F recommendation + `SelectionTrace` context, and Phase G policy/learned-rule context are all genuine remaining gaps (not yet designed — see `phase-h-slice-plan.md`).

**Next proposed work: H5 design inventory for Phase C evidence-library context** — design only, not implementation.

**Open issue:** [#46 — Reconcile Phase E compliance gate functions and approval trigger into schema.sql](https://github.com/HydraCoreSystems/gm-commerce/issues/46) (open). Ordered migrations remain the deployment source of truth; this does not block Phase H.

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

- **Phase H**: H1–H5 merged (`main` at `4b5ff6abc13789d4d06bb4b50d42615290becf32`, PR #48). Phase H remains in progress (D/F/G read-only context not yet surfaced). Next proposed work is the **H6 design inventory for Phase F recommendation + `SelectionTrace` context** (design only).
- **GitHub Issues**: Several tasks lack formal Issues. GitHub API access has been intermittent.
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Phase

**Phase H — H6 design inventory for Phase F recommendation + `SelectionTrace` context** (design only, not implementation), per `phase-h-slice-plan.md`. The H6 design must be reviewed and approved before any coding.

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | Phase H status + coordinator handoff |
| Claude | Primary coordination + implementation + review | Available | H5 merged (PR #48); H6 design inventory (next) |
| GitHub Copilot | Implementation contributor | Quota-limited | Phase H review assistance |
| Phil | Product owner | Available as schedule permits | Phase H authorization |

Capacity status should be updated whenever a provider limit is reached or resets.
