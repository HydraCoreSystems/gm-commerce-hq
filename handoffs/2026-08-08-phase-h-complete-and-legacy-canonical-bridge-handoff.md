# Handoff to ChatGPT (or any successor coordinator) — Phase H complete; critical architectural gap found and next phase proposed

_2026-08-08. Written by Claude under an active session-usage constraint — kept as complete and actionable as possible in one pass. Verify all SHAs against GitHub before acting; this is a snapshot, not a substitute for live repository state._

## 1. Where things stand — Phase H is done

`HydraCoreSystems/gm-commerce` `main` is at **`c65b0232d834c200195b38ce992d1e42272954d1`** (merge of PR #53). Phase H (`gm-commerce-hq/phase-h-slice-plan.md`) is fully complete: all eight slices (H1–H8) plus three prerequisite corrections (Phase E, F, and G record_purpose fixes) are merged. Full table with every PR number and merge SHA is in `phase-h-slice-plan.md` — do not reconstruct it from memory, read that file.

What H1–H8 built: the commerce package detail page in `/review-shell` now shows, read-only, for a canonical `CommercePackage` — compliance gate status, evidence-library provenance, Phase F recommendations with `SelectionTrace`, Phase D vision-analysis results, and applicable Phase G policies/learned rules. All five commerce capabilities (`approve`, `reject`, `targetedRegenerate`, `correctionException`, `legacyEdit`) remain `false`. Nothing here mutates anything.

Three real defects were found and fixed during this work (all the same class): `gmcom_current_compliance_check`/`gmcom_compliance_gate_outcome` (PR #45), `gmcom_current_recommendation` (PR #49), and `PolicyRepositoryImpl.fetchPolicies`/`RuleEngineImpl.queryActiveRules` (PR #52) all originally derived "current"/"applicable" state without filtering `record_purpose`, meaning a test/demo/fixture row could contaminate real operational derivation — in two of the three cases (recommendations, policies/rules) with **no incidental production-only protection at all**, since those tables predate the `production-forces-operational` constraint pattern. All three are now fixed and covered by regression tests. This pattern is worth remembering if you scope any new "current state" derivation later — check for this exact bug class every time.

One process note: PR #52's first CI run genuinely failed (a test-isolation bug in the new CI step itself, not the fix) — the implementer's own report claimed it passed when it hadn't. Caught only because I independently re-pulled CI status rather than trusting the report. **Do not trust an implementing agent's "CI passed" claim without independently checking the actual GitHub Actions run** — this is not optional discipline, it caught a real problem.

## 2. The critical finding — Phase H's review surface has nothing real to review yet

This is the important part. In the course of scoping "what's next," I found that **the canonical model Phase B through Phase H were built around has no live population path from real intake.**

The real, working, production system (GMCOM-001 through GMCOM-012, all verified — GMCOM-012 confirmed end-to-end against a real Shopify store) is:

**Skrybix selection** (a human selects a plant cutting in Skrybix itself; `lib/sources/skrybix.ts` polls and imports — GMCOM-003) **or direct Product-SKU-Generator selection** (for non-plant/non-Skrybix products) → both converge into a **SKU-named photo folder** (`lib/photo-root.ts`, explicitly built to give Skrybix-origin products a folder since "Skrybix has no folder concept at all today") → human adds photos and **explicitly confirms** ("I've added photos," GMCOM-006 — a deliberate human attestation, not inferred from a file count) → **AI listing generation** (GMCOM-007/008/009/010) → **Shopify draft publish** (GMCOM-012).

This entire pipeline writes into the **legacy schema**: `products` (SKU-keyed, `source_system` check `'skrybix'|'product_sku_generator'`, pipeline `status` enum `intake → ready_for_ai → review → published → archived`, `photo_folder_path`), `listing_packages` (SKU-keyed, AI-generated `proposed_title`/`description`/`price`/`tags`, `review_status`), `photo_sets`, `commerce_details`.

**Nothing bridges this into the canonical model** (`canonical_product_concepts`, `canonical_skus`, `canonical_commerce_packages`, `canonical_claims`, etc. — the eighteen §5 entity types Phase B built, and everything Phase C–H layered on top of). I grepped the entire application for any writer to `canonical_product_concepts`/`canonical_skus` outside the generic repository framework itself: **zero**. `gm-commerce-hq/DECISIONS.md` names this exact gap explicitly (in a Phase F decision that had to route around it): *"the GMCOM-011-to-canonical bridge"* has "not been designed or approved."

**Practical consequence:** unless canonical rows are being created some other way I haven't found (worth confirming directly with Phil — ask whether anyone seeds canonical entities manually today), Phase H's entire review surface — everything H1 through H8 built — has no real `CommercePackage`s to show in production. The rigorous canonical system and the real working system are two parallel tracks that never merged.

## 3. What Phil confirmed, and the proposed next phase

Phil confirmed this matches his original intent (the Skrybix/SKU-Generator → photo-folder → confirm → generate → publish pipeline is exactly what he designed and wants) and agreed the bridge is the right next priority — **ahead of Etsy, ahead of anything else**, because Etsy (§7) and everything else in the canonical/Phase-H-built system is specifically supposed to consume `CommercePackage`, which currently has no real population path.

Also clarified: "public editorial publishing" in `PRODUCT_RESET_2026-08-03.md` (§17/§18, `EditorialAsset`) means **public educational/informational content** (e.g. a care guide), explicitly split away from marketplace listings by design (Codex Correction #1: "so this section's guarantees apply only to genuine public editorial content, never to a commerce listing"). Phil had been assuming it meant something else (an exportable document for manual listing on platforms without a live adapter) — that turned out to be his own assumption, not a documented feature, and is now resolved as a non-issue; no separate feature request came out of it. Don't conflate the two if it comes up again — `EditorialAsset` is educational content, not a listing-export mechanism.

## 4. Proposed next phase — smallest-first-slice only, NOT a full plan

I have not had time under the usage constraint to do a full multi-slice plan the way Phase B/C/D/E/F/G each got one. This is real, substantial scoping work someone needs to do properly — do not skip it and jump straight to implementation.

**Working name: Phase I — Legacy-to-Canonical Bridge.**

**Proposed smallest first slice (I1):** when a `products` row reaches `status = 'ready_for_ai'` (the same gate GMCOM-006 already established — photos confirmed), also create the corresponding canonical `ProductConcept` + `SKU` rows (identity only — no `CommercePackage`, no claims, no photo-evidence bridge yet). This mirrors how Phase B itself started (entities before claims before everything else) and is the minimum needed to make Phase H's review surface show something real for the first time.

**Explicitly NOT scoped yet, needs its own design work before any slice touches it:**
- How `listing_packages`' AI-generated content (title/description/price/tags) becomes canonical `Claim`s with proper `sourceCategory`/precedence — this is the biggest open design question, since claims need evidence anchors and a defensible source category, and AI-generated listing copy doesn't obviously map to the existing `ai_research`/`derived_rule`/etc. categories without a real decision.
- Whether/how `photo_sets` becomes `PhotoAsset` + evidence records.
- Whether this is a one-time backfill of existing legacy rows, an ongoing dual-write, or both.
- Whether `canonical_commerce_packages` gets created in this same phase or a later slice.
- Whether this is even the right architecture, or whether Phil wants a different reconciliation approach entirely — **do not assume slice I1 is authorized; it is a proposal, not a decision.**

## 5. Standing operating discipline (carry this forward exactly)

- Never accept an implementing agent's (DeepSeek's) self-report as fact — independently re-verify the actual diff and the actual GitHub Actions CI run before any verdict, every time, no exceptions. PR #52 is the proof this matters.
- Every implementation goes: ground the design against actual code/schema first (not assumption) → write a complete, self-contained, copy-paste-ready brief for DeepSeek in a single fenced code block with NOTHING addressed to Phil inside it → DeepSeek pushes a branch and opens a **draft** PR, explicitly instructed it is not authorized to mark ready or merge under any circumstance → independent review of the actual diff → independent CI check → only on Phil's explicit, separate "go"/"proceed" does the PR get marked ready and merged → `gm-commerce-hq` (`STATUS.md` and the relevant phase-plan doc) gets updated and pushed immediately after every merge, recording the exact merge SHA.
- Merge method is always a real merge commit (`merge`), never squash/rebase — confirmed as this repo's established convention across every prior PR.
- Format discipline Phil explicitly asked for: anything meant to be copy-pasted to DeepSeek goes in one fenced code block, nothing else mixed in.
- Coordination between AI contributors has no live channel — everything routes through Phil relaying manually. Keep briefs fully self-contained for exactly this reason.

## 6. Immediate next action for whoever picks this up

Do NOT start implementing slice I1 yet. First: properly scope Phase I the way every prior phase got scoped — read the full legacy schema (`products`, `listing_packages`, `photo_sets`, `commerce_details` — starting points are in `supabase/schema.sql`), read `lib/photo-root.ts` and `app/select/actions.ts` for the full intake flow, decide the claims-mapping question in section 4 above, and get Phil's explicit authorization on the resulting slice plan before any branch is created. This is a bigger, higher-stakes body of work than any individual H-slice — treat it with at least the same rigor, not less.

---

## 7. SUPERSEDING ADDENDUM (2026-08-08) — Phase I plan is now authoritative; the proposed-I1 language in §4 is superseded

Phase I was properly scoped (read-only design inventory → approved plan). **The authoritative document is now `phase-i-slice-plan.md` (this repo).** §4's "proposed smallest first slice" and its language are superseded by the approved decisions below. **No Phase I slice, migration, RPC, or application change has been implemented.** I1 requires a separate owner go.

### Approved owner decisions (verbatim; supersede any earlier proposed-I1 language)

1. Canonical identity creation is triggered when a legacy `products` row reaches `status='ready_for_ai'`.
2. Create a new `canonical_legacy_entity_bridge` table. Do **not** widen or repurpose `canonical_legacy_field_bridge`.
3. ProductConcept + SKU + SourceRecord + entity-bridge mapping must be created **atomically by one database RPC**.
4. The legacy pipeline must **not roll back** because canonical bridging fails.
5. A durable bridge job/outbox must record `pending`, `processing`, `done`, `failed`, `mismatch`, `retry` information, `correlation_id`, and error context.
6. **Activate ongoing production bridging before backfilling** existing rows.
7. Initially **skip archived rows and record them as excluded**; do not invent retention behavior.
8. Do **not** create canonical Claims for AI-generated listing content until a defensible SourceCategory/evidence design is explicitly approved.
9. **I1 does not make anything visible in the Phase H queue.** That requires a canonical CommercePackage in a later slice.
10. CommercePackage creation timing remains a later design decision (likely generation finalization / entry into review), but is **not** authorized as an I1 assumption.
11. The legacy-to-canonical bridge architecture itself is approved as the next phase.

### Correction to §4's claim

§4 said the smallest I1 (identity at `ready_for_ai`) is "the minimum needed to make Phase H's review surface show something real for the first time." That is **incorrect**: Phase H's queue lists canonical **CommercePackages**, not SKUs. Only a canonical CommercePackage can enter the Phase H queue. I1 (ProductConcept + SKU + SourceRecord + mapping) is required groundwork but makes nothing visible in Phase H.

### Immediate next action (updated)

Read `phase-i-slice-plan.md`. Do not implement I1 until Phil issues the separate owner "go" for I1. The plan records per-slice entry criteria, exclusions, dependencies, and the unresolved decisions that block each later slice (and the one hard owner-gated blocker: AI-content Claims, decision 8).

## 8. SUPERSEDING ADDENDUM (2026-08-08) — I1 delivered and merged

**Current-state correction:** §7's statement "No Phase I slice, migration, RPC, or application change has been implemented" is superseded. **I1 was implemented and merged.**

- **I1 merged via PR #54** (`HydraCoreSystems/gm-commerce`): "Phase I Slice 1: identity bridge foundation + atomic identity RPC".
- **Merge SHA:** `2c26b61dafae3ac79cdb54dced7160adab06a7fd` (`gm-commerce/main`).
- What shipped: `canonical_legacy_entity_bridge` (durable, environment-scoped bridge/outbox substrate; statuses `pending/processing/done/failed/mismatch/retry/excluded`; `correlation_id`, `retry_count`, `error_context`, `first_bridged_at`, `last_bridged_at`) and the atomic `gmcom_bridge_product_identity(p_environment, p_sku, p_correlation_id)` RPC creating ProductConcept → SKU → SourceRecord → mapping in one transaction (operational-only, `owner_approval_state='pending'`, idempotent replay, drift/mismatch fail-visible, concurrent-winner convergence, atomic rollback, service_role-only execute, RLS fail-closed + env-scoped). Consolidated `schema.sql` mirror updated in the same merge.
- **Genuine two-session concurrency was tested and passed:** two simultaneous PostgreSQL sessions calling the RPC for the same eligible product both return the same ProductConcept/SKU/SourceRecord IDs, with exactly one identity set and three bridge rows remaining.
- **I1 still does not create CommercePackages and does not populate the Phase H queue.** I2 has not begun and remains pending design and owner authorization.
- Post-merge `main` CI for `2c26b61` was still queued at the time of this update (Copilot workflow completed success); verify the CI workflow independently.

The authoritative current state is `STATUS.md` and `phase-i-slice-plan.md` (both updated for I1's delivery). This addendum exists to correct the outdated "no implementation begun" current-state fact while preserving the historical record above.
