# Phase E slice plan — independent validation and marketplace-compliance gates

Status: **Plan approved by Phil (2026-08-04). E1 authorized to begin implementation.
E2/E3/E4 remain unauthorized until E1 lands as an unmerged PR and passes
independent review — each slice's authorization is separate and does not carry
forward to the next.**

Scope authority: `PRODUCT_RESET_2026-08-03.md` §12 (Marketplace-policy compliance
gate), §13 (Required persisted artifacts, item 6), and §23 ("Phase E — independent
validation and marketplace-compliance gates ... plus the deterministic/
evidence-grounded validator split from Revision 2 §4/§6"). §1's disposition table
is the authority for what legacy code migrates in vs. is discarded.

Prerequisite: Phase D (image understanding) is merged to `gm-commerce` `main`
as of `9322ff2` (all four slices, CI green). Phase E slices below assume that
tip as their starting point.

## Current-state check (verified against `gm-commerce` `main` before writing this
plan, not assumed from the doc alone)

- `canonical_commerce_packages` table already exists (Phase B Slice 1). No
  `ComplianceCheck` record type exists yet anywhere in `lib/canonical/entities.ts`
  or the schema — it is net-new for Phase E, same category as the `MetricEvent` /
  `KnowledgePromotionCandidate` / `LegacyCorrectionEvent` record types the reset
  doc calls out as "implied but not previously named."
- The legacy `lib/commerce-readiness/` gate (`gate.ts`, `denylist.ts`, `taxonomy.ts`,
  `defaults.ts`, `types.ts`, `load.ts`) is local-only, untracked, never run in CI,
  and per §1's disposition table is to be **migrated** (deterministic checks become
  inputs to Phase E), not discarded and not kept as its own standalone gate module.
- The alt-text automatic duplicate-repair mechanism (deterministic detection → one
  targeted repair call → fail closed if still duplicated) already exists and is
  covered by passing tests; per §1 it is to be **retained** but reframed as an
  output of Phase E's independent-validation layer rather than folded into the
  generation call itself.
- Phase C Slice 5 (freshness/revalidation/promotion gates, merged) already built
  the pattern Phase E's marketplace-policy staleness cascade needs: versioned
  policy rows, "what is currently active" derived correctly (not a mutable flag),
  and revalidation triggered by policy change. Phase E's `ComplianceCheck` staleness
  behavior (§12: a policy change invalidates every check whose `policySnapshotId`
  predates it) should reuse that pattern rather than re-inventing it.

## Slices (finite, dependency-ordered — each builds on the previous slice's tip)

### E1 — `ComplianceCheck` record type and schema
Data-model boundary only, mirroring how Phase D Slice 1 established the
vision-provider contract before any vendor wiring existed.
- New `ComplianceCheck` record type (§12): `id`, `commercePackageId`,
  `policySnapshotId`, `marketplace`, `shop`, `region`, `checkedAt`, `freshnessTtl`,
  `result` (`PASS`/`FAIL`/`STALE`), `violations[]` (`rule`, `detail`,
  `adapterMapping`).
- New `canonical_compliance_checks` table, RLS/environment-scoping consistent with
  every other canonical table in this repo, append-only or versioned per the same
  reasoning Phase C Slice 5 already worked through for freshness policies (a
  compliance check result is a historical fact — it should not be silently
  mutated in place).
- **Must remain marketplace-neutral.** `ComplianceCheck` is the record of a check
  result, parameterized by `marketplace`/`shop`/`region` — it must not bake in
  any marketplace-specific rule logic itself (that's E2/E3's job, and even there,
  rules are versioned policy artifacts, never hardcoded — see E2 below).
- **Must carry a complete `RecordContext`** (§25's envelope — `environment`,
  `recordPurpose`, `sourceRun`, `correlationId`, `ownerApproval`, `eligibility`,
  `retention`), the same envelope every other canonical entity in this repo
  carries. No shortcut/partial context.
- **Must carry field-level lineage** for whatever `CommercePackage` fields the
  check evaluated — i.e., a `ComplianceCheck` needs to be traceable back to the
  specific field(s) it assessed, consistent with §7's `fieldLineage[]` model,
  not just a package-wide pass/fail blob.
- No gate-enforcement logic yet, no deterministic checks wired in yet — this
  slice only proves the record type can be created, queried, and correctly
  scoped/versioned.

### E2 — Deterministic validation layer (migrate `lib/commerce-readiness/*`)
- **Compliance rules themselves must be versioned, marketplace-specific policy
  artifacts** — not hard-coded denylists relocated as-is. Each rule needs
  provenance (where it came from / who authorized it), an effective-date range,
  and its own freshness/staleness status, following the same versioned-policy
  pattern Phase C Slice 5 built for freshness policies and Phase E's own
  `policySnapshotId` concept. Porting `lib/commerce-readiness/denylist.ts`'s
  *logic* is fine; porting its hardcoded list *as a static constant* is not —
  the list itself becomes policy data, versioned and queryable, not code.
- Port the deterministic checks (denylist, near-duplicate detection, structural
  completeness) out of the untracked `lib/commerce-readiness/` files into Phase
  E's own module, operating against canonical `CommercePackage` / `Claim` data
  instead of the legacy `listing_packages` shape.
- The old field-by-field blocking logic is **not** ported — it's superseded by
  the canonical `CommercePackage` readiness state (§7) and the compliance gate
  (§12), per §1's disposition.
- **Every finding must identify**: the exact field it applies to, the specific
  rule that fired, the policy version checked against, the evidence backing the
  finding, a severity level, and a remediation (what would need to change to
  pass). A finding that only says "FAIL" without all six of these is incomplete.
- Each deterministic check run produces a persisted validation-result artifact
  (§13 item 6 — "Deterministic validation results, formalized as their own
  artifact rather than folded silently into a pass/fail").
- Reframe the existing alt-text duplicate-repair mechanism as an explicit output
  of this layer.

### E3 — Marketplace-compliance fail-closed gate
- Wire `ComplianceCheck` results into `CommercePackage` approval: a package
  cannot reach `approved` while its `ComplianceCheck` for the target marketplace
  is `FAIL` or `STALE` (§12, fail-closed, no exceptions).
- Marketplace-policy-change staleness cascade: a policy update invalidates every
  `ComplianceCheck` scoped to that marketplace whose `policySnapshotId` predates
  the change — they become `STALE` immediately, never silently still-passing.
  Reuse the Phase C Slice 5 "derive current state correctly, don't freeze a flag
  at write time" pattern here directly; do not reintroduce that bug class.
- **Invalidation must never delete historical results.** A `ComplianceCheck`
  that becomes `STALE` is a new state derived/recorded, not a deletion or
  overwrite of the prior result — the historical record of what passed/failed
  and when must remain queryable. Readiness is blocked going forward; history
  is preserved.
- CI coverage in the same live-Postgres pattern already used for Phase C Slice 5
  and Phase D: real fail-closed assertions, not mocked.

### E4 — Persisted-artifact wiring and read-only compliance review surface
- Confirm all nine §13 artifacts are actually threaded by one `correlationId` for
  a full recommendation run, with item 6 (deterministic validation results) and
  the `ComplianceCheck` records included end-to-end.
- Add a read-only review page (same pattern as Phase D Slice 4's
  `/photos/[sku]/vision` page) surfacing `ComplianceCheck` results per
  `CommercePackage` — pass/fail/stale, violations, which policy version was
  checked against. **The page must present a completed compliance assessment
  and any exceptions clearly — it exists so Phil can review outcomes, not so
  Phil has to perform routine compliance work himself.** No approve/override
  actions in this slice; owner-override
  mechanics belong to §14's `Correction`/`OwnerDecision` flow, out of scope here.

## Explicitly out of scope for Phase E (do not let any slice's agent drift into
these)
- Phase F's recommendation services (price, taxonomy, collections, marketplace
  suitability, photography, SEO, merchandising) and `SelectionTrace` — Phase E
  validates; it does not recommend.
- Etsy, public editorial publishing, portfolio recommendations — explicitly
  later/unsequenced phases per §23.
- Any UI beyond the one read-only review page in E4 — no owner-action UI, no
  publish-flow UI.
- Reworking `lib/vision/` or anything from Phase D — Phase E consumes vision
  claims as inputs where relevant; it does not modify Phase D's contract.

## Process, same discipline used for Phase D
- Each slice branches from the prior slice's tip (`agent/phase-e-slice-N-...`),
  stacked, one PR per slice.
- No PR is merged without Phil's explicit, separate authorization per slice —
  authorization for one slice is not authorization for the next.
- Every "done" report must be independently verified against real GitHub state
  (branch existence, exact HEAD SHA, real CI conclusion, actual file diff scope)
  before being reported as complete — self-reported completion from any agent,
  cheap or expensive, is not sufficient on its own.

---

**This document proposes the plan only. No branch, commit, or code for Phase E
has been created. Implementation does not begin until Phil authorizes this plan
(or a revised version of it).**
