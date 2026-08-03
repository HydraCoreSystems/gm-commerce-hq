# Handoff — Phase B slice 1: canonical entity foundation

## Task

- Objective: Phase B's full scope (PRODUCT_RESET_2026-08-03.md §23) —
  canonical entities (§5, 18 types), claim/evidence model (§6), Repository
  foundation (§9), legacy migration (§6), and `LegacyCorrectionEvent`
  migration (§14.1) — is too large for one PR. Assigned directly by Phil
  (relayed via Claude in chat): propose a finite slice breakdown, then
  implement only the first slice.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `phase-b-slice-1-canonical-entities`, **not merged**. Commits
  `aa09c2d` (canonical entity migration + `lib/canonical/` +
  `supabase/canonical-entities-migration.test.ts`), `26f429a` (CI
  extension), `2d44509` (test-fake bug fix — see "What went wrong" below).
- PR: https://github.com/HydraCoreSystems/gm-commerce/pull/5

## Verified state before starting (re-verified, not trusted from prior docs)

- `gm-commerce` `main` at `46d364d` (Phase B0 hardening merge). Phase A and
  Phase B0 both genuinely complete — confirmed against `git log`, not
  `STATUS.md`'s text, which had a stale "Phase B0 not started" line
  corrected as part of this handoff (see STATUS.md's 2026-08-03 correction
  notice).
- 8 migrations exist in `supabase/migrations/`, through
  `20260803020000_phase_b0_function_search_path.sql`.
- Local test suite before this work: 105 passing (confirmed via `npm test`
  after `npm ci`, since `node_modules` was not present in the checkout).

## Slice breakdown proposed (Step 2 of the assignment)

- **Slice 1 (this PR):** Canonical entity foundation — `RecordContext`
  envelope (§25) as real Postgres + TypeScript infrastructure, 18 entity
  tables (§5) with ULID-shaped IDs, RLS, and the minimal
  create/get/list slice of `IntelligenceRepositoryV1` (§9). No Claim/
  Evidence semantics, no CommercePackage assembly, no
  `requestCrossEnvironmentAccess`.
- **Slice 2:** Evidence & Claim model (§6) — `EvidenceSource`/
  `EvidenceRevision`/`EvidenceAnchor`/`Claim` business logic (precedence,
  contradiction, corroboration per §10.1), the evidence/claim Repository
  commands, `queryApplicableClaims`. Depends on slice 1's tables.
- **Slice 3:** Repository contract completion (policies, recommendations/
  packages, decisions/corrections, `promoteEvidence`, audit trail,
  `requestCrossEnvironmentAccess`) + the `content_provenance`/
  `commerce_details` dual-write bridge and backfill validation (§6).
  Depends on slices 1-2.
- **Slice 4:** `LegacyCorrectionEvent` → canonical `Correction` migration
  job (§14.1). Depends on slices 1-2 (needs entities + claims to resolve
  a legacy field edit into a canonical reference).

## Work completed (slice 1 only)

**`supabase/migrations/20260803030000_phase_b_slice1_canonical_entities.sql`**
(additive; no existing table/column touched): `gmcom_ulid()` (Postgres-side
ULID generator, search_path pinned in the same migration rather than a
follow-up hardening PR this time), and 18 tables —
`canonical_product_concepts`, `canonical_skus`, `canonical_variants`,
`canonical_inventory_items`, `canonical_source_records`,
`canonical_photo_assets`, `canonical_claims`, `canonical_evidence_sources`,
`canonical_evidence_revisions`, `canonical_evidence_anchors`,
`canonical_policies`, `canonical_recommendations`,
`canonical_commerce_packages`, `canonical_marketplace_listings`,
`canonical_marketplace_publication_attempts`, `canonical_editorial_assets`,
`canonical_owner_decisions`, `canonical_corrections` — each with real
foreign keys matching §5's stated relationships (not a flat relabeling),
a full `RecordContext` envelope (fail-closed defaults, the same
approval-state-shape / eligibility-requires-genuine CHECK constraints
Phase B0 established), RLS enabled, `anon` fully revoked, `service_role`
full CRUD, and `authenticated` given a real environment-scoped `SELECT`
policy gated on session GUC `app.gmcom_caller_environment` (fail-closed:
unset GUC → zero rows).

**`lib/canonical/`** (new): `ulid.ts` (app-layer ULID generator, primary
ID-issuing path), `record-context.ts` (`applyRecordContextDefaults`, fails
closed on missing eligibility/approval fields, throws on a malformed
approval-state shape or an eligibility flag without genuine approval),
`entities.ts` (the 18-type `EntityType` union + table mapping),
`repository.ts` (`createEntity`/`getEntity`/`listEntities` — structural,
not just RLS-based, environment isolation: every read requires an
explicit `callerEnvironment`, and `listEntities` throws rather than
silently accepting `environment` as a filter key).

**`.github/workflows/ci.yml`**: extended the existing `schema-from-empty`
job (real Postgres 15 service container) with three new steps — confirm
all 18 tables exist with RLS + the expected grant shape, confirm
`gmcom_ulid()` produces 1,000 well-formed/unique IDs live, and confirm the
RLS policy is actually fail-closed with no GUC and correctly scoped once
one is set (insert a production + a test fixture row, `SET ROLE
authenticated`, assert 0 rows unscoped and exactly 1 scoped).

**`supabase/schema.sql`**: migration content appended, per this repo's
existing clean-install-reference convention.

## What went wrong, and how it was caught

The first push (`aa09c2d`) included `lib/canonical/repository.test.ts`
with a bug in its fake Supabase query-builder: one builder instance was
reused per table across independent queries, so `.eq()` filters from an
earlier call (e.g. a `production`-scoped `listEntities`) leaked into a
later, unrelated call (e.g. a subsequent `test`-scoped one), making the
second call return zero rows instead of the test fixture. This was caught
by CI's `verify` job on the PR (run `91762740166`) — 7 tests failed — not
by local testing, because the fix I made locally to the test file was
mistakenly never `git add`ed/committed before the first push (a session
interruption mid-task; the working tree had the fix, the checkpoint commit
didn't). Fixed in `2d44509`: each `.from(table)` call now returns a fresh
`FakeQueryBuilder` (matching supabase-js's real behavior) backed by
persistent per-table row storage. Re-ran locally (191/191 passing) before
re-pushing; CI's next run (`91763171851`/`91763164389`) passed clean.
Recorded here rather than silently amended, since a reviewer will see the
failed run in the PR's check history and should know why.

## Verification (exact)

- `npm run typecheck` — clean.
- `npm test` — 191 passed, 0 failed (105 pre-existing + 86 new).
- `npm run build` — succeeds (`next build`; pre-existing unrelated
  `libheif-js` critical-dependency warning, present on `main`).
- Schema-from-empty: **not run locally** — no Docker daemon or local
  Postgres was reachable in this session's environment (disclosed, not
  silently skipped, same as `supabase/ledger-bootstrap.test.ts`'s own
  precedent from Phase A). Verified instead by CI on the PR (real
  Postgres 15 service container) — run `30836601131` / job
  `91763171826`, all steps green, including the three new live checks
  this PR adds (see above).

## Honest limitation (stated in the PR, restated here)

`service_role` (this app's only real caller — `lib/supabase.ts` uses only
the service-role key, no browser-side Supabase Auth exists yet) has
Postgres `BYPASSRLS`, so RLS does not actually gate today's real caller.
The genuine environment-isolation enforcement for today's app is the
TypeScript repository layer (`lib/canonical/repository.ts`'s mandatory
`callerEnvironment` parameter, no widening path). The `authenticated`-role
RLS policy is real, live-verified, and correct defense-in-depth for a
future non-service-role caller, but is not load-bearing today. Stated
directly in the migration's comments and the repository module comment,
not just here.

## Deferred to later slices, and why

- `requestCrossEnvironmentAccess` (§9/§25) — slice 3, per the agreed
  breakdown; out of scope per the assignment brief.
- Claim/Evidence precedence, contradiction, corroboration thresholds
  (§6/§10.1) — slice 2; slice 1's `canonical_claims`/`canonical_evidence_*`
  tables are entity substrate only, no business logic.
- `CommercePackage.content`/`offers[]`/`fieldLineage[]` assembly (§7) —
  slice 3; slice 1's `canonical_commerce_packages` is substrate only.

## Next step

PR #5 needs review and, if approved, a merge by someone other than Claude
(never merges its own PR per this project's standing rule). Slice 2
(Evidence & Claim model) is the natural next assignment once slice 1 lands.
