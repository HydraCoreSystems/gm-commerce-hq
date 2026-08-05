# Phase F Slice 5 — Photography Recommendation Service

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-f-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§6, §11, §14, §18, §23)
- `HydraCoreSystems/gm-commerce`: existing recommendation services in `lib/recommendations/price.ts`, `lib/recommendations/taxonomy.ts`, `lib/recommendations/collections.ts`, `lib/recommendations/marketplace-suitability.ts` (patterns to follow)
- `HydraCoreSystems/gm-commerce`: `lib/vision/types.ts` (vision inference types and predicates), `lib/photos/types.ts` (InspectionResult, photo pipeline types)
- `HydraCoreSystems/gm-commerce-hq`: `PHOTO_PIPELINE_SCOPE.md` (Gathering Moss photo standards)

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `f242e4254bbde8bb9cf0798b2ef2b930370a2fcf` |
| Branch name | `agent/phase-f-slice-5-photography` |

## Objective

Build the fifth core recommendation service. Photograph evaluation assesses whether a subject's photo assets meet Gathering Moss standards for marketplace listing readiness. The service produces one `Recommendation` (`kind = 'photography'`) with one complete §18 `SelectionTrace` per invocation.

## Required behavior

### Data sources

The service evaluates photography readiness from two input domains:

1. **Vision-derived claims** (`vision.*` predicates, from Phase D). These already exist as `canonical_claims` with predicates:
   - `vision.plant_vs_pot_vs_unusable` — one claim per photo, classification + confidence
   - `vision.composition_framing` — one claim per photo, hero-slot suitability
   - `vision.duplicate_view_detection` — one claim per photo pair, boolean duplicate judgment
   - `vision.expected_shot_type_presence` — one claim per expected shot type

2. **Photo-set state** (GMCOM-011 legacy tables `photo_sets`, `photo_assets`, `photo_derivatives`). Read directly via Supabase queries scoped to the subject's SKU. Relevant signals:
   - `photo_sets.status` — `pending`, `processing`, `needs_review`, `approved`
   - `photo_sets.hero_photo_id` — whether a hero image has been selected
   - `photo_assets.inspection_result` — JSONB containing `blurScore`, `exposureMean`, `perceptualHash`, `warnings`
   - `photo_derivatives.alt_text` — whether alt text exists for each derivative
   - Derivative counts per type (`square_marketplace`, `detail_uncropped`)
   - `photo_sets.approved_at` — when the set was last approved (freshness signal)

The vision claims and photo-set state are combined into a unified photography readiness assessment. The service does NOT create intermediate claims for photo-set state — the legacy tables are read directly, the same way compliance gate outcomes are read directly.

### Recommendation value shape

```ts
export const PHOTOGRAPHY_RECOMMENDATION_KIND = "photography";

// Photography predicates the service examines:
// - vision.plant_vs_pot_vs_unusable
// - vision.composition_framing
// - vision.duplicate_view_detection
// - vision.expected_shot_type_presence
export const PHOTOGRAPHY_VISION_PREDICATES = [
  "vision.plant_vs_pot_vs_unusable",
  "vision.composition_framing",
  "vision.duplicate_view_detection",
  "vision.expected_shot_type_presence",
] as const;

export type PhotographyStatus =
  | "photography_assessed"
  | "no_photography_signals";

export interface PhotographySignal {
  signal: string;          // "hero_selected", "alt_text_complete", "derivatives_present",
                           // "photo_set_approved", "vision_analysis_present",
                           // "blur_flagged", "exposure_flagged", "near_duplicate_flagged",
                           // "unusable_photo_present"
  status: "pass" | "fail" | "warning" | "not_applicable";
  detail: string;          // human-readable explanation
  source: string;          // "photo_set", "vision_claim", or specific claim ID
}

export interface PhotographyRecommendationValue {
  kind: typeof PHOTOGRAPHY_RECOMMENDATION_KIND;
  subjectType: "SKU" | "ProductConcept";
  subjectId: string;
  signals: PhotographySignal[];
  visionClaimsExamined: number;
  status: PhotographyStatus;
  complianceGate: ComplianceGateContext;
  freshness: {
    freshnessAtDecision: string;
  };
}
```

A `PhotographySignal` is derived per signal category, not per claim. Vision claims are aggregated: e.g., one `"blur_flagged"` signal for the whole set (not one per photo), surfacing the count of flagged photos. Conflicts for the same signal across multiple vision claims are resolved with the same precedence rules as F2–F4.

### Precedence rules

When two vision claims reach contradictory conclusions about the same signal (e.g., one claim says a photo is `unusable` and another says `plant`), resolve with the identical precedence mechanism used by F2–F4:

1. `computePrecedenceRank` (owner override > verified > under_review > stale)
2. Confidence (higher wins within same rank)
3. Freshness (`establishedAt` tiebreaker)
4. ID (deterministic tiebreaker)

Record every conflict in `precedenceDecisions` with the exact rule used. Reject every losing claim in `claimsRejected` with a real reason.

### Confidence

- Averaged across vision claims, with conflict and staleness discounts (identical constants to F2–F4: `CONFLICT_CONFIDENCE_DISCOUNT = 0.9`, `STALE_CLAIM_CONFIDENCE_DISCOUNT = 0.85`)
- Photo-set signals contribute a fixed `0.85` confidence weight per signal (hero/alt-text/derivatives/approved are deterministic, not claim-based, so their confidence is high but not 1.0 to leave room for the vision layer)
- Overall confidence: average of (vision claims confidence average) and (photo-set signals confidence average), bounded to [0, 1]
- `"no_photography_signals"` returns confidence 0

### Compliance gate

- Identical interaction to F2–F4
- Query `gmcom_compliance_gate_outcome` when `commercePackageId` is provided
- Surface as `complianceGate` context on the value
- FAIL/STALE/NO_CHECK sets `blockedFromApproval: true` but never suppresses recommendation generation
- No compliance gate decision is made by this service

### Environment and subject isolation

- Private constructor gated by module-private `Symbol` token
- `resolveTrustedEnvironment()` bound at instance creation
- No method accepts an environment override
- Claims filtered by subject type, subject ID, and environment via `queryApplicableClaims`

### SelectionTrace completeness

Every invocation produces exactly one §18 SelectionTrace containing:
- `claimsConsidered`: all vision claims examined
- `claimsRejected`: every claim excluded from active consideration, with a real reason
- `precedenceDecisions`: every within-signal conflict resolution with the rule used
- `confidence`: computed as described above
- `freshnessAtDecision`: ISO timestamp
- `policiesApplied`: any policy snapshots carried by winning claims + caller-supplied IDs
- `ownerDecisionAnchors`: `OwnerDecisionId`s of any vision claims that have been confirmed

### Atomic persistence

- Use `recordRecommendation` via `gmcom_record_recommendation` (same as all F1–F4 services)
- Recommendation and trace are written in one database transaction
- No migration needed — generic recommendation/trace schema covers `kind = 'photography'`

## Files to create

| File | Purpose |
|---|---|
| `lib/recommendations/photography.ts` | Service + pure computation |
| `lib/recommendations/photography.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | Add `phase-f-slice5-photography-live` job after the F4 job |

## Test coverage required

### Unit tests (pure computation, no database)

1. **Vision claim aggregation**: multiple `vision.plant_vs_pot_vs_unusable` claims aggregated into one `blur_flagged` and one `unusable_photo_present` signal
2. **Conflicting vision claims**: two claims with contradictory classifications for the same signal, resolved by precedence — loser rejected with a real reason
3. **Owner-override precedence**: an owner-override vision claim beats a verified claim about the same signal
4. **Photo-set signal derivation**: `hero_selected`, `alt_text_complete`, `derivatives_present`, `photo_set_approved` signals derived correctly from mock photo-set state
5. **Confidence bounds**: overall confidence stays within [0, 1]; `no_photography_signals` returns 0
6. **Deterministic ordering**: signals ordered deterministically; trace stability across repeated builds
7. **Exactly one trace**: output contains exactly one SelectionTrace
8. **§18 trace completeness**: all trace fields populated, including `claimsRejected` for inactive/unparseable/superseded/rejected vision claims
9. **Non-vision claims filtered**: claims with non-`vision.*` predicates are excluded from consideration
10. **Signal confidence discounting**: conflict and staleness discounts apply within a signal category
11. **No-claims edge case**: zero vision claims + no photo set produces `no_photography_signals` with confidence 0
12. **Rejected/inactive vision claims**: superseded/rejected status claims are excluded with real reasons
13. **Compliance gate context**: PASS/FAIL/STALE/NO_CHECK each produce correct `blockedFromApproval` without suppressing recommendations
14. **Subject isolation**: claims for a different subject do not leak in

### Live-Postgres CI job

A new `phase-f-slice5-photography-live` job in `ci.yml`, inserted after the existing F4 job, exercising against real Postgres:

- Vision-claim precedence round-trip (two conflicting `vision.plant_vs_pot_vs_unusable` claims for the same photo, verified beat under_review)
- Photo-set state read (create a `photo_set` + `photo_asset` with an `inspection_result`, verify signal derivation)
- FAIL-gate context (compliance gate FAIL surfaced as context, recommendation still produced)
- Confidence bounds enforced by DB CHECK constraint (confidence 1.5 rejected)
- Malformed-trace rejection (precedence decision missing rule rejected)
- Trace cardinality (exactly one trace per recommendation)

Use the F4 CI job as the exact structural template — same SQL pattern, same `gmcom_record_recommendation` round-trip verification, same notice/raise error discipline.

## Explicit exclusions

- **Do not** create new claim predicates for photo-set state (hero, alt text, derivatives). The photo-set signals are derived from the legacy tables, not from claims. Creating photo-claim predicates would require a migration and broader architectural decisions about the GMCOM-011-to-canonical bridge.
- **Do not** modify `lib/photos/` or `lib/vision/`. This service reads their outputs, never changes them.
- **Do not** add UI, review surfaces, or policy editing.
- **Do not** add publishing adapters or remote publication verification.
- **Do not** add SEO, merchandising, or any other recommendation kind.
- **Do not** add real vendor vision calls (no OpenAI vision API — this service reads already-existing claims).
- **Do not** add a migration unless a real schema limitation is demonstrated and approved separately.
- **Do not** modify `lib/recommendations/types.ts` or `lib/recommendations/repository.ts` — the generic recommendation infrastructure already supports `kind = 'photography'`.
- **Do not** change the `SelectionTrace` schema or `gmcom_record_recommendation`.

## Verifications before opening PR

Run locally before pushing:

```bash
npx vitest run lib/recommendations/photography.test.ts
npx tsc --noEmit
npm run build
```

All must pass.

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count (unit + live-Postgres)
- CI job name added to `ci.yml`
- Any implementation decisions made (record in handoff, flag durable ones for `DECISIONS.md`)
- Confirmation that no migration was added

## Stop condition

Stop when:
- All acceptance criteria are met
- All tests (unit + CI live-Postgres) pass
- Typecheck and build pass
- PR is opened against `main`
- Handoff is written

Do not begin implementation of Slice 6 (SEO) or any other work.
