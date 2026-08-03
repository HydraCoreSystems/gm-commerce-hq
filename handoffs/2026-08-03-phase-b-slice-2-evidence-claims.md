# Handoff — Phase B slice 2: Evidence & Claim intelligence

## Task

- Objective: implement Phase B slice 2 per the breakdown proposed in
  `handoffs/2026-08-03-phase-b-slice-1-canonical-entities.md` — the
  Evidence & Claim model (`PRODUCT_RESET_2026-08-03.md` §6): precedence,
  contradiction, corroboration (§10.1), the evidence/claim Repository
  commands (§9), `queryApplicableClaims`, and untrusted-research-ingestion
  protections (§10).
- Authorized directly by Phil, relayed via chat, after confirming Phase B
  slice 1 (`gm-commerce` PR #5) was merged and slice 2 had not started.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `phase-b-slice-2-evidence-claims`, **not merged**. Commits
  `4d42644` (schema + business-logic checkpoint), `43f3475` (tests, CI
  extension, `ownerOverrideId` support, final verification).
- PR: https://github.com/HydraCoreSystems/gm-commerce/pull/6

## Verified state before starting (re-verified, not trusted from prior docs)

- `gm-commerce` `main` HEAD confirmed exactly `b1af27aa737bbf6f792ffa22d01190efe8ce1859`
  via a fresh clone (not `git log` on an existing checkout) — Phase B slice
  1 genuinely merged.
- No `phase-b-slice-2*` branch existed remotely; slice 2 genuinely not
  started.
- `HY-LOB01-C04` confirmed test-artifact-only per `PRODUCT_RESET_2026-08-03.md`
  §1.2/§22.1 — not touched by this work.
- Working trees on both `gm-commerce-hq` and `gm-commerce` were clean;
  nothing uncommitted to preserve.

## Work completed

**`supabase/migrations/20260803040000_phase_b_slice2_evidence_claims.sql`**
(additive; no table from any prior migration touched — verified by a
structural test and live-checked against real Postgres):

- `canonical_claims` gains `sources` (jsonb `SourceRef[]`, non-empty array
  enforced by CHECK), `evidence_anchor_ids` (text[]), `confidence` (0–1
  range CHECK), `established_at`/`last_verified_at`, `freshness_ttl_seconds`/
  `freshness_policy_id`, `policy_snapshot_id`, `applicability_scope` (jsonb
  `Scope`), `contradicts` (text[]), `superseded_by`/`owner_override_id`
  (composite FKs), `status` (full lifecycle CHECK), and a `subject_type`
  CHECK against exactly the 18 canonical entity types.
- `canonical_evidence_sources` gains an 8-type `source_type` CHECK (now also
  `NOT NULL`), `source_family_id` (lineage grouping), `acl`, and
  `current_revision_id` (composite FK to `canonical_evidence_revisions`,
  added after that table exists in the same migration to resolve the
  circular reference).
- `canonical_evidence_revisions` gains `raw_content` (kept separate from the
  pre-existing `extracted_text` — §10 pipeline step 1), OneDrive-relocation-
  safe identity (`one_drive_path`/`one_drive_item_id`), `extraction_version`,
  `promotion_state` (§18 lifecycle), `supersedes` (composite self-FK), and
  `suspicious_content_detected`/`suspicious_content_detail` (§10 step 6).
- New `canonical_contradictions` (full `RecordContext` envelope, a
  resolved-shape CHECK ensuring `resolution`/`resolved_at`/`resolved_by` are
  only ever set together with `status = 'resolved'`) and
  `canonical_contradiction_claims` (composite-FK join table to both
  `canonical_claims` and `canonical_contradictions`).
- Two columns (`canonical_claims.sources`, `.evidence_anchor_ids`)
  deliberately carry **no** database FK — Postgres cannot FK into jsonb
  array elements or text[] values without a join table + trigger per
  element; the application layer (`proposeClaim`) resolves and validates
  every referenced id against the bound-environment repository before
  insert. Documented as a deliberate tradeoff in the migration's header
  comment, mirroring slice 1's own precedent (`applicability` jsonb with no
  FK).

**`lib/canonical/claims/`** (new):
- `types.ts` — `Claim`/`ClaimDraft`/`SourceRef`/`EvidenceSource`/
  `EvidenceRevision`/`Contradiction`/`Scope`/etc., plus the six source
  categories and eight evidence types as typed unions.
- `precedence.ts` — `computePrecedenceRank`/`orderByPrecedence`. Precedence
  is **derived at query time, never persisted** (per §6's explicit
  instruction) — there is no `precedence_rank` column anywhere in the
  schema.
- `source-families.ts` — independent-source-family counting, keyed by an
  explicit `sourceFamilyId` label (assigned at ingestion) rather than
  automated content-similarity detection, which is deferred to whichever
  later phase builds a real research-ingestion connector with actual
  documents to compare.
- `verification.ts` — `classifyClaim` (the §10.1 claim-class classifier —
  commerce-consequential predicates are checked **before** the generic
  deterministic/derived source-category rule, since §10.1 carves out its
  own three-way split for that predicate class regardless of source
  category), `evaluateVerificationEligibility` (the full corroboration/
  freshness/contradiction gate), `canActorVerify` (the §15 RBAC narrowing
  for the verify-claim grant: Phil and Crystal may verify, the service
  identity may only trigger an already-`auto_eligible` transition, staff
  never verifies anything).
- `ingestion-security.ts` — `sanitizeRawContent`, `detectSuspiciousContent`,
  `buildExtractionRequest`/`assertChannelsNotBled` (structural channel
  separation), `toValidatorInput` (bounded claim-candidate shape). No live
  AI research-extraction call exists yet in this slice — this is the
  reusable security plumbing a later phase's connector must go through.
- `repository.ts` — `ClaimEvidenceRepositoryImpl`, a **sibling** to slice
  1's `CanonicalEntityRepositoryImpl`, not a modification of it (that file
  has zero edits in this PR). Independently re-implements the same
  construction discipline: private constructor + module-private symbol
  token, one environment resolved from trusted config and bound for the
  instance's lifetime, no method anywhere accepting a different
  environment. Implements `ingestEvidence`/`getEvidenceRevision`/
  `searchEvidence`/`proposeClaim`/`verifyClaim`/`supersedeClaim`/
  `recordContradiction`/`resolveContradiction`/`queryApplicableClaims`.

**`.github/workflows/ci.yml`**: extended `schema-from-empty` with 3 new
live steps (run in a deliberate order — the RLS-scoping check runs first,
before the other two steps insert additional production-environment
`canonical_contradictions` fixture rows that would otherwise break its
exact-count assertion): CHECK-constraint rejection (empty `sources[]`,
out-of-range confidence, invalid evidence `source_type`, contradiction
resolved-shape), RLS fail-closed/scoped on the two new tables, and
composite-FK cross-environment rejection on the new self-referential and
cross-table FKs.

## Deferred to later slices, and why

- `requestCrossEnvironmentAccess` (§9/§25) — slice 3, per the agreed
  breakdown and Phil's explicit slice 2 scope list.
- `CommercePackage.content`/`offers[]`/`fieldLineage[]` assembly (§7) —
  slice 3.
- `Policy` versioning/`createPolicyVersion` (§9) — slice 3. Claims may cite
  an existing policy snapshot by opaque `policy_snapshot_id` (no FK yet —
  `canonical_policies` remains entity substrate only).
- `LegacyCorrectionEvent` → canonical `Correction` migration (§14.1) —
  slice 4.
- Automated content-similarity-based independent-source-family detection —
  deferred to a real research-ingestion connector (Phase C+); slice 2 uses
  an explicit, assigned `sourceFamilyId` label instead.
- Full RBAC persistence (grants/revocations table) — slice 2 implements
  only the minimal role-gate function (`canActorVerify`) needed for
  §10.1's automated/verifier/Phil-required transitions; a persisted,
  auditable grants system is a larger cross-cutting concern for a later
  phase.

## Verification (exact)

- `npm run typecheck` — clean.
- `npm test` — **378 passed, 0 failed** (279 pre-existing at slice 1 merge +
  99 new: 25 migration structural tests, 74 unit/adversarial/repository
  tests).
- `npm run build` — succeeds (pre-existing, unrelated `libheif-js`
  critical-dependency warning, present on `main` since before this work).
- Schema-from-empty: **live-verified twice against a real local Postgres 16
  instance** in this session's environment (a working Postgres cluster was
  available this time, unlike slice 1's session) — once ad hoc during
  development (every new CHECK/FK/RLS behavior individually confirmed),
  once replaying the exact three new CI steps, in their final committed
  order, against a freshly-migrated database. Also verified via CI on the
  PR itself.

## Honest limitations carried forward

- `canonical_claims.sources`/`.evidence_anchor_ids` carry no database FK
  (see migration header and above) — application-layer enforcement only.
- Independent-source-family detection is a manually/ingestion-assigned
  label, not automated content-similarity comparison.
- `verifyClaim`'s RBAC check is a pure function over an `ActorRef{id,
  role}` the caller supplies — there is no persisted grants/audit table
  behind it yet (§15's full RBAC model is a larger concern than this
  slice's scope).
- `searchEvidence` filters over `canonical_evidence_sources` only
  (`sourceType`, `sourceFamilyId`); full-text search across
  `EvidenceRevision.extractedText` is deferred to a real research-ingestion
  connector actually worth full-text indexing.

## Next step

PR #6 needs review and, if approved, a merge by someone other than Claude
(never merges its own PR per this project's standing rule). Slice 3
(Repository contract completion — policies, recommendations/packages,
`requestCrossEnvironmentAccess`, the `content_provenance`/`commerce_details`
dual-write bridge) is the natural next assignment once slice 2 lands, but
is explicitly **not** authorized to start automatically — awaiting Phil's
separate go-ahead, same discipline as slice 1 → slice 2.
