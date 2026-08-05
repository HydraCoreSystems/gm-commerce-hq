# Phase G Slice 4 — Rule Activation Engine

**Status: Ready for implementation. Awaiting Phil's authorization.**

## Repositories and source documents

Read before implementing:

- `HydraCoreSystems/gm-commerce-hq`: `phase-g-slice-plan.md`, `PRODUCT_RESET_2026-08-03.md` (§14, §18)
- `HydraCoreSystems/gm-commerce`: `lib/canonical/intelligence-repository-contract.ts` — `activateRule`, `revokeRule`, `queryActiveRules`
- `HydraCoreSystems/gm-commerce`: `lib/recommendations/price.ts` — example recommendation service that Phase G rules should influence
- `HydraCoreSystems/gm-commerce`: `supabase/migrations/20260803030000_phase_b_slice1_canonical_entities.sql` — reference for canonical table patterns
- `HydraCoreSystems/gm-commerce`: `supabase/migrations/20260804030000_phase_c_slice5_freshness_revalidation.sql` — reference for append-only chain pattern, guard triggers

## Base

| Field | Value |
|---|---|
| Repository | `HydraCoreSystems/gm-commerce` |
| Base branch | `main` |
| Verified base SHA | `b0d5a03205b872c1a73c756c66cf0cccde57ccdf` |
| Branch name | `agent/phase-g-slice-4-rule-activation` |

## Objective

An owner-confirmed decision becomes a standing learned rule. Rules have scope, precedence, and can be queried by Phase F recommendation services and the compliance validation engine. This is the activation flow — the bridge from "owner confirmed this recommendation" to "the system applies this rule automatically going forward."

## Required behavior

### Migration: `learned_rules` table

Create `supabase/migrations/` file with a timestamp after the last migration. The table:

```sql
create table canonical_learned_rules (
  id              text primary key,
  rule_key        text not null,           -- unique key for the rule type
  predicate       text not null,           -- the claim predicate this rule applies to
  scope           jsonb not null,          -- Scope (§5) — where this rule applies
  definition      jsonb not null,          -- the rule's value (what it asserts)
  activated_by    text not null references canonical_owner_decisions(id),
  correction_id   text references canonical_corrections(id),  -- nullable: the correction that produced this rule
  active          boolean not null default true,
  activated_at    timestamptz not null default now(),
  revoked_at      timestamptz,
  revoked_by      text references canonical_owner_decisions(id),
  revocation_reason text,
  supersedes_rule_id text references canonical_learned_rules(id),  -- nullable: when a rule is revised
  -- Full RecordContext columns
  environment     text not null,
  record_purpose  text not null,
  source_run      text not null,
  correlation_id  text,
  owner_approval_state text not null,
  test_exception_ref text,
  reviewer_actor  text,
  decision_timestamp timestamptz,
  owner_decision_ref text,
  eligibility_knowledge_promotion boolean not null default false,
  eligibility_pricing_performance boolean not null default false,
  eligibility_publication boolean not null default false,
  retention_status text not null default 'active',
  retention_reviewed_at timestamptz,
  retention_reviewed_by text,
  inserted_at     timestamptz not null default now()
);

-- Index for querying active rules by scope
create index canonical_learned_rules_active_scope_idx
  on canonical_learned_rules (predicate, active)
  where active = true;
```

Use `check (environment in ('development','test','staging','production'))` constraint. Use `check (owner_approval_state in ('genuine','test_exception','pending','not_required'))`. Add a guard trigger that enforces `active = true` rows cannot be updated to `active = false` — they must go through `gmcom_revoke_rule` (an append-only path).

### Three operations

1. **`activateRule(ownerDecisionId, ruleKey, predicate, scope, definition, context)`** — creates a `canonical_learned_rules` row linked to the authorizing `OwnerDecision`. Validates the decision exists and is genuine. Gated to owner principals only (using G2's principal mechanism). Returns the rule ID.

2. **`revokeRule(ruleId, ownerDecisionId, reason, context)`** — sets `active = false`, records `revoked_at`, `revoked_by`, `revocation_reason`. Gated to owner principals. Only the activating owner or a superseding owner decision can revoke.

3. **`queryActiveRules(scope)`** — returns all active rules whose scope intersects with the requested scope. Ordered by `activated_at desc`. Used by Phase F recommendation services and the compliance engine.

### How rules influence recommendations

A learned rule is a claim with `source: derived_rule` and `status: verified` that was created from an owner decision. When a Phase F recommendation service evaluates claims for a subject, active learned rules that match the subject's scope appear in the claim set with high precedence (owner-override tier). The recommendation service doesn't need to change — the claim precedence model already handles this:

```
Owner-confirmed rule → OwnerDecision → activateRule → canonical_learned_rules row
                                                              │
                                          (Phase F reads claims for subject)
                                                              │
                                          Rule-matching claims appear in queryApplicableClaims
                                          with owner_override_id set → computePrecedenceRank
                                          places them above verified/research claims
```

For Slice 4, integration means: when `queryActiveRules` returns a rule that matches a subject, the rule is surfaced as a claim the recommendation service sees. No Phase F code changes required — the claim model and precedence are already built.

### Trust boundary

- `RuleEngineImpl` with private constructor + module-private Symbol token
- `createRuleEngine(supabase)` — the one way to obtain an instance
- `resolveTrustedEnvironment()` bound at construction
- Principal check: only `{ kind: "owner" }` can activate or revoke rules

## Files to create

| File | Purpose |
|---|---|
| `supabase/migrations/<timestamp>_phase_g_slice4_learned_rules.sql` | Migration: `canonical_learned_rules` table + indexes + guard trigger |
| `lib/rules/types.ts` | LearnedRule, RuleActivationRequest, RuleRevocationRequest types |
| `lib/rules/rule-engine.ts` | RuleEngineImpl — activateRule, revokeRule, queryActiveRules |
| `lib/rules/rule-engine.test.ts` | Unit + service orchestration tests |

## Files to modify

| File | Purpose |
|---|---|
| `.github/workflows/ci.yml` | Add `phase-g-slice4-rule-activation-live` job |

## Test coverage required

1. **Activate a rule**: creates a `canonical_learned_rules` row with correct `rule_key`, `predicate`, `scope`, `definition`, linked to an `OwnerDecision`
2. **Rule becomes active**: the row has `active = true`, `activated_at` set
3. **Service principal cannot activate rules**: `UNAUTHORIZED`
4. **Activation with non-existent OwnerDecision**: throws descriptive error
5. **Revoke a rule**: sets `active = false`, records `revoked_at`, `revoked_by`, `revocation_reason`
6. **Service principal cannot revoke rules**: `UNAUTHORIZED`
7. **Revoke an already-revoked rule**: throws descriptive error or is idempotent
8. **Query active rules by scope**: returns only active rules matching the scope
9. **Query excludes inactive rules**: revoked rules not returned
10. **Deterministic rule ordering**: `activated_at desc`
11. **Environment isolation**: rules in different environments are isolated
12. **Activation with non-genuine OwnerDecision**: rejected (test_exception/pending decisions cannot activate rules)

### Live-Postgres CI job

A new `phase-g-slice4-rule-activation-live` job:

- Owner decision inserted → rule activated → `canonical_learned_rules` row with correct values
- Rule activation round-trips: `active = true`, `activated_at` set, FK to `OwnerDecision` correct
- Rule revoked → `active = false`, `revoked_at` set
- Query active rules returns only active rules
- Query excludes revoked rules
- Guard trigger prevents direct UPDATE of `active` column (must go through revoke path)

## Explicit exclusions

- **Do not** implement RBAC enforcement (Slice G5). Use G2's owner principal check.
- **Do not** implement the full rule → claim bridge that surfaces learned rules to Phase F recommendation services. The precedence model already handles this; building the bridge requires determining when and how rules become claims, which is a design question for a follow-up slice or phase. For G4, focus on the rule CRUD lifecycle.
- **Do not** add UI, rule management dashboard, or policy editing UI.
- **Do not** implement rule evaluation against subjects (that's what Phase F already does via claims).
- **Do not** implement the `updateRule` or revision chain for rules (a later concern).

## Verifications before opening PR

```bash
npx vitest run lib/rules/rule-engine.test.ts
npx tsc --noEmit
npm run build
```

All must pass. Also verify schema-from-empty works with the new migration:

```bash
# In CI, the schema-from-empty job applies all migrations from scratch
# locally you can verify:
psql -h localhost -U postgres -d postgres -c "drop schema public cascade; create schema public;" 
for f in $(ls supabase/migrations/*.sql | sort); do
  psql -h localhost -U postgres -d postgres -v ON_ERROR_STOP=1 -f "$f"
done
```

## Handoff contents

- PR number and link
- Head SHA, base SHA
- Test count
- CI job name
- Migration file path and summary
- Any implementation decisions

## Stop condition

Stop when all acceptance criteria are met, all tests pass, PR is open against `main`, and handoff is written. Do not start Slice G5.
