# Phase G Slice 1 — Policy Management

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-g-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§14, §15, §23)
- `HydraCoreSystems/gm-commerce`: `lib/canonical/intelligence-repository-contract.ts` — `IntelligenceRepositoryV1` interface, `REPOSITORY_V1_CAPABILITIES`, `REPOSITORY_V1_OPERATION_POLICIES`
- `HydraCoreSystems/gm-commerce`: `lib/compliance/validation.ts` — existing policy reader (`gmcom_current_policy`) to understand consumption patterns
- `HydraCoreSystems/gm-commerce`: `supabase/migrations/20260804030000_phase_c_slice5_freshness_revalidation.sql` — `canonical_freshness_policies` append-only chain pattern (same model for policy versioning)

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `4eac37ca6f490bed485cc2866d4e19db7740d15d` |
| Branch name | `agent/phase-g-slice-1-policy-management` |

## Objective

Make policies owner-editable. Today `canonical_policies` rows exist only from migrations (seeded directly in SQL). No application code can create, update, or evaluate policies at runtime. The compliance validation engine reads policies via `gmcom_current_policy` — this slice makes that pipeline writable.

## Required behavior

### Three operations

1. **`createPolicyVersion(policyKey, definition, supersedesPreviousVersionId?, context)`** — creates a new version of a policy. Follows the same append-only chain pattern as `canonical_freshness_policies` (Phase C Slice 5): each new version supersedes the previous, "current" is derived as the un-superseded chain head. Never mutates an existing row.

2. **`queryApplicablePolicies(scope, asOf?)`** — returns all policies whose applicability scope matches the requested scope, at the requested point in time. A policy's applicability is determined by its definition's `applicability` field (a Scope, per §5). Filters to the caller's bound environment.

3. **`evaluatePolicy(policyId, subject)`** — evaluates whether a specific policy applies to a specific subject (EntityRef: SKU, ProductConcept, etc.). Returns `{ applicable: boolean, constraints: PolicyConstraint[] }`. The evaluation is deterministic: scope matching, not an AI judgment.

### Policy value shape

```ts
export interface PolicyDefinition {
  policyKey: string;           // e.g. "shopify-copy-policy", "photo-alt-text-policy"
  applicability: Scope;       // e.g. { kind: "marketplace", marketplace: "shopify" }
                              //      or { kind: "global" }
  constraints: PolicyConstraint[];
  description: string;
  version: number;            // derived from chain position, not caller-supplied
}

export interface PolicyConstraint {
  field: string;              // e.g. "title", "description", "alt_text"
  rule: string;               // e.g. "min_length:20", "required", "unique"
  severity: "error" | "warning";
  detail: string;             // human-readable explanation
  evidence?: string;          // reference to the source/justification
}

export interface PolicyEvaluationResult {
  policyId: string;
  policyKey: string;
  applicable: boolean;
  matchedConstraints: PolicyConstraint[];
  evaluatedAt: string;
}
```

### Schema choice

The `canonical_policies` table already exists with `id`, `policy_key`, `policy_version`, `definition` (jsonb). It does NOT have a `supersedes_policy_id` column or a chain-position column.

**Add a migration** to add `supersedes_policy_id` (FK to `canonical_policies.id`, nullable) and a guard trigger that prevents chain-branching (same pattern as `canonical_freshness_policies`). This is a real schema limitation — the table exists but can't support append-only versioning without these columns.

Also add an `inserted_at` column (timestamptz, default now()) for deriving "current" from chain position — the existing table lacks a reliable sort column for version ordering within a `policy_key`.

### Current policy derivation

Add a `gmcom_current_policy(p_environment text, p_policy_key text)` function that returns the un-superseded head of the chain for a given policy key. The existing compliance validation engine calls `gmcom_current_policy` — ensure the new function signature matches or wraps the existing call pattern so the compliance layer continues to work.

### Trust boundary

Follow the established pattern:
- `lib/policy/repository.ts` — `PolicyRepositoryImpl` with private constructor + module-private Symbol token
- `createPolicyRepository(supabase)` — the one way to obtain an instance
- `resolveTrustedEnvironment()` bound at construction

Do NOT implement RBAC gating yet — that's Slice G5. For Slice G1, the service identity (`{ kind: "service" }`) can create policies. Any principal can create a policy; the RBAC slice will gate this later.

## Files to create

| File | Purpose |
|---|---|
| `lib/policy/types.ts` | PolicyDefinition, PolicyConstraint, PolicyEvaluationResult types |
| `lib/policy/repository.ts` | PolicyRepositoryImpl — createPolicyVersion, queryApplicablePolicies, evaluatePolicy |
| `lib/policy/repository.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `supabase/migrations/` | New migration: add `supersedes_policy_id`, `inserted_at` to `canonical_policies` + chain guard trigger + `gmcom_current_policy` function |
| `.github/workflows/ci.yml` | Add `phase-g-slice1-policy-management-live` job |

## Test coverage required

1. **Create policy version**: creates a row with correct `policy_key`, `definition`, `version` computed from chain position
2. **Version chain**: two successive `createPolicyVersion` calls for the same `policy_key` produce versions 1 and 2; version 1 is superseded, version 2 is current
3. **Chain-branching prevention**: inserting two rows both claiming to supersede the same predecessor is rejected by the guard trigger
4. **Current policy derivation**: `gmcom_current_policy` returns the un-superseded head
5. **queryApplicablePolicies**: returns policies matching a scope; excludes policies with non-matching scope
6. **evaluatePolicy**: returns `applicable: true` for a matching subject; `applicable: false` for a non-matching one
7. **Environment isolation**: policies in different environments are isolated
8. **Empty policy store**: queryApplicablePolicies on a store with no matching policies returns an empty array
9. **Append-only**: existing policy rows are never mutated by createPolicyVersion

### Live-Postgres CI job

A new `phase-g-slice1-policy-management-live` job exercising against real Postgres:

- Create policy, verify version = 1
- Create second version, verify version = 2, first is superseded
- Guard trigger blocks chain-branching
- `gmcom_current_policy` returns the un-superseded head
- Query policies by scope (match and non-match)
- Environment isolation (policies in `test` not visible from `production` scope)

## Explicit exclusions

- **Do not** implement RBAC gating (Slice G5). Any principal can create policies for now.
- **Do not** implement owner confirmation flow (Slice G2).
- **Do not** implement correction capture (Slice G3).
- **Do not** implement rule activation (Slice G4).
- **Do not** add UI, review surfaces, or policy editing interface.
- **Do not** modify the compliance validation engine's policy reading — this slice adds the write path; the read path already works.
- **Do not** change the `IntelligenceRepositoryV1` contract — implement behind the existing interface.
- **Do not** add a full authentication system.

## Verifications before opening PR

```bash
npx vitest run lib/policy/repository.test.ts
npx tsc --noEmit
npm run build
```

All must pass.

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- Migration file path and summary of columns/functions added
- Any implementation decisions

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. Do not start Slice G2.
