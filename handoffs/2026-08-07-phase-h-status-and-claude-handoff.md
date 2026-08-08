# Claude Handoff — Phase H State (H1–H4 merged) and Next Design Work

_2026-08-07. Read once, verify against GitHub before acting. This handoff is a snapshot; the repositories and merged PRs are authoritative._

## Repositories and current SHAs

- **Application** `HydraCoreSystems/gm-commerce` — `main` at `606c5a5301605e05ee71e470cabaf129ec28e589` (merge of PR #47).
- **Headquarters** `HydraCoreSystems/gm-commerce-hq` — branch `agent/phase-h-status-and-claude-handoff` (this documentation). HQ `main` is at `84f430c`.
- Authoritative records: merged PRs, migration files, CI, and the docs in this repo (`phase-h-slice-plan.md`, `STATUS.md`, `DECISIONS.md`). **Model workflow history is not authoritative** — do not try to reconstruct the prior ChatGPT conversation.

## What H1–H4 delivered (all read-only; all commerce capabilities remain `false`)

| Slice | Delivery | PR / merge |
|---|---|---|
| H1 | Canonical commerce → read-only `ReviewPackage` (b0.v1) adapter | #42 / `f44ceb2` |
| H2 | Canonical loader (package + SKU + claims + contradictions) and detail routing | #43 / `9540a1a` |
| H3 | Operational-only commerce queue discovery in `/review-shell` | #44 / `d3cba35` |
| E-corr | Compliance current-state/gate derivation restricted to `record_purpose='operational'` | #45 / `1472660` |
| H4 | Read-only Phase E compliance context on the commerce detail page | #47 / `606c5a5` |

## Architectural invariants to preserve

- **Environment-bound repositories** — every canonical/compliance read goes through an environment-bound repository (`createCanonicalEntityRepository`, `createClaimEvidenceRepository`, `createComplianceCheckRepository`). No request-scoped environment widening.
- **Operational-only discovery and compliance derivation** — the commerce queue discovers only `record_purpose='operational'` packages; compliance current-state/gate considers only operational checks.
- **Canonical claim precedence comes from `queryApplicableClaims`** (`orderByPrecedence`) — never recomputed in application code; no local rank tables.
- **Open contradictions suppress field values** and remain visible as `conflicting` evidence; no side is silently chosen. Resolved contradictions do not suppress fields.
- **Commerce is read-only / all capabilities `false`** (`approve`, `reject`, `targetedRegenerate`, `correctionException`, `legacyEdit`). No legacy-editor link for commerce packages. No mutation controls.
- **Failure isolation** — a legacy failure is reported (never silently an empty success); a commerce or compliance failure never suppresses or reroutes the other path.

## Open issue #46 (does not block H5)

[#46 — Reconcile Phase E compliance gate functions and approval trigger into schema.sql](https://github.com/HydraCoreSystems/gm-commerce/issues/46) is open: the consolidated `schema.sql` omits the Phase E gate functions/trigger (`gmcom_compliance_check_is_stale`, `gmcom_current_compliance_check_status`, `gmcom_compliance_gate_outcome`, `gmcom_compliance_checks_with_status`, `gmcom_guard_commerce_package_approval` + trigger and dependencies). Ordered migrations remain the deployment source of truth, so this does **not** block H5. It is tracked separately; do not fold it into H5 scope without a separate authorization.

## Remaining Phase H read-only gaps (genuine, NOT yet designed)

- Phase C evidence-library context.
- Phase D vision-analysis context.
- Phase F recommendation + `SelectionTrace` context.
- Phase G policy / learned-rule context.

## Immediate next action

**Design the H5 design inventory for Phase C evidence-library context — before any coding.** Produce a read-only design (records/repository method, current-record rule, smallest boundary change, UI, tests, exclusions, blockers), have it reviewed and approved, then implement. Do not begin implementation or scope expansion on the basis of this handoff alone.

## Excluded from Phase H

Commerce approve/reject/correction/regeneration; capability changes; publishing; invented precedence/current-version rules; unrelated pagination work.
