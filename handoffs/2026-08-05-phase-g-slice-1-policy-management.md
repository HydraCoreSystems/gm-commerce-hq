# Phase G Slice 1 Handoff — Policy Management

## Task

- Issue: Phase G Slice 1 (policy management) implementation
- Objective: Make `canonical_policies` owner-writable via append-only versioning (`createPolicyVersion`, `queryApplicablePolicies`, `evaluatePolicy`), following the same chain pattern as `canonical_freshness_policies` (Phase C Slice 5). The compliance engine already reads policies via `gmcom_current_policy`; this slice adds the write path and chain semantics without touching that read path.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `agent/phase-g-slice-1-policy-management`

## PR

- **PR #30**: https://github.com/HydraCoreSystems/gm-commerce/pull/30
- **Head SHA**: `ff7159bbd9017ab8d2dce643c2872897d1e8351c`
- **Base SHA**: `4eac37ca6f490bed485cc2866d4e19db7740d15d` (brief's verified base)
- Mergeability: MERGEABLE, mergeStateStatus CLEAN

## Work Completed

- **Migration `supabase/migrations/20260805000000_phase_g_slice1_policy_management.sql`**:
  - Adds `inserted_at timestamptz not null default now()` (append-only chain-position clock) and `supersedes_policy_id text` (nullable).
  - Composite FK `canonical_policies_supersedes_fk` on `(supersedes_policy_id, environment)` → `canonical_policies (id, environment)` — a successor can only supersede a predecessor in the SAME environment (cross-environment supersession is a trust violation).
  - Unique partial index `canonical_policies_supersedes_unique` on `supersedes_policy_id where not null` (the unmaskable anti-branching backstop), plus `canonical_policies_chain_lookup_idx` on `(environment, policy_key, policy_version desc, inserted_at desc)`.
  - Guard trigger `gmcom_guard_policy_chain` (before insert): predecessor must exist; must be the same `policy_key`; new version must be strictly higher; the predecessor must not already have a successor (no branching); and every `policy_version > 1` must carry `supersedes_policy_id` referencing the current head (freshness-pattern invariant). Revoked from `public/anon/authenticated/service_role`.
  - Rewrites `gmcom_current_policy(text, text)` to derive the un-superseded chain head (row nothing else's `supersedes_policy_id` references yet), same signature and `(id, policy_version, definition)` return shape — the compliance read path is unchanged. Execute granted to `service_role` only.
- **`lib/policy/types.ts`**: `PolicyDefinitionInput`, `PolicyDefinition`, `PolicyRecord`, `PolicyVersionCreated`, `PolicyEvaluationResult`, `PolicySubject`, `PolicyConstraint`.
- **`lib/policy/repository.ts`**: `PolicyRepositoryImpl` with private constructor + module-private `CONSTRUCTION_TOKEN`; `createPolicyRepository(supabase)` is the only way to obtain an instance; `resolveTrustedEnvironment()` bound at construction. `createPolicyVersion` derives `version = head.version + 1` (never caller-supplied); `queryApplicablePolicies(scope, asOf?)` matches the Scope union `{entity | genus | category | marketplace | global}` against `definition.applicability`; `evaluatePolicy(policyId, subject)` is a deterministic scope match; `getCurrentPolicy` returns the un-superseded head.
- **`lib/policy/repository.test.ts`**: 9 scenarios — create v1, successive v1→v2 supersession, chain-branch rejection, head derivation, scope match and non-match, `evaluatePolicy` match/non-match, environment isolation, empty store, append-only (existing rows never mutated).
- **CI job `phase-g-slice1-policy-management-live`** in `.github/workflows/ci.yml` (after the F7 job, before `schema-drift-deferred`): fresh `postgres:15`, all committed migrations applied, then a DO-block covering create v1, supersede v2, guard chain-branch rejection, `gmcom_current_policy` head derivation, scope match/non-match, and environment isolation (test vs production).

## Verification

- Tests: **9 passed** (`lib/policy/repository.test.ts`), **996 passed** (full suite)
- Typecheck: `npx tsc --noEmit` passed
- Build: `npm run build` passed (only the pre-existing unrelated `libheif-js` critical-dependency warning)
- CI: all checks green on both the push and PR runs, including `verify`, `schema-from-empty`, `schema-drift-deferred`, every existing live job, and the new `phase-g-slice1-policy-management-live`

## Implementation Decisions (flagged for DECISIONS.md)

1. **Chain guard enforces the strict freshness invariant: every `policy_version > 1` must carry `supersedes_policy_id` referencing the current head.** Four pre-existing live-Postgres jobs (E2/E3/E4/F2) superseded `shopify-copy-policy` v1→v2 without the link and began failing on the new guard. Their fixtures were updated (ci.yml only — no read-path changes) to chain the v2 to the seeded v1 head; E2's duplicate-version check now observes the guard rejection (a lapsed version can never recur).
2. **`queryApplicablePolicies` matches against `definition.applicability` scopes.** E2's seeds carry a top-level `marketplace` key instead, so the live scope assertions use `count(distinct policy_key)` to avoid double-counting seed and G1 rows.
3. **Repository-level predecessor validation in addition to the DB guard**: `createPolicyVersion` refuses a caller-supplied predecessor that is not the current head, mirroring the trigger for a fast, type-safe failure before SQL is reached.

## Confirmation

- Migration added: yes — `supabase/migrations/20260805000000_phase_g_slice1_policy_management.sql`.
- Explicitly NOT done (deferred per brief): RBAC gating (G5), owner confirmation (G2), correction capture (G3), rule activation (G4), any UI, any change to the compliance read path, and any change to the `IntelligenceRepositoryV1` contract.

## Stop Condition

Per the brief: all acceptance criteria met, all tests pass, PR open against `main` with green CI, handoff written. Slice G2 not started.
