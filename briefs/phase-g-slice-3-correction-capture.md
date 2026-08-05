# Phase G Slice 3 — Correction Capture and Scope Inference

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-g-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§14)
- `HydraCoreSystems/gm-commerce`: `lib/canonical/intelligence-repository-contract.ts` — `IntelligenceRepositoryV1` contract, `recordCorrection`, `proposeCorrectionScope`
- `HydraCoreSystems/gm-commerce`: `lib/canonical/claims/types.ts` — `OwnerDecisionDraft`, `CorrectionDraft` (if defined), `Scope` type
- `HydraCoreSystems/gm-commerce`: `supabase/migrations/20260803030000_phase_b_slice1_canonical_entities.sql` — `canonical_corrections` table schema
- `HydraCoreSystems/gm-commerce`: `lib/canonical/entities.ts` — entity type definitions for scope derivation

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `e5c708b4f46e613abcf4912f67a02faa58bfa61a` |
| Branch name | `agent/phase-g-slice-3-correction-capture` |

## Objective

When Phil corrects something (a price, a taxonomy assignment, a collection membership), the system records the correction and derives what scope it could apply to. This is §14's eligible-scope derivation — the system doesn't auto-generalize, but it computes what scopes a correction *could* apply to and surfaces them for owner confirmation.

## Required behavior

### Two operations

1. **`recordCorrection(correction, context)`** — creates a `canonical_corrections` row linked to an `OwnerDecision`. Records what was corrected (`before_value`, `after_value`), the scope of the specific correction (`scope: { kind: "entity", ... }` for a single-item fix), and the rationale.

2. **`proposeCorrectionScope(correctionId)`** — derives the set of scopes this correction *could* generalize to, based on entity hierarchy and predicate semantics. Returns `{ eligibleScopes: Scope[], requiresConfirmation: boolean }`. A correction never auto-generalizes; broader scope is always a proposal.

### Scope-derivation rules (§14)

| Predicate category | Eligible scopes |
|---|---|
| `commerce.price` | `{ this SKU }`, `{ this ProductConcept }` — price is not botanical |
| `taxonomy.category`, `taxonomy.subcategory`, `taxonomy.product_type` | `{ this SKU }`, `{ this ProductConcept }`, `{ all products }` |
| `collections.*` | `{ this SKU }`, `{ this ProductConcept }` |
| Plant-specific (care, condition, shipping, rooting) | `{ this SKU }`, `{ this ProductConcept }`, `{ this genus/species }`, `{ all plants }` |
| `commerce.marketplace_suitability` | `{ this SKU }`, `{ this ProductConcept }` |
| `seo.*` | `{ this SKU }`, `{ this ProductConcept }` |
| `merchandising.*` | `{ this SKU }`, `{ this ProductConcept }` |
| Safety, compliance, policy, editorial, brand | `{ this SKU }` only — always requires owner confirmation, never broader |

Default: when the predicate doesn't match any known category, return `{ this SKU }` only.

### Confirmation rules (§14)

- Any scope broader than `{ this SKU }` always requires owner confirmation
- Price, safety, compliance, marketplace policy, editorial, and brand corrections always require confirmation regardless of scope
- `requiresConfirmation` in the return value is computed from these rules

### Value shapes

```ts
export interface CorrectionRequest {
  ownerDecisionId: string;      // the OwnerDecision that authorized this correction
  subjectType: "SKU" | "ProductConcept";
  subjectId: string;
  predicate: string;            // the claim predicate being corrected
  beforeValue: unknown;         // the value before correction
  afterValue: unknown;          // the value after correction
  rationale: string;            // why this was corrected
  scope: Scope;                 // the specific scope of this correction
                                // (initially { kind: "entity", entityType, entityId })
}

export interface CorrectionRecord {
  correctionId: string;
  ownerDecisionId: string;
  subjectType: string;
  subjectId: string;
  predicate: string;
  scope: Scope;
  rationale: string;
  beforeValue: unknown;
  afterValue: unknown;
  createdAt: string;
}

export interface ScopeProposal {
  eligibleScopes: Scope[];
  requiresConfirmation: boolean;
  derivationNotes: string;      // which rules were applied
}
```

### Trust boundary

- `CorrectionServiceImpl` with private constructor + module-private Symbol token
- `createCorrectionService(supabase)` — the one way to obtain an instance
- `resolveTrustedEnvironment()` bound at construction
- Principal check: only `{ kind: "owner" }` can record corrections (using the G2 principal mechanism)

## Files to create

| File | Purpose |
|---|---|
| `lib/correction/types.ts` | CorrectionRequest, CorrectionRecord, ScopeProposal types |
| `lib/correction/scope-inference.ts` | Predicate-to-scope derivation rules, eligibility computation |
| `lib/correction/correction-service.ts` | CorrectionServiceImpl — recordCorrection, proposeCorrectionScope |
| `lib/correction/correction-service.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | Add `phase-g-slice3-correction-capture-live` job |

## Test coverage required

1. **Record a correction**: creates a `canonical_corrections` row with correct `before_value`, `after_value`, `rationale`, linked to an `OwnerDecision`
2. **Service principal cannot record corrections**: `UNAUTHORIZED` when caller is not owner
3. **Price correction scope**: eligible scopes are `{this SKU}`, `{this ProductConcept}` — never botanical
4. **Plant care correction scope**: eligible scopes include `{genus/species}`, `{all plants}`
5. **Unknown predicate defaults to SKU-only**: safe default, never auto-generalizes
6. **Broader-than-SKU scope requires confirmation**: `requiresConfirmation = true` when eligible scopes include anything broader
7. **Price correction always requires confirmation**: regardless of scope narrowness (per §14 rule)
8. **Safety/compliance correction always requires confirmation**: same behavior
9. **Cross-subject correction rejected**: correcting a claim for SKU-A against SKU-B's subject throws
10. **Environment isolation**: corrections in different environments are isolated
11. **Propose scope for non-existent correction**: throws descriptive error

### Live-Postgres CI job

A new `phase-g-slice3-correction-capture-live` job:

- Owner records a correction → `canonical_corrections` row with correct values
- Scope proposal for price correction → eligible scopes exclude botanical groupings
- Scope proposal for plant-care correction → eligible scopes include genus/species, all plants
- `requiresConfirmation = true` for broader-than-SKU scope
- `requiresConfirmation = true` for price correction even at SKU scope
- Service principal rejected

## Explicit exclusions

- **Do not** implement rule activation (Slice G4). The correction is recorded; activation from a correction is G4.
- **Do not** implement RBAC enforcement (Slice G5). Use G2's owner principal check.
- **Do not** add UI, correction forms, or review surfaces.
- **Do not** add a migration unless a real schema limitation is demonstrated. The `canonical_corrections` table already exists.
- **Do not** change the `recordOwnerDecision` flow (G2).
- **Do not** implement the legacy-to-canonical correction migration bridge (that's Phase B0 code, already exists via `gmcom_migrate_legacy_corrections`).

## Verifications before opening PR

```bash
npx vitest run lib/correction/correction-service.test.ts
npx tsc --noEmit
npm run build
```

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- Scope-derivation rule set implemented
- Any implementation decisions

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. Do not start Slice G4.
