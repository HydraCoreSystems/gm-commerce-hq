# Phase G Slice 2 Handoff — Owner Confirmation Flow

## Task

- Issue: Phase G Slice 2 (owner confirmation flow) implementation
- Objective: Build the §14 owner confirmation flow. An owner confirms a Phase F `Recommendation`, the system records a genuine `OwnerDecision` in `canonical_owner_decisions`, and optionally promotes the recommendation's value to an owner-authorized claim that outranks every non-owner claim. This is the "confirmation flow (Phase G)" all seven Phase F recommendation services reference.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `agent/phase-g-slice-2-owner-confirmation`

## PR

- **PR #31**: https://github.com/HydraCoreSystems/gm-commerce/pull/31
- **Head SHA**: `60d922585d936e66f55e29c3e9c9ffd0c7c18664`
- **Base SHA**: `a24ad83386933e6e5fe786206c29f7d9bb74c135` (brief's verified base)
- CI: all checks green (38/38 across the push and PR runs), including the new `phase-g-slice2-owner-confirmation-live` job, `verify`, `schema-from-empty`, `schema-drift-deferred`, and every existing live job.

## Work Completed

- **Principal mechanism** (`lib/canonical/claims/caller-principal.ts`): `resolveTrustedCallerPrincipal(env)` now reads the trusted deployment-config variable `GM_COMMERCE_CALLER`. `owner` constructs the `authenticated_owner` principal the codebase always modeled but could never build (`actorId` from `GM_COMMERCE_CALLER_ACTOR`, role from `GM_COMMERCE_ROLE`); unset/anything else (the default) resolves to `{ kind: "service" }`. This is the single authority-decision point — no passwords/sessions/JWTs, no per-request identity (per the brief's "no full auth system"). A future real owner-session layer touches this one function.
- **`recordOwnerDecision` unblocked** (`lib/canonical/claims/repository.ts`): now proceeds for an owner principal instead of throwing `UNAUTHORIZED`, recording a genuine `canonical_owner_decisions` row (subject-pair + predicate validated, `actor` derived from the bound principal — never caller-supplied). A service-bound repository still fails closed with the same UNAUTHORIZED.
- **Confirmation service** (`lib/confirmation/`):
  - `types.ts` — `ConfirmationRequest`, `ConfirmationResult`.
  - `principal.ts` — `resolvePrincipal()` / `isOwner()` / `requireOwner()` (fail-closed on non-owner).
  - `confirmation-service.ts` — `ConfirmationServiceImpl` (private constructor + module-private symbol; env-bound via `resolveTrustedEnvironment()`). `confirmRecommendation(recommendationId)` reads the recommendation + its §18 `SelectionTrace`, records a `confirm_recommendation` OwnerDecision, and proposes an `owner_exception`-sourced claim with `ownerOverrideId` (the 1000 top precedence tier). Idempotent, cross-subject-guarded, environment-scoped.
  - `confirmation-service.test.ts` — 15 tests covering the brief's required 9 scenarios (service principal rejected; owner success; real recommendation; non-existent recommendation; owner-authorized claim precedence above verified; idempotency; cross-subject rejection; environment isolation; principal default to service) plus `resolvePrincipal`/`requireOwner`/predicate-mapping extras.
- **Read path** (`lib/recommendations/repository.ts`): added `getRecommendation(recommendationId)` (environment-scoped) alongside the existing `getSelectionTrace`.
- **Test updates**: the stale "throws UNAUTHORIZED unconditionally" claims test now asserts the service-bound fail-closed behavior; `caller-principal.test.ts` wording updated to reflect the Phase G owner variant.
- **CI job `phase-g-slice2-owner-confirmation-live`** in `.github/workflows/ci.yml` (after the G1 job, before `schema-drift-deferred`): fresh `postgres:15`, all committed migrations applied, then live checks — a genuine OwnerDecision recorded against a real Recommendation row + §18 trace, the owner-authorized claim linked via `owner_override_id` outranking a verified non-owner claim (mirrors `computePrecedenceRank`: 1000 > 350), the idempotency guard query, and the cross-subject guard.

## Principal resolution mechanism chosen

`GM_COMMERCE_CALLER` environment variable (trusted deployment config, same category as `GMCOM_ENVIRONMENT`): `owner` → `authenticated_owner` principal (default role `phil`; `GM_COMMERCE_ROLE=crystal` and `GM_COMMERCE_CALLER_ACTOR` refine); unset → `service` (the pipeline). Fail-closed on everything else. Never read from a request/parameter.

## Verification

- Tests: **1012 passed** (996 prior + 16 new: 15 confirmation + 1 caller-principal owner variant), full suite green
- Typecheck: `npx tsc --noEmit` passed
- Build: `npm run build` passed
- CI: green end-to-end, including the new live job
- Local live validation: the G2 DO-block was executed against a pristine `postgres:15` container (all 28 migrations + capability enable + full check suite) and passed; additionally confirmed the Phase B fail-closed guarantee still holds — with the capability disabled (production env), a raw genuine owner-decision insert is still rejected.

## Implementation Decisions (flagged for DECISIONS.md)

1. **No committed migration.** The brief's exclusion ("do not add a migration unless a real schema limitation is demonstrated") and the plan's invariant #6 were respected. Phase B Slice 2's load-bearing `canonical_owner_decisions_authority_gate` (which rejects any genuine OwnerDecision while `canonical_verification_pipeline_capabilities.enabled` is false) is opened **only in the G2 scratch database as explicit test setup**. The real authority boundary for owner decisions is the application-layer `GM_COMMERCE_CALLER=owner` principal; the gate stays closed in every committed-migration run (`schema-from-empty` still asserts raw genuine inserts are rejected), and production capability state is untouched. If G2 must ever persist genuine decisions in a real deployment, a future reviewed migration must enable the capability (or decouple the owner-decision trigger from the contradiction gate) — flagged here for a separate decision, not taken in this slice.
2. **Confirmation idempotency is application-layer.** The service's `findConfirmationDecision` (decision_type + `detail.recommendationId`) guard reuses an existing decision/claim rather than duplicating. No DB unique constraint enforces this; the live job mirrors the guard query and asserts a single decision per recommendation.
3. **Cross-subject and principal gating are application-layer** (service + repository), mirrored in the live job only to the extent the DB can verify (decision subject equals recommendation subject). Both are covered by unit tests.

## Confirmation

- Migration added: **no** (per the brief's exclusion; see decision #1).
- Explicitly NOT done (deferred per brief): §15 RBAC (G5), correction capture/scope inference (G3), rule activation (G4), UI/review surfaces/approval dashboard, full authentication, and any change to the `recordOwnerDecision` database schema or the `IntelligenceRepositoryV1` contract.

## Stop Condition

Per the brief: all acceptance criteria met, all tests pass, PR open against `main` with green CI, handoff written. Slice G3 not started.
