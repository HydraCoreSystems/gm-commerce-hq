# Phase F Slice Plan — Core Recommendation Services

_Authoritative plan covering the seven core recommendation services defined in PRODUCT_RESET_2026-08-03.md §22 (§23 Phase F). Last updated: 2026-08-05. Implementation briefs live in `briefs/`._

## Dependency

All Phase F slices depend on:

- Phase F Slice 1 (merged): `SelectionTrace` schema, `canonical_recommendations` + `canonical_selection_traces` tables, `gmcom_record_recommendation` function, `RecommendationRepositoryImpl`.
- Phase E (merged): compliance-check infrastructure, `gmcom_compliance_gate_outcome`, fail-closed gate.
- Phase D (merged): canonical claims and evidence model, `createClaimEvidenceRepository`.

Each slice builds exactly one recommendation service. Services share the same `SelectionTrace` contract (§18), repository, compliance-gate interaction pattern, and trust-boundary discipline. No slice adds a migration unless a real schema limitation is demonstrated.

## Slice inventory

| Slice | Service | Kind | PR | Status | Base SHA |
|---|---|---|---|---|---|
| F1 | SelectionTrace foundation | (infrastructure) | #23 | Merged | `1ba4ac9` |
| F2 | Price | `price` | #24 | Merged | `d565e65` |
| F3 | Taxonomy + Collections | `taxonomy` / `collections` | #25 | Merged | `d258ab3` |
| F4 | Marketplace suitability | `marketplace_suitability` | #26 | Merged | `40e672f` |
| F5 | Photography | `photography` | #27 | Merged | `f242e42` |
| F6 | SEO | `seo` | #28 | Merged | `a46a04` |
| F7 | Merchandising | `merchandising` | #29 | Merged | `7063838` |

Each slice depends strictly on the previous slice's merge into `main`. No parallel service branches.

## Per-slice specification

### F1 — SelectionTrace foundation (merged)

- `lib/recommendations/types.ts` — `SelectionTrace`, `RecommendationDraft`, `RecommendationRecordResult`.
- `lib/recommendations/repository.ts` — `RecommendationRepositoryImpl`, atomic `recordRecommendation` via `gmcom_record_recommendation`.
- Migration `20260804120000_phase_f_slice1_selection_trace.sql`.
- CI: `phase-f-slice1-selection-trace-live`.

### F2 — Price recommendation (merged)

- `lib/recommendations/price.ts` — `PRICE_RECOMMENDATION_KIND = 'price'`, `PriceRecommendationServiceImpl`.
- Evidence-gated: price claims always require owner confirmation (§14). Service produces candidates only.
- Compliance-gate interaction: surfaces gate outcome, never suppresses recommendation.
- CI: `phase-f-slice2-price-recommendation-live`.

### F3 — Taxonomy + Collections (merged)

- `lib/recommendations/taxonomy.ts` — `TAXONOMY_RECOMMENDATION_KIND = 'taxonomy'`.
- `lib/recommendations/collections.ts` — `COLLECTIONS_RECOMMENDATION_KIND = 'collections'`.
- Both grouped as categorization-style services in one slice per implementer convenience.
- CI: `phase-f-slice3-taxonomy-collections-live`.

### F4 — Marketplace suitability (PR #26)

- `lib/recommendations/marketplace-suitability.ts` — `MARKETPLACE_SUITABILITY_RECOMMENDATION_KIND = 'marketplace_suitability'`.
- Multiple marketplaces coexist as independent candidates; conflicts resolved within a single marketplace key.
- Precedence: owner override > verified > under_review > stale, with confidence/freshness tiebreakers.
- Compliance-gate context surfaced; FAIL/STALE/NO_CHECK never suppress recommendation.
- CI: `phase-f-slice4-marketplace-suitability-live`.

### F5 — Photography (not started)

Proposed scope:

- `lib/recommendations/photography.ts` — `PHOTOGRAPHY_RECOMMENDATION_KIND = 'photography'`.
- Evaluate photo-set readiness, quality signals, derivative completeness against Gathering Moss standards.
- Surface photography-related claims with precedence resolution.
- Produce exactly one §18 `SelectionTrace`.
- Compliance-gate interaction identical to F2–F4.
- Live-Postgres CI job mirroring the existing F4 pattern.
- Exclude: camera hardware integration, real photo capture, ML-based quality models.

### F6 — SEO (not started)

Proposed scope:

- `lib/recommendations/seo.ts` — `SEO_RECOMMENDATION_KIND = 'seo'`.
- Evaluate SEO claims (title length, description completeness, keyword coverage, alt text).
- Surface SEO-related claims with precedence resolution.
- Produce exactly one §18 `SelectionTrace`.
- Compliance-gate interaction identical to prior slices.
- Live-Postgres CI job.
- Exclude: external SEO tooling, SERP scraping, competitive analysis.

### F7 — Merchandising (not started)

Proposed scope:

- `lib/recommendations/merchandising.ts` — `MERCHANDISING_RECOMMENDATION_KIND = 'merchandising'`.
- Evaluate cross-sell, upsell, bundling, and portfolio-positioning claims.
- Surface merchandising claims with precedence resolution.
- Produce exactly one §18 `SelectionTrace`.
- Compliance-gate interaction identical to prior slices.
- Live-Postgres CI job.
- Exclude: automated pricing strategy, demand forecasting, inventory-level optimization.

## Shared invariants across all slices

1. Exactly one `SelectionTrace` per service invocation.
2. Trust boundary: private constructor + module-private symbol token; environment resolved from trusted config.
3. Pure computation function exported separately for unit-testing without a database.
4. `gmcom_record_recommendation` for atomic Recommendation + trace persistence.
5. Compliance-gate context surfaced; FAIL/STALE/NO_CHECK never suppress recommendation.
6. Confidence bounded [0,1] with documented discount rules.
7. Claims filtered by predicate, subject, and environment.
8. No migration unless a real schema limitation is demonstrated.
9. CI job per slice: round-trip precedence, gate context, confidence bounds, trace cardinality, malformed-trace rejection.

## Authorization

Phil authorizes each slice's implementation before work begins. This plan records the slice ordering and dependencies. Individual slice briefs provide exact scope, acceptance criteria, and exclusions.
