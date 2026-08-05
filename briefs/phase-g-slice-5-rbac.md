# Phase G Slice 5 — RBAC Enforcement

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-g-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§15)
- `HydraCoreSystems/gm-commerce`: `lib/canonical/intelligence-repository-contract.ts` — `REPOSITORY_V1_OPERATION_POLICIES` authorization map
- `HydraCoreSystems/gm-commerce`: `lib/confirmation/principal.ts` — existing `resolvePrincipal`, `requireOwner` (G2)
- `HydraCoreSystems/gm-commerce`: `lib/policy/repository.ts` — G1 policy repository (currently no RBAC gate)
- `HydraCoreSystems/gm-commerce`: `lib/rules/rule-engine.ts` — G4 rule engine (currently no RBAC gate)
- `HydraCoreSystems/gm-commerce`: `lib/correction/correction-service.ts` — G3 correction service (currently uses G2 principal check)
- `HydraCoreSystems/gm-commerce`: `lib/confirmation/confirmation-service.ts` — G2 confirmation service (currently uses G2 principal check)

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `1e631b7020e27d0c83cb6795620774c8e1408738` |
| Branch name | `agent/phase-g-slice-5-rbac` |

## Objective

Wire the §15 role matrix into every privileged operation. Today G2 introduced `requireOwner` — a minimal check that works but is duplicated across each service. G5 makes authorization centralized, auditable, and consistent: every owner-gated operation goes through the same role check, and the permission matrix is in one place.

## Required behavior

### Current state

Each G2-G4 service has its own inline `requireOwner` check:

```ts
// lib/confirmation/principal.ts
export function requireOwner(): CallerPrincipal {
  const principal = resolvePrincipal();
  if (principal.kind !== "owner") {
    throw new Error("UNAUTHORIZED: only the owner can perform this operation");
  }
  return principal;
}
```

Each service imports and calls this. It works — but there's no distinction between owner, co-owner, designated staff, or service identity. And the authorization check is scattered across four files.

### What G5 builds

1. **Formal role types** — expand principal beyond `service` vs `owner`:

```ts
export type PrincipalRole = "owner" | "co_owner" | "staff" | "service";

export interface CallerPrincipal {
  role: PrincipalRole;
  actor: string;             // "Phil", "Crystal", "system"
  scopedPermissions?: string[];  // e.g. ["photography"] for task-scoped staff
}

// Default: { role: "service", actor: "system" }
export function resolvePrincipal(): CallerPrincipal;
```

2. **Centralized authorization module** — `lib/auth/authorization.ts`:

```ts
// Checks if the current principal holds a required role.
// Throws UNAUTHORIZED with a descriptive message on failure.
export function requireRole(requiredRole: PrincipalRole): CallerPrincipal;

// Checks if the current principal has a specific permission.
// Used for task-scoped staff (e.g., photography-only access).
export function requirePermission(permission: string): CallerPrincipal;

// Checks if the principal's role is sufficient for the required role.
// owner >= co_owner >= staff >= service
export function roleIsSufficient(actual: PrincipalRole, required: PrincipalRole): boolean;
```

3. **Wire into existing services** — replace all inline `requireOwner()` calls with `requireRole("owner")`. Replace any service-identity checks with `requireRole("service")` or the default pass-through. The behavior at each gate is unchanged; the mechanism is centralized.

4. **Audit attribution** — every operation that checks a role attaches `actor` + `role` to the audit context. For `recordOwnerDecision`, `recordCorrection`, `activateRule`, and `revokeRule`, the `reviewer_actor` field in RecordContext is set from the resolved principal.

### Files to touch (replace inline checks)

| File | Current check | Replace with |
|---|---|---|
| `lib/confirmation/principal.ts` | `requireOwner()` | Expand to full role types, rename to `resolvePrincipal` |
| `lib/confirmation/confirmation-service.ts` | `requireOwner()` | `requireRole("owner")` |
| `lib/correction/correction-service.ts` | `requireOwner()` | `requireRole("owner")` |
| `lib/rules/rule-engine.ts` | `requireOwner()` | `requireRole("owner")` |
| `lib/policy/repository.ts` | (no gate yet) | Add `requireRole("owner")` at `createPolicyVersion` |

### Trust boundary

- `lib/auth/authorization.ts` is a pure module — it reads `GM_COMMERCE_CALLER` from env, resolves the principal, and exports check functions. No database access, no class, no constructor.
- The authorization check is a function call at the top of each gated method — any bypass is a code defect, not a runtime vulnerability.

## Files to create

| File | Purpose |
|---|---|
| `lib/auth/types.ts` | PrincipalRole, CallerPrincipal types |
| `lib/auth/authorization.ts` | resolvePrincipal, requireRole, requirePermission, roleIsSufficient |
| `lib/auth/authorization.test.ts` | Unit tests for authorization module |

## Files to modify

| File | Purpose |
|---|---|
| `lib/confirmation/principal.ts` | Replace with re-export from `lib/auth/authorization.ts` |
| `lib/confirmation/confirmation-service.ts` | Replace `requireOwner` with `requireRole("owner")` |
| `lib/correction/correction-service.ts` | Replace `requireOwner` with `requireRole("owner")` |
| `lib/rules/rule-engine.ts` | Replace `requireOwner` with `requireRole("owner")` |
| `lib/policy/repository.ts` | Add `requireRole("owner")` at `createPolicyVersion` |
| `.github/workflows/ci.yml` | Add `phase-g-slice5-rbac-live` job |

## Test coverage required

1. **Default principal is service**: `resolvePrincipal()` with no env returns `{ role: "service", actor: "system" }`
2. **Owner principal resolved**: `GM_COMMERCE_CALLER=owner` → `{ role: "owner", actor: "Phil" }`
3. **requireRole passes for sufficient role**: `requireRole("owner")` with owner principal → returns principal
4. **requireRole fails for insufficient role**: `requireRole("owner")` with service principal → `UNAUTHORIZED`
5. **roleIsSufficient hierarchy**: owner >= co_owner >= staff >= service
6. **requirePermission for scoped staff**: staff with `scopedPermissions: ["photography"]` passes `requirePermission("photography")`, fails `requirePermission("policy_edit")`
7. **Co-owner can do owner-gated operations**: `requireRole("co_owner")` with co_owner principal passes calls gated at `co_owner` level
8. **Staff cannot do owner-gated operations**: staff principal fails `requireRole("owner")`
9. **Invalid GM_COMMERCE_CALLER value**: defaults to service with a warning
10. **Policy creation gated**: service principal fails `createPolicyVersion`

### Live-Postgres CI job

A new `phase-g-slice5-rbac-live` job:

- Verify `recordOwnerDecision` with service principal fails (previous slices verified this; G5 confirms it still holds after refactor)
- Verify `createPolicyVersion` gated behind owner role
- Verify `activateRule` gated behind owner role
- Verify `recordCorrection` gated behind owner role
- All existing live-Postgres jobs continue to pass (the authority gate is in the CI scratch DB, not the production capability set)

The G5 live job exercises the authorization mechanism at the DB boundary — it confirms that the TypeScript-layer gates (requireRole) align with the DB-layer gates (capability triggers).

## Explicit exclusions

- **Do not** add per-person accounts, passwords, sessions, or cookies. Role is determined from `GM_COMMERCE_CALLER` env var only.
- **Do not** add UI, login pages, or authentication flows.
- **Do not** change the database schema. The capability gates already exist; G5 aligns the TypeScript layer with them.
- **Do not** change the behavior of any G2-G4 operation beyond replacing the authorization check location. The same operations succeed/fail for the same principals.
- **Do not** add a migration.

## Verifications before opening PR

```bash
npx vitest run lib/auth/authorization.test.ts
npx vitest run lib/confirmation/confirmation-service.test.ts
npx vitest run lib/correction/correction-service.test.ts
npx vitest run lib/rules/rule-engine.test.ts
npx vitest run lib/policy/repository.test.ts
npx vitest run  # full suite
npx tsc --noEmit
npm run build
```

All must pass.

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- List of files modified with `requireRole` replacement
- Any implementation decisions

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. This is the final Phase G slice.
