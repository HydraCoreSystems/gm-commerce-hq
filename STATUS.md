# Current Project Status

_Last updated: 2026-08-05_

## Phase G — Owner-Editable Policies and Learned-Rule Activation (planned)

Phase G enables Phil to edit policies and confirm Phase F recommendations into standing learned rules. The authoritative slice plan is `phase-g-slice-plan.md` (this repo).

### Planned

| Slice | Scope | Status |
|---|---|---|
| G1 | Policy management — create, query, evaluate policies | #30 | Merged | `4eac37` |
| G2 | Owner confirmation flow — recommendation → owner decision | #31 | Merged | `a24ad83` |
| G3 | Correction capture and scope inference | — | Brief ready, awaiting authorization | `e5c708b` |
| G4 | Rule activation engine — learned rules table, activate/revoke/query | Not started |
| G5 | RBAC enforcement — §15 role matrix at Repository command layer | Not started |

`gm-commerce/main` is at `4eac37`. All Phase G infrastructure prerequisites (tables, contract interface) exist from Phases B-D.

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

`PRODUCT_RESET_2026-08-03.md` is at Revision 3. Copilot's re-review returned `REQUIRES REVISION 3 AND RE-REVIEW`; Revision 3 incorporated the corrections; Codex subsequently returned `APPROVE AFTER DOCUMENTED CORRECTIONS`. Per §24, the review step is complete. The reset does not block Phase G.

## Schema State

- Migration gap resolved: `commerce_details` table and `listing_packages.seo_title`/`seo_description` columns have committed DDL at `20260802040000_commerce_readiness.sql`.
- `20260803000000_commerce_field_ownership.sql` (price/`content_provenance`) was deliberately retired as never-applied.
- CI schema-from-empty passes from committed migrations.
- `HY-LOB01-C04` test data verified absent from the live database (2026-08-05).

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
- **Phase C Slice 5** — Freshness, Revalidation, and Promotion Gates (PR #14, merged).
- **Phase D Slice 1** — Vision-provider contract and authorized-media-access boundary (built, on branch `agent/phase-d-slice-1-vision-provider`, not yet merged).
- **Phase E Slices 1–4** — ComplianceCheck, deterministic validation, fail-closed gate, review surface (PRs #19–#22, all merged).

Supabase project: `wcrcllhvgbhykbonopzx` (separate co-owner account).

## Active Work

- **Pre-Phase G audit**: Phase F comprehensive survey complete (see handoff). All 7 services consistent in trust boundary, precedence, confidence math, compliance gate interaction, and §18 trace contract. No cross-service dependencies, no TODOs, no commented-out code.
- **GitHub Issues**: Several tasks lack formal Issues. GitHub API access has been intermittent.
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Phase

**Phase G** — owner-editable policies and learned-rule activation, per `PRODUCT_RESET_2026-08-03.md` §23. No briefs exist yet.

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | Phase G planning |
| Claude | Primary implementation | Available | Phase G, awaiting slice assignment |
| GitHub Copilot | Implementation contributor | Quota-limited | Phase F slices 3-7 + handoffs |
| Phil | Product owner | Available as schedule permits | Phase G authorization |

Capacity status should be updated whenever a provider limit is reached or resets.
