# Phase G Slice Plan — Owner-Editable Policies and Learned-Rule Activation

_Authoritative plan per `PRODUCT_RESET_2026-08-03.md` §23 Phase G. Last updated: 2026-08-05._

## Dependency

Phase G depends on:

- **Phase F** (merged): all 7 recommendation services producing §18 `SelectionTrace`-backed `Recommendation` records. Owner confirms a recommendation, and it becomes a standing rule — the Phase F recommendation surface is the primary input to Phase G's confirmation flow.
- **Phase E** (merged): compliance-check infrastructure, `canonical_compliance_checks`, policy versioning used by the validation engine.
- **Phase D** (merged): canonical claims and evidence model, `canonical_owner_decisions` table, `canonical_corrections` table, `canonical_policies` table.
- **Phase B** (merged): canonical entities, `IntelligenceRepositoryV1` contract, verification pipeline.

Each slice depends strictly on the previous slice's merge into `main`.

## Current infrastructure state

### Exists (from prior phases)

| Resource | Status | Notes |
|---|---|---|
| `canonical_policies` table | Exists | Created Phase B Slice 1. `policy_key`, `policy_version`, `definition` (jsonb). Used by compliance validation engine via `gmcom_current_policy`. |
| `canonical_owner_decisions` table | Exists | Created Phase B Slice 1. `decision_type`, `subject_type`, `subject_id`, `predicate`, `applicability_scope`. |
| `canonical_corrections` table | Exists | Created Phase B Slice 1. FK to `canonical_owner_decisions`. `scope`, `rationale`, `before_value`, `after_value`. |
| `canonical_claim_verifications` table | Exists | Phase B Slice 2. Immutable audit of verification decisions. |
| `IntelligenceRepositoryV1` contract | Exists | `lib/canonical/intelligence-repository-contract.ts`. Methods defined but many deferred/fail_closed. |
| `recordOwnerDecision` | Implemented, fail_closed | `lib/canonical/claims/repository.ts:474`. Throws `UNAUTHORIZED` — `callerPrincipal` can only be `{ kind: "service" }`. |
| `proposeClaim` (owner_override) | Implemented | Already validates owner overrides against `canonical_owner_decisions`. |
| Compliance policy reading | Implemented | `gmcom_current_policy` reads from `canonical_policies`; validation engine uses it. |
| Phase F recommendations | Implemented | All 7 services produce `Recommendation` records with §18 traces. |

### Deferred or fail_closed (to be built in Phase G)

| Method | State | Phase G slice |
|---|---|---|
| `createPolicyVersion` | `deferred` | G1 |
| `queryApplicablePolicies` | `deferred` | G1 |
| `evaluatePolicy` | `deferred` | G1 |
| `recordOwnerDecision` | `fail_closed` (no auth) | G2 |
| `recordCorrection` | `deferred` | G3 |
| `proposeCorrectionScope` | `deferred` | G3 |
| `activateRule` | `fail_closed` (no auth) | G4 |
| `revokeRule` | `fail_closed` (no auth) | G4 |
| `queryActiveRules` | `deferred` | G4 |
| Learned rules table | Does not exist | G4 |
| RBAC enforcement (§15) | Contract only | G5 |

## Slice inventory

| Slice | Scope | Status | Base SHA |
|---|---|---|---|
| G1 | Policy management — create, query, evaluate | #30 | Merged | `4eac37` |
| G2 | Owner confirmation flow | #31 | Merged | `a24ad83` |
| G3 | Correction capture and scope inference | Not started | After G2 |
| G4 | Rule activation engine — learned rules, activate/revoke/query | Not started | After G3 |
| G5 | RBAC enforcement — §15 role matrix at Repository command layer | Not started | After G4 |

## Per-slice specification

### G1 — Policy management

**Objective**: Make policies owner-editable. Today `canonical_policies` rows exist only from migrations; no application code can create or modify them.

**Deliverables**:
- Implementation of `createPolicyVersion` — creates a new version of a policy (append-only chain, same pattern as freshness policies)
- Implementation of `queryApplicablePolicies(scope, asOf)` — returns policies that apply to a given scope
- Implementation of `evaluatePolicy(policyId, subject)` — evaluates a policy against a subject
- Live-Postgres CI job

### G2 — Owner confirmation flow (§14)

**Objective**: An owner sees a Phase F recommendation, confirms it, and the system records an `OwnerDecision`. This is the §14 confirmation flow that all Phase F services reference as future work.

**Deliverables**:
- Unblock `recordOwnerDecision` — introduce an owner principal mechanism (minimal, local-to-`localhost` identity with no full auth system)
- Confirmation service — given a `RecommendationId` + owner identity, creates an `OwnerDecision` linked to the recommendation
- Owner-override claim creation — an owner decision can create or supersede a claim with owner authority
- Live-Postgres CI job

### G3 — Correction capture and scope inference

**Objective**: When Phil corrects something, record the correction and derive what scope it could apply to (§14's eligible-scope derivation).

**Deliverables**:
- `recordCorrection` implementation — creates a `canonical_corrections` row linked to an `OwnerDecision`
- `proposeCorrectionScope` implementation — derives eligible scopes from entity hierarchy + predicate semantics
- Scope-derivation rules per predicate type (price → SKU/concept only; care instructions → SKU/concept/genus/all plants; etc.)
- Live-Postgres CI job

### G4 — Rule activation engine

**Objective**: An owner-confirmed decision activates as a standing learned rule. Rules have scope, precedence, and can be queried by Phase F recommendation services and the compliance validation engine.

**Deliverables**:
- `learned_rules` table migration — `rule_key`, `predicate`, `scope`, `definition`, `activated_by` (FK to `canonical_owner_decisions`), `active`, `activated_at`
- `activateRule(ruleProposalId, confirmedBy)` implementation
- `revokeRule(ruleId, reason)` implementation
- `queryActiveRules(scope)` implementation — returns all active rules applicable to a scope
- Integration: Phase F recommendation services query active rules during evaluation
- Live-Postgres CI job

### G5 — RBAC enforcement (§15)

**Objective**: Enforce the §15 role matrix at the Repository command layer. The service identity (pipeline) cannot override claims or change policies. Only owners can activate rules.

**Deliverables**:
- Principal resolution — determine which §15 role the current caller holds (minimal: owner vs. service, for localhost)
- Gate `createPolicyVersion` behind owner role
- Gate `activateRule` behind owner role
- Gate `recordOwnerDecision` behind owner role
- Audit-event attribution with resolved role
- Live-Postgres CI job

## Shared invariants across all slices

1. No full authentication system — Phase G uses a minimal localhost identity mechanism. The §15 role matrix distinguishes `{ kind: "service" }` (the pipeline itself) from `{ kind: "owner" }` (Phil/Crystal). Full per-person accounts are out of scope.
2. All writes are append-only — policies, decisions, corrections, and rules are never mutated or deleted after insert.
3. Every owner action is audit-logged with actor + role + timestamp.
4. Environment isolation — all tables carry `environment`; Repository methods are environment-scoped.
5. Security — no method relaxes the existing trust-boundary discipline (module-token constructors, bound environment).
6. No migration unless a real schema limitation is demonstrated and approved separately.

## Authorization

Phil authorizes each slice before implementation begins. This plan records the slice ordering and dependencies. Individual slice briefs provide exact scope, acceptance criteria, and exclusions.
