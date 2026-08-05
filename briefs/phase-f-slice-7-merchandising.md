# Phase F Slice 7 — Merchandising Recommendation Service

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-f-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§6, §18, §23)
- `HydraCoreSystems/gm-commerce`: `lib/recommendations/marketplace-suitability.ts` (most similar — multi-candidate output), `lib/recommendations/seo.ts` (reads external data), `lib/recommendations/photography.ts` (reads legacy tables)
- `HydraCoreSystems/gm-commerce-hq`: `PHOTO_PIPELINE_SCOPE.md`, `PRODUCT_RESET_2026-08-03.md` §17, §18

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `7063838434c3ac0432786c677453c430c67cda56` |
| Branch name | `agent/phase-f-slice-7-merchandising` |

## Objective

Build the seventh and final core recommendation service. Merchandising evaluation assesses cross-sell, upsell, bundling, and portfolio-positioning opportunities for a subject. The service produces one `Recommendation` (`kind = 'merchandising'`) with one complete §18 `SelectionTrace` per invocation.

## Required behavior

### Data sources

The service evaluates merchandising opportunities from claims only — unlike F5/F6 which read legacy tables, merchandising claims have no legacy-table equivalent today. The service filters `queryApplicableClaims` by the `merchandising.` predicate prefix for forward compatibility.

Today, no `merchandising.*` predicates exist. The service produces a `no_merchandising_claims` recommendation with a complete §18 trace (empty claims, confidence 0, full freshness/policies metadata). This is the correct, honest baseline — the service exists so that when merchandising claims are later added (Phase B's claim model, Phase G's rule activation), consumers of the recommendation surface don't need to change.

### Recommendation value shape

```ts
export const MERCHANDISING_RECOMMENDATION_KIND = "merchandising";

export type MerchandisingStatus =
  | "merchandising_from_claims"
  | "no_merchandising_claims";

export interface MerchandisingCandidate {
  type: "cross_sell" | "upsell" | "bundle" | "portfolio_position";
  targetSubjectId: string;
  targetSubjectType: "SKU" | "ProductConcept";
  reasoning: string;
  claimId: string;
  claimStatus: string;
  confidence: number;
  establishedAt: string | null;
  lastVerifiedAt: string | null;
}

export interface MerchandisingRecommendationValue {
  kind: typeof MERCHANDISING_RECOMMENDATION_KIND;
  subjectType: "SKU" | "ProductConcept";
  subjectId: string;
  candidates: MerchandisingCandidate[];
  status: MerchandisingStatus;
  complianceGate: ComplianceGateContext;
  freshness: {
    freshnessAtDecision: string;
  };
}
```

### Multi-candidate design

Multiple merchandising candidates legitimately coexist — a single product can have cross-sell, upsell, and bundle opportunities. The service surfaces all valid candidates. Conflicts are resolved within the same `type` + `targetSubjectId` pair (two claims both saying "upsell to SKU-X" but disagreeing on reasoning/confidence).

### Precedence

When two merchandising claims conflict on the same type+target pair:

1. Owner-override claims always win
2. Verified claims beat under_review
3. Higher confidence wins within same status
4. Freshness tiebreaker
5. Deterministic ID tiebreaker

Record every conflict in `precedenceDecisions`. Reject every loser in `claimsRejected` with a real reason.

### Confidence

- Average of winning candidates' confidences with F2-F6 conflict/staleness discounts
- `"no_merchandising_claims"`: confidence 0
- Always bounded [0, 1]

### Compliance gate

- Identical to F2-F6
- FAIL/STALE/NO_CHECK never suppresses recommendation

### Trust boundary

- Private constructor + module-private Symbol token
- `resolveTrustedEnvironment()` bound at construction
- No method accepts an environment override

## Files to create

| File | Purpose |
|---|---|
| `lib/recommendations/merchandising.ts` | Service + pure computation |
| `lib/recommendations/merchandising.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | Add `phase-f-slice7-merchandising-live` job after the F6 job |

## Test coverage required

1. **Multi-candidate coexistence**: two distinct merchandising types (cross_sell + upsell) for different targets both appear as candidates
2. **Same type+target conflict**: two claims about the same type+target pair resolved via precedence, loser rejected
3. **Owner-override precedence**: owner-override claim wins over verified claim for the same type+target
4. **Verified-vs-under_review precedence**: verified win with real precedence decision
5. **Stale-claim handling**: stale claim loses to fresh; confidence discounted
6. **Deterministic candidate ordering**: candidates ordered by type then precedence then confidence
7. **No-claims baseline**: `"no_merchandising_claims"` with confidence 0, complete trace
8. **Confidence bounds**: within [0, 1] for all scenarios; 0 for no-claims
9. **§18 trace completeness**: claimsConsidered, claimsRejected, precedenceDecisions all populated
10. **Rejected/inactive claims**: superseded/rejected status excluded with real reasons
11. **Under-evidenced claim rejection**: no evidence anchors for anchoring-required source category → rejected
12. **Compliance gate context**: PASS/FAIL/STALE/NO_CHECK each produce correct blockedFromApproval
13. **Subject isolation**: claims for different subject do not leak

### Live-Postgres CI job

A new `phase-f-slice7-merchandising-live` job inserted after F6, exercising against real Postgres:

- Multi-candidate round-trip (two merchandising claims for different type+target pairs → two candidates, one trace)
- Same type+target conflict (owner-override vs verified → one winner, precedence decision recorded)
- No-claims baseline (no merchandising claims → `no_merchandising_claims`, confidence 0)
- FAIL-gate context (recommendation still produced)
- Confidence bounds enforced by DB CHECK constraint
- Malformed-trace rejection
- Trace cardinality (exactly one trace per recommendation)

Copy the F6 CI job structure.

## Explicit exclusions

- **Do not** create new claim predicates
- **Do not** add UI, review surfaces, or policy editing
- **Do not** add publishing adapters, remote publication verification, or any marketplace-specific logic
- **Do not** add automated pricing strategy, demand forecasting, or inventory-level optimization
- **Do not** add a migration unless a real schema limitation is demonstrated
- **Do not** modify `lib/recommendations/types.ts` or `lib/recommendations/repository.ts`
- **Do not** reference any future slice or phase

## Verifications before opening PR

```bash
npx vitest run lib/recommendations/merchandising.test.ts
npx tsc --noEmit
npm run build
```

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- Any implementation decisions
- Confirmation that no migration was added

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. This is the final Phase F slice — do not begin any Phase G work.
