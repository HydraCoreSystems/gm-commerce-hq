# Current Project Status

_Last updated: 2026-08-05_

## Phase F — Core Recommendation Services (active)

Implementation has resumed under the product-reset architecture. Phase F builds the seven core recommendation services defined in `PRODUCT_RESET_2026-08-03.md` §23. Each service produces a §18 `SelectionTrace`. The authoritative slice plan is `phase-f-slice-plan.md` (this repo).

### Merged

| Slice | PR | Service | Base SHA |
|---|---|---|---|
| F1 | #23 | SelectionTrace foundation | `1ba4ac9` |
| F2 | #24 | Price recommendation | `d565e65` |
| F3 | #25 | Taxonomy + Collections | `d258ab3` |
| F4 | #26 | Marketplace suitability | `40e672f` |
| F5 | #27 | Photography | `f242e42` |
| F6 | #28 | SEO | `a46a04` |
| F7 | #29 | Merchandising | `7063838` |

`gm-commerce/main` is at `4eac37` (PR #29 merge). **Phase F complete.** All seven core recommendation services merged with CI-verified test coverage. Typecheck, tests, build, schema-from-empty, and all live-Postgres jobs pass.

### Phase F complete

No remaining Phase F work. Phase G (owner-editable policies and learned-rule activation, per `PRODUCT_RESET_2026-08-03.md` §23) is the next phase.

See `phase-f-slice-plan.md` for per-slice scope, acceptance criteria, and exclusions.

## Product Reset

`PRODUCT_RESET_2026-08-03.md` is at Revision 3 (corrected), awaiting Copilot's focused re-review. Awaiting `APPROVE FOR IMPLEMENTATION` / `APPROVE AFTER DOCUMENTED CORRECTIONS` / `REQUIRES REVISION 4`. The reset does not block Phase F — Phase F is explicitly sequenced in the reset's own §23 and the merged slices follow the reset's architecture (canonical entities, claim model, compliance gate, SelectionTrace).

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

- **Live database migration gap**: The `commerce_details` table and `listing_packages.content_provenance` column exist in the live Supabase database but the migration files are uncommitted (see `PRODUCT_RESET_2026-08-03.md` §1, §1.1). CI cannot reproduce the schema from git alone. This is a pre-Phase-A blocker per the reset document.
- **`HY-LOB01-C04` test data**: Quarantine disposition pending Phil's direction (see `PRODUCT_RESET_2026-08-03.md` §1.2). Records untouched per standing constraint — no deletion without Phil's explicit approval.
- **GitHub Issues**: Several tasks (GMCOM-012, Phase F slices) lack formal Issues.
- **Shopify CSV export**: Phil to provide a current export for GMCOM-014 real-export validation.

## Current Blockers

- Product reset awaiting Copilot's re-review (does not block Phase F but gates broader architectural decisions).
- No live channel between AI contributors; coordination depends on Phil relaying.
- AI provider usage limits not automatically visible.

## Next Highest-Priority Tasks

1. Coordinator prepares Slice 5 (photography) implementation brief from `main` at `f242e42`.
2. Commit the undocumented live-database migration so the schema is reproducible (reset §1.1 prerequisite).

## AI Capacity

| Contributor | Role | Capacity | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / coordinator | Available | Headquarters reconciliation, slice planning |
| Claude | Primary implementation | Available | Phase F Slices 1-4; awaiting next assignment |
| GitHub Copilot | Implementation contributor | Quota-limited | PR #26 review skipped; prior work on GMCOM-003/005/010/013/014 |
| Phil | Product owner | Available as schedule permits | Merge authorization for PR #26; Shopify CSV export; reset review |

Capacity status should be updated whenever a provider limit is reached or resets.
