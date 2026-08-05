# Phase F Slice 6 — SEO Recommendation Service

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-f-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§6, §18, §23)
- `HydraCoreSystems/gm-commerce`: `lib/recommendations/photography.ts` (closest pattern — reads claims + external data source), `lib/recommendations/marketplace-suitability.ts` (multi-signal pattern)
- `HydraCoreSystems/gm-commerce`: `lib/recommendations/types.ts` (SelectionTrace, RecommendationDraft)

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `a46a04295da74fea46d2b38294284b74d08579aa` |
| Branch name | `agent/phase-f-slice-6-seo` |

## Objective

Build the sixth core recommendation service. SEO evaluation assesses whether a subject's listing content meets search-engine discoverability standards. The service produces one `Recommendation` (`kind = 'seo'`) with one complete §18 `SelectionTrace` per invocation.

## Required behavior

### Data sources

The service evaluates SEO readiness from two input domains:

1. **SEO-relevant claims** — filtered from `queryApplicableClaims` by predicate. Today no `seo.*` predicates exist, so this returns empty; the filter exists for forward compatibility when SEO claims are added. Use the predicate prefix `seo.` as the filter (not a single constant, since multiple SEO predicates may exist in the future).

2. **Listing package content** — read from `listing_packages` via direct Supabase query scoped to the subject's SKU. Evaluate deterministic signals:
   - `title` — length (characters), presence
   - `description` — length, presence
   - `tags` — count (from `tags` array)
   - `product_type` — presence

The listing packages table has these columns from committed migrations (GMCOM-007/008/009). Do not read `seo_title` or `seo_description` — those columns exist only in the undocumented live-database migration, not in committed git migrations.

### Signal thresholds (deterministic, documented)

```
title_length:
  - pass: >= 20 chars
  - fail: < 20 chars

description_length:
  - pass: >= 120 chars
  - fail: < 120 chars

tags_count:
  - pass: >= 3 tags
  - warning: 1-2 tags
  - fail: 0 tags

product_type:
  - pass: present and non-empty
  - fail: absent or empty
```

Thresholds are documented constants, not configurable policy — same discipline as F2-F5's confidence discount constants.

### Recommendation value shape

```ts
export const SEO_RECOMMENDATION_KIND = "seo";

export type SeoStatus = "seo_assessed" | "no_seo_signals";

export interface SeoSignal {
  signal: string;    // "title_length", "description_length", "tags_count", "product_type"
  status: "pass" | "fail" | "warning";
  detail: string;    // e.g. "title is 47 characters (pass)" or "description absent (fail)"
  source: string;    // "listing_package" or claim ID
}

export interface SeoRecommendationValue {
  kind: typeof SEO_RECOMMENDATION_KIND;
  subjectType: "SKU" | "ProductConcept";
  subjectId: string;
  signals: SeoSignal[];
  seoClaimsExamined: number;
  status: SeoStatus;
  complianceGate: ComplianceGateContext;
  freshness: {
    freshnessAtDecision: string;
  };
}
```

### Precedence

When an SEO claim conflicts with a deterministic listing-package signal about the same field:

1. Owner-override SEO claims always win
2. Verified SEO claims beat deterministic signals
3. Under-review SEO claims are surfaced but do not override deterministic signals
4. Deterministic listing-package signals are the baseline; they are never recorded as "losing" claims (they aren't claims), but when a claim overrides them, the conflict and rule are recorded in `precedenceDecisions`

### Confidence

- When only deterministic listing-package signals are present (no SEO claims): confidence = 0.80 (deterministic evaluation is reliable but not exhaustive)
- When SEO claims contribute: average of (claims average with F2-F5 conflict/staleness discounts) and (0.80 deterministic component)
- `"no_seo_signals"` (no listing package exists): confidence = 0
- Always bounded [0, 1]

### Listing package absence

When no `listing_packages` row exists for the subject's SKU, status is `"no_seo_signals"` with confidence 0 and empty signals. The recommendation is still produced with a complete §18 trace (claims considered/rejected recorded, precedence decisions empty, freshness timestamped). Do not throw or return null.

### Compliance gate

- Identical to F2-F5
- Query `gmcom_compliance_gate_outcome` when `commercePackageId` is provided
- FAIL/STALE/NO_CHECK sets `blockedFromApproval: true` but never suppresses recommendation

### Trust boundary

- Private constructor + module-private Symbol token
- `resolveTrustedEnvironment()` bound at construction
- No method accepts an environment override

## Files to create

| File | Purpose |
|---|---|
| `lib/recommendations/seo.ts` | Service + pure computation |
| `lib/recommendations/seo.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | Add `phase-f-slice6-seo-live` job after the F5 job |

## Test coverage required

1. **Deterministic signal evaluation**: title_length, description_length, tags_count, product_type all assessed correctly against thresholds
2. **Listing package absence**: returns `no_seo_signals` with confidence 0, trace still complete
3. **Title boundary cases**: exactly threshold length, empty, null, very long
4. **Description boundary cases**: exactly threshold length, empty, null
5. **Tags boundary cases**: 0 tags (fail), 1-2 (warning), 3+ (pass)
6. **Product type**: present, absent, empty string
7. **SEO claim vs deterministic signal conflict**: verified SEO claim overrides a deterministic signal; under-review SEO claim does not
8. **Owner-override SEO claim**: trumps both verified claims and deterministic signals
9. **Confidence bounds**: within [0, 1] for all scenarios; 0 for no-listing-package; 0.80 for deterministic-only
10. **Deterministic signal ordering**: signals emitted in fixed documented order
11. **§18 trace completeness**: all fields populated even with no claims
12. **Compliance gate context**: PASS/FAIL/STALE/NO_CHECK each produce correct blockedFromApproval
13. **Subject isolation**: listing package and claims for a different subject do not leak

### Live-Postgres CI job

A new `phase-f-slice6-seo-live` job inserted after F5, exercising against real Postgres:

- Deterministic signal round-trip (insert listing_package with title/description/tags, verify all 4 signals)
- Listing-package absence (no row → `no_seo_signals`, confidence 0)
- SEO claim vs deterministic signal conflict (verified claim with the same signal, resolved via precedence, recorded in precedenceDecisions)
- FAIL-gate context (recommendation still produced)
- Confidence bounds enforced by DB CHECK constraint
- Malformed-trace rejection
- Trace cardinality (exactly one trace per recommendation)

Copy the F5 CI job structure — same SQL pattern, same `gmcom_record_recommendation` round-trip verification.

## Explicit exclusions

- **Do not** read `listing_packages.seo_title` or `listing_packages.seo_description` — these columns exist only in the undocumented live migration, not in committed git migrations
- **Do not** create new claim predicates
- **Do not** add UI, review surfaces, or policy editing
- **Do not** add publishing adapters or remote publication verification
- **Do not** add merchandising or any other recommendation kind
- **Do not** add external SEO tooling, SERP scraping, or competitive analysis
- **Do not** add a migration unless a real schema limitation is demonstrated
- **Do not** modify `lib/recommendations/types.ts` or `lib/recommendations/repository.ts`

## Verifications before opening PR

```bash
npx vitest run lib/recommendations/seo.test.ts
npx tsc --noEmit
npm run build
```

All must pass.

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- Any implementation decisions (flag for DECISIONS.md)
- Confirmation that no migration was added

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. Do not start Slice 7.
