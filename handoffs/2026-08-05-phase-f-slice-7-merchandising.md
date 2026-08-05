# Phase F Slice 7 Handoff — Merchandising Recommendation Service

## Task

- Issue: Phase F Slice 7 (merchandising recommendation service) implementation — the final Phase F slice
- Objective: Implement the merchandising recommendation service producing one `Recommendation` (kind `merchandising`) with one complete §18 `SelectionTrace` per invocation, following the exact patterns of merged Slices 2–6
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `agent/phase-f-slice-7-merchandising`

## PR

- **PR #29**: https://github.com/HydraCoreSystems/gm-commerce/pull/29
- **Head SHA**: `3d262d395387f6ac154b8c62ee6674cbee44ad9a`
- **Base SHA**: `7063838434c3ac0432786c677453c430c67cda56` (brief's verified base; confirmed against `origin/main` HEAD before branching)
- Mergeability: MERGEABLE (no conflict)

## Work Completed

- Implemented `lib/recommendations/merchandising.ts`: pure `buildMerchandisingRecommendation` + guarded `MerchandisingRecommendationServiceImpl` via `createMerchandisingService` (symbol-token private constructor, environment bound at construction, atomic persistence through `recordRecommendation`/`gmcom_record_recommendation`, trust boundary identical to Slices 2–6).
- Claims-only data source: filters `queryApplicableClaims` by the `merchandising.` predicate **prefix** (forward-compatible). Today no `merchandising.*` predicates exist, so the service produces the honest `no_merchandising_claims` baseline with a complete §18 trace (empty candidates, confidence 0, full freshness/policies metadata).
- Multi-candidate design (mirrors F4): 4 candidate types (`cross_sell`, `upsell`, `bundle`, `portfolio_position`) legitimately coexist; conflicts resolved WITHIN the same `type`+`targetSubjectId` pair, never a false single winner, exactly ONE §18 SelectionTrace per recommendation.
- Precedence for same-pair conflicts: owner-override always wins; verified beats under_review; higher confidence wins within same status; freshness tiebreaker; deterministic ID tiebreaker. Every conflict recorded in `precedenceDecisions`; every loser rejected in `claimsRejected` with a real reason.
- Confidence: average of winning candidates' confidences with F2–F6 conflict (0.9) / staleness (0.85) discounts; 0 for `no_merchandising_claims`; always bounded [0,1].
- Compliance gate identical to F2–F6 (FAIL/STALE/NO_CHECK surfaced as context, never suppresses).
- Added `phase-f-slice7-merchandising-live` CI job in `.github/workflows/ci.yml` (after the F6 job): multi-candidate round-trip (two type+target pairs → two candidates, one trace), same type+target conflict (verified vs under_review → one winner, precedence decision recorded), no-claims baseline (confidence 0, complete trace), FAIL-gate context case, confidence 1.5 DB rejection, malformed precedenceDecisions (missing `rule`) DB rejection, trace cardinality checks.

## Verification

- Tests: **37 passed** (merchandising), **987 passed** (full suite)
- Typecheck: `npx tsc --noEmit` passed
- Build: `npm run build` passed (only pre-existing unrelated `libheif-js` critical-dependency warning)
- CI: all checks green on both push and PR runs, including the new `phase-f-slice7-merchandising-live`, `verify`, `schema-from-empty`, `schema-drift-deferred`

## Implementation Decisions (flagged for DECISIONS.md)

1. **Live-CI conflict fixture uses verified-vs-under_review, not owner-override-vs-verified.** The Phase B capability gate (`gmcom_guard_owner_authority_transition`, migration `20260803070000_phase_b_slice2_verification_capability_gate.sql`) fail-closes ALL owner-authority transitions in every environment — `canonical_owner_decisions` inserts are rejected (FK + trigger), so a real owner-override claim cannot be inserted in live CI. This matches F2–F6, which all live-tested with empty `ownerDecisionAnchors`. Owner-override precedence is fully covered by unit tests (scenario 3). The live conflict fixture exercises the identical precedence machinery (one winner, loser rejected, precedence decision round-trips).

## Confirmation

- **No migration was added** (schema work deferred, consistent with Slices 2–6).

## Stop Condition

Per the brief: all acceptance criteria met, all tests pass, PR open against `main`, handoff written. **This is the final Phase F slice — no Phase G work begun.**
