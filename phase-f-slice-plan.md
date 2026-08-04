# Phase F slice plan — core recommendation services

Status: **proposed, staged for distribution to coders. Slice F1 is
authorized to begin (Phil, 2026-08-04); F2 onward each require their
own separate, explicit authorization after the prior slice lands as an
unmerged, independently-reviewed PR — same discipline as Phase E.**

Scope authority: `PRODUCT_RESET_2026-08-03.md` §23 ("Phase F — core
recommendation services (price, taxonomy, collections, marketplace
suitability, photography, SEO, merchandising — Revision 2 §9), each
producing a `SelectionTrace` (§18)"), §5 (`Recommendation` entity,
already one of the eighteen canonical types), §18 (`SelectionTrace`
schema and the knowledge-lifecycle state machine it makes queryable).

Prerequisite: Phase E (independent validation and marketplace-
compliance gates) is merged to `gm-commerce` `main` as of `1ba4ac9`
(all four slices, CI green). Phase F assumes that tip as its starting
point, and assumes every recommendation it produces can be blocked
from `CommercePackage.approved` by Phase E's fail-closed gate if its
underlying `ComplianceCheck` is `FAIL`/`STALE` — Phase F does not
re-implement or bypass that gate.

## Current-state check (verified against `gm-commerce` `main` before
writing this plan)

- `canonical_recommendations` table already exists (Phase B Slice 1):
  `id`, `subject_type`, `subject_id`, `kind`, `value jsonb`, full
  RecordContext envelope, append-only-capable grants. It is generic —
  no recommendation-kind-specific shape, and no `SelectionTrace`
  linkage exists yet (`value` is untyped `jsonb`).
- **No `SelectionTrace` table or type exists anywhere yet.** §18
  specifies its exact shape: `recommendationId`, `claimsConsidered[]`,
  `claimsRejected: {claimId, reason}[]`, `precedenceDecisions:
  {conflictBetween, winner, rule}[]`, `confidence`,
  `freshnessAtDecision`, `policiesApplied: PolicyId[]`,
  `ownerDecisionAnchors: OwnerDecisionId[]`. This is net-new schema
  work, same boundary-first approach Phase D Slice 1 and Phase E
  Slice 1 both used before any vendor/validation logic existed.
- None of the seven recommendation services (price, taxonomy,
  collections, marketplace suitability, photography, SEO,
  merchandising) exist in this codebase today in canonical form. No
  legacy code is being migrated here (unlike Phase E's
  `lib/commerce-readiness/*` migration) — this is new build against
  the canonical `CommercePackage`/`Claim`/`Evidence` model.
- Phase E's derive-don't-freeze pattern (`gmcom_current_policy`,
  `gmcom_current_compliance_check_status`) is the load-bearing
  precedent for anything in Phase F that needs a notion of "current"
  recommendation for a subject — recommendations are also naturally
  append-only historical facts (a later recommendation supersedes an
  earlier one; it does not overwrite it), so any "current
  recommendation" derivation must follow the same pattern, never a
  mutable flag.

## Slices (finite, dependency-ordered — each builds on the previous
slice's tip)

### F1 — `SelectionTrace` schema and the `Recommendation` contract (data-model boundary only)
**Authorized to begin now.**
- New `canonical_selection_traces` table implementing §18's exact
  schema above, RLS/environment-scoping consistent with every other
  canonical table, append-only (a trace is a historical fact about
  how a recommendation was made — it is never edited after the fact).
- Extend `canonical_recommendations` (or confirm its existing `jsonb`
  `value` column can cleanly carry a typed discriminated union) so
  every `Recommendation` row can resolve exactly one
  `SelectionTrace` via `selectionTraceId` (§18: "every Recommendation
  has exactly one SelectionTrace, queryable via `getSelectionTrace`").
- A `gmcom_current_recommendation()`-style derivation function (mirror
  the Phase C/E pattern) for "what is the current recommendation for
  this subject" — derived, not a mutable flag.
- Sibling `lib/recommendations/{types,repository}.ts` module (mirror
  `lib/compliance/*`'s construction discipline) exposing
  `recordRecommendation`, `getCurrentRecommendation`,
  `getSelectionTrace`. Do not add `Recommendation`/`SelectionTrace` as
  new entries to `lib/canonical/entities.ts`'s 18-type list —
  `Recommendation` is already in that list backed by
  `canonical_recommendations`; `SelectionTrace` is a new sibling type
  the same way `ComplianceCheck` was for Phase E.
- **No actual recommendation logic yet** — no price/taxonomy/SEO/etc.
  algorithms, no claim-precedence resolution, no calls into any of the
  seven services. This slice only proves the schema/contract, same as
  every other Phase's Slice 1.
- Live-Postgres CI coverage (existing RLS/append-only/GUC-scoping
  pattern) for the new table and derivation function.

### F2 — First recommendation service: price
- The first of the seven services, chosen first because it is the
  most bounded and most directly evidence-gated (claims about price
  are exactly the kind of claim §14 already requires Phil's
  confirmation for policy/price/safety/compliance/editorial/brand
  claims).
- Produces real `Recommendation` rows (`kind = 'price'`) with a real
  `SelectionTrace` per recommendation: which `Claim`s were considered,
  which were rejected and why, precedence decisions between
  conflicting claims, confidence, freshness at decision time, which
  policies were applied.
- Must consume Phase E's compliance gate correctly: a price
  recommendation for a `CommercePackage` whose current
  `ComplianceCheck` is `FAIL`/`STALE` must still be produced (it's a
  candidate, not a publish) but must be clearly marked as blocked from
  approval — do not silently suppress the recommendation or silently
  ignore the gate.
- Live-Postgres CI: real claim-precedence scenarios (conflicting price
  claims, stale claims, missing evidence) with real fail-closed
  assertions.

### F3 — Taxonomy and collections recommendation services
- Grouped because both are categorization-style recommendations
  operating over the same `ProductConcept`/`SKU`/`Claim` inputs and
  are lower-risk than price (no §14-mandated-confirmation claim types
  involved by default, unless a specific claim is flagged as such).
- Same `Recommendation`/`SelectionTrace` contract as F2, same
  compliance-gate interaction, same live-CI discipline.

### F4 — Marketplace-suitability recommendation service
- Determines whether/how a `CommercePackage` should be recommended
  for a specific marketplace — this is a **recommendation**, distinct
  from Phase E's compliance gate, which is a **block**. Do not merge
  these two concepts: suitability recommends where to sell; the
  compliance gate independently blocks approval regardless of any
  suitability recommendation.

### F5 — Photography recommendation service
- Consumes Phase D's vision-provider claims as read-only input (per
  §23/Phase D's own contract) — does not modify Phase D's vision
  pipeline. Produces recommendations about which images/angles/edits
  to use, with `SelectionTrace` linking back to the specific vision
  claims considered.

### F6 — SEO recommendation service
- Same contract, operating over title/description/keyword claims.

### F7 — Merchandising recommendation service
- Same contract; last of the seven per no stated ordering dependency
  beyond F1's schema. Confirm at authorization time whether any
  cross-service dependency (e.g. merchandising consuming price/
  taxonomy recommendations as inputs) requires resequencing before
  Phil authorizes this slice.

### F8 — Persisted-artifact wiring and read-only recommendation review surface
- Same shape as Phase E Slice 4: confirm all seven services'
  recommendations and `SelectionTrace`s are threaded by one
  `correlationId` per run alongside Phase E's compliance artifacts;
  one read-only review page (mirroring Phase D Slice 4 / Phase E
  Slice 4's pattern) surfacing recommendations + their traces +
  compliance-gate status per `CommercePackage`. No approve/override
  UI — that is §14's `Correction`/`OwnerDecision` flow, out of scope.

## Explicitly out of scope for Phase F (do not let any slice's agent
drift into these)
- §14's `Correction`/`OwnerDecision` confirmation flow and §15's
  RBAC-gated policy changes — that is Phase G.
- The completed-review UI redesign — that is Phase H, deliberately
  sequenced after Phase F/G so it has real recommendation-rich data to
  display.
- Etsy, public editorial publishing, broader proactive portfolio
  recommendations, Skrybix contract expansion — later/unsequenced
  phases per §23.
- Modifying Phase D's vision contract or Phase E's compliance-check/
  gate logic — Phase F consumes both as read-only inputs.
- Any UI beyond the one read-only review page in F8.

## Process, same discipline used for Phases C/D/E
- Each slice branches from the prior slice's tip
  (`agent/phase-f-slice-N-...`), stacked, one PR per slice.
- No PR is merged without Phil's explicit, separate authorization per
  slice — authorization for one slice is not authorization for the
  next.
- Every "done" report must be independently verified against real
  GitHub state (branch existence, exact HEAD SHA, real CI conclusion
  polled to completion, actual file diff scope) before being reported
  as complete — self-reported completion from any agent, cheap or
  expensive, is not sufficient on its own.

---

**F1 is authorized to begin. F2 through F8 each require their own
separate authorization from Phil after the prior slice lands as an
unmerged PR and passes independent review.**
