# Phase G Slice 2 — Owner Confirmation Flow

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-g-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§14, §15)
- `HydraCoreSystems/gm-commerce`: `lib/canonical/intelligence-repository-contract.ts` — `IntelligenceRepositoryV1` interface, `REPOSITORY_V1_CAPABILITIES`, `REPOSITORY_V1_OPERATION_POLICIES`
- `HydraCoreSystems/gm-commerce`: `lib/canonical/claims/repository.ts` — existing `recordOwnerDecision` implementation (lines ~474), `proposeClaim` owner_override logic
- `HydraCoreSystems/gm-commerce`: `lib/canonical/claims/types.ts` — `OwnerDecisionDraft` type
- `HydraCoreSystems/gm-commerce`: `lib/recommendations/price.ts` (lines ~20-22) — example of a recommendation service that references Phase G's confirmation flow
- `HydraCoreSystems/gm-commerce`: `supabase/migrations/20260803030000_phase_b_slice1_canonical_entities.sql` — `canonical_owner_decisions` table schema

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `a24ad83386933e6e5fe786206c29f7d9bb74c135` |
| Branch name | `agent/phase-g-slice-2-owner-confirmation` |

## Objective

Build the §14 owner confirmation flow. An owner sees a Phase F recommendation (e.g., `kind = 'price'`, `kind = 'taxonomy'`), confirms it, and the system records an `OwnerDecision` in `canonical_owner_decisions`. This is the flow that all 7 Phase F recommendation services reference as "confirmation flow (Phase G)."

## Required behavior

### What exists today

`recordOwnerDecision` is implemented in `lib/canonical/claims/repository.ts` (~lines 474-536) but is unreachable — it throws `UNAUTHORIZED` because `callerPrincipal` can only be `{ kind: "service" }` (the pipeline itself). The table `canonical_owner_decisions` exists with `decision_type`, `subject_type`, `subject_id`, `predicate`, `applicability_scope`, and full RecordContext columns.

### What this slice builds

1. **Principal mechanism** — a minimal, localhost-only identity that distinguishes `{ kind: "service" }` (the pipeline) from `{ kind: "owner" }` (Phil/Crystal). No full auth system. A `GM_COMMERCE_CALLER` environment variable or a simple `x-gmcommerce-caller` header pattern, with `owner` being the value that gates privileged operations. Default is `service` when unset.

2. **`recordOwnerDecision` unblocking** — when `callerPrincipal.kind === "owner"`, the method proceeds instead of throwing `UNAUTHORIZED`. Validates that the recommendation exists, the subject is real, and the decision is well-formed. Creates a `canonical_owner_decisions` row.

3. **Confirmation service** — `lib/confirmation/confirmation-service.ts`. A dedicated service that:
   - Accepts a `RecommendationId` + owner identity
   - Reads the recommendation and its §18 SelectionTrace
   - Creates an `OwnerDecision` linked to the recommendation
   - Optionally creates or supersedes a claim with owner authority (the confirmation "activates" the recommendation into a claim that outranks all non-owner claims via the existing precedence model)

### OwnerDecision shape

Uses the existing `OwnerDecisionDraft` from `lib/canonical/claims/types.ts`:

```ts
export interface OwnerDecisionDraft {
  decisionType: string;         // e.g. "confirm_recommendation", "override_claim"
  subjectType: "SKU" | "ProductConcept";
  subjectId: string;
  predicate: string;            // e.g. "commerce.price", "taxonomy.category"
  applicabilityScope: Scope;    // §5 Scope type — where this decision applies
  detail: Record<string, unknown>;  // decision-specific payload
  recommendationId?: string;    // the Phase F recommendation being confirmed
  claimId?: string;            // the claim being overridden (for override decisions)
}
```

### Confirmation flow (the core interaction)

```
Phase F Recommendation (price=24.95, confidence=0.72, §18 trace)
       │
       ▼
Owner reviews and confirms
       │
       ▼
recordOwnerDecision({ decisionType: "confirm_recommendation", ... })
       │
       ▼
canonical_owner_decisions row created
       │
       ▼
Optionally: proposeClaim with owner_override linked to the OwnerDecisionId
       │
       ▼
The owner-authorized claim outranks all non-owner claims
(via existing computePrecedenceRank — owner override is the top tier)
```

### Trust boundary

- `ConfirmationServiceImpl` with private constructor + module-private Symbol token
- `createConfirmationService(supabase)` — the one way to obtain an instance
- `resolveTrustedEnvironment()` bound at construction
- Principal resolution from config, never from an untrusted request parameter

## Files to create

| File | Purpose |
|---|---|
| `lib/confirmation/types.ts` | ConfirmationRequest, ConfirmationResult types |
| `lib/confirmation/principal.ts` | Principal resolution — `resolvePrincipal()`, `isOwner()` |
| `lib/confirmation/confirmation-service.ts` | ConfirmationServiceImpl — confirmRecommendation |
| `lib/confirmation/confirmation-service.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `lib/canonical/claims/repository.ts` | Unblock `recordOwnerDecision` for owner principal |
| `.github/workflows/ci.yml` | Add `phase-g-slice2-owner-confirmation-live` job |

## Test coverage required

1. **Service principal cannot record owner decisions**: `callerPrincipal.kind === "service"` → `UNAUTHORIZED`
2. **Owner principal can record owner decisions**: `callerPrincipal.kind === "owner"` → success
3. **Confirmation of a real recommendation**: returns an `OwnerDecision` with `decisionType = "confirm_recommendation"`, linked to the recommendation
4. **Confirmation of a non-existent recommendation**: throws a descriptive error
5. **Confirmation creates an owner-authorized claim**: the resulting claim has `ownerOverrideId` set and `computePrecedenceRank` places it above verified claims
6. **Duplicate confirmation is idempotent**: confirming the same recommendation twice with the same decision type returns the existing decision, does not create a duplicate
7. **Cross-subject confirmation is rejected**: confirming a recommendation for SKU-A against SKU-B's subject throws
8. **Environment isolation**: decisions in different environments are isolated
9. **Principal resolution defaults to service**: when `GM_COMMERCE_CALLER` is unset, `resolvePrincipal()` returns `{ kind: "service" }`

### Live-Postgres CI job

A new `phase-g-slice2-owner-confirmation-live` job:

- Service principal attempts `recordOwnerDecision` → rejected
- Owner principal creates decision → `canonical_owner_decisions` row with correct `decision_type`, `subject_type`, `subject_id`, `predicate`
- Decision links to a real `canonical_recommendations` row
- Owner-authorized claim created via `proposeClaim` with `owner_override` → claim's `owner_override_id` matches the decision
- Duplicate decision idempotency
- Cross-subject rejection

Use the G1 CI job as structural template.

## Explicit exclusions

- **Do not** implement §15 RBAC enforcement (Slice G5). Principal resolution is minimal — `service` vs. `owner` only.
- **Do not** implement correction capture or scope inference (Slice G3).
- **Do not** implement rule activation (Slice G4). The decision is recorded; rule activation from the decision is G4.
- **Do not** add UI, review surfaces, or approval dashboard.
- **Do not** add a full authentication system (passwords, sessions, cookies).
- **Do not** change the `recordOwnerDecision` database schema — the `canonical_owner_decisions` table is sufficient.
- **Do not** add a migration unless a real schema limitation is demonstrated.

## Verifications before opening PR

```bash
npx vitest run lib/confirmation/confirmation-service.test.ts
npx tsc --noEmit
npm run build
```

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- Principal resolution mechanism chosen
- Any implementation decisions

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. Do not start Slice G3.
