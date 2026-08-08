# Phase H Slice Plan — Read-Only Completed-Review Refinement

_Authoritative plan per `PRODUCT_RESET_2026-08-03.md` §23 Phase H ("completed-review refinement"). Last updated: 2026-08-08._

## Purpose

Phase H deepens the Phase B0 review shell with everything Phases B–G actually produced — **read-only**. It surfaces canonical commerce packages and their real supporting context (compliance, and eventually evidence/vision/recommendation/policy context) in the review UI without adding any mutation capability. Every commerce capability remains `false`; Phase H is strictly read-only.

## Dependency

Phase H depends on:

- **Phase E** (merged): `canonical_compliance_checks`, deterministic validation, the fail-closed approval gate, and the read-only review surface (`gmcom_compliance_checks_with_status`).
- **Phase B** (merged): canonical entities and the `IntelligenceRepositoryV1` contract.
- **Phases B–G** (merged): the canonical records (claims, evidence, vision, recommendations, policies/rules) that later Phase H slices surface read-only.
- **Phase B0** (merged): the review shell, review actions, legacy-editor closeout.

Each slice depends strictly on the previous slice's merge into `main`.

## Current state (2026-08-08)

Application `HydraCoreSystems/gm-commerce` `main` is at `23c4e39f450d779605b221b3466af127d235a29c` (merge of PR #52).

| Slice | Scope | PR | Status | Merge |
|---|---|---|---|---|
| H1 | Canonical commerce → read-only `ReviewPackage` (b0.v1) adapter | #42 | Merged | `f44ceb20e90872ad01f2dadd9a4961d485a11a37` |
| H2 | Canonical loader (package + SKU + claims + contradictions), detail routing | #43 | Merged | `9540a1a01df0fc18bb7808bf3b028288dcd6beec` |
| H3 | Operational-only commerce queue discovery | #44 | Merged | `d3cba3547ebdbabc86424f53e7c2fa1e3693b9e1` |
| Phase E prerequisite correction | Operational-only compliance current-state/gate derivation | #45 | Merged | `1472660232d8475229cd1da07a2c9105d55ff82c` |
| H4 | Read-only Phase E compliance context on commerce detail | #47 | Merged | `606c5a5301605e05ee71e470cabaf129ec28e589` |
| H5 | Read-only Phase C evidence-library context on commerce detail (restores `evidenceAnchorIds` through the H1/H2 boundary; resolves `EvidenceAnchor → EvidenceRevision → EvidenceSource`; pointer-based current-revision rule; `rawContent` never rendered) | #48 | Merged | `4b5ff6abc13789d4d06bb4b50d42615290becf32` |
| Phase F prerequisite correction | Operational-only recommendation current-state derivation (`canonical_recommendations` had no incidental production-only protection, unlike compliance) | #49 | Merged | `372ee23c50faf1bc288148dc51806ccae3e92f35` |
| H6 | Read-only Phase F recommendation + §18 `SelectionTrace` context on commerce detail (per-kind current-recommendation resolution across all 7 kinds; never the kind-omitted form; per-kind failure isolation) | #50 | Merged | `905d658f5996da7b999d070e7ac25b69825e92eb` |
| H7 | Read-only Phase D vision-analysis context on commerce detail (new `listVisionRequestsBySubject` read method, `record_purpose='operational'` filtered from the start; closed 4-type inference union; non-`completed` outcomes render a lighter note, never a fabricated result) | #51 | Merged | `7f5a153b3e23fccece0a87e39b3949288132210a` |
| Phase G prerequisite correction | Operational-only policy/rule read paths (`PolicyRepositoryImpl.fetchPolicies`, `RuleEngineImpl.queryActiveRules`) — same defect class as the Phase E/F corrections, and the same worst-case category as recommendations (`canonical_policies`/`canonical_learned_rules` have no production-forces-operational constraint) | #52 | Merged | `23c4e39f450d779605b221b3466af127d235a29c` |

All commerce capabilities remain `false` (`approve`, `reject`, `targetedRegenerate`, `correctionException`, `legacyEdit`). Phase H remains read-only.

## Remaining Phase H read-only gaps (genuine, NOT yet designed)

The following context surfaces are genuine remaining Phase H read-only gaps. **They are not yet designed** — each needs its own design inventory before any implementation:

- **Phase G policy / learned-rule context** — surfacing applicable policies and active learned rules read-only. This is the last remaining documented Phase H gap, and it is now unblocked: Phase G Slice 5 (RBAC) is confirmed merged (PR #34), and the prerequisite operational-purpose correction for policy/rule reads is merged (PR #52). **Proposed next work is the H8 design inventory for this gap** (design only, not implementation).

## Explicitly excluded from Phase H

- Commerce approve/reject/correction/regeneration actions.
- Capability changes (all commerce capabilities remain `false`).
- Publishing.
- Invented precedence/current-version rules (canonical `queryApplicableClaims` / `orderByPrecedence` are the only precedence authorities).
- Unrelated pagination work (ordered pagination of the commerce queue is a future shared-repository concern, tracked separately from Phase H).

## Architectural invariants (all H1–H5 slices)

- **Environment-bound repositories** — every canonical/compliance read goes through an environment-bound repository (`createCanonicalEntityRepository`, `createClaimEvidenceRepository`, `createComplianceCheckRepository`); no request-scoped environment widening.
- **Operational-only discovery and compliance derivation** — the commerce queue discovers only `record_purpose='operational'` packages, and compliance current-state/gate derivation considers only operational checks.
- **Canonical claim precedence** comes from `queryApplicableClaims` (`orderByPrecedence`) — never recomputed in application code.
- **Open contradictions suppress field values** and remain visible as `conflicting` evidence; no side is silently chosen.
- **Commerce is read-only** — all capabilities `false`; no legacy-editor link for commerce packages.
- **Failure isolation** — a legacy failure is reported (never an empty success) and a commerce/compliance failure never suppresses or reroutes the other queue/path.

## Related tracking

- **Issue #46** — `Reconcile Phase E compliance gate functions and approval trigger into schema.sql` (open). Ordered migrations remain the deployment source of truth; this does not block Phase H.
