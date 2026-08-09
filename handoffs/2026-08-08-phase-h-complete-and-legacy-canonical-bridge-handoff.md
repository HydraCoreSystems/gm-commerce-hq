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

## 9. SUPERSEDING ADDENDUM (2026-08-08) — I2 delivered and merged

**Current-state correction:** §8's "I2 has not begun and remains pending design and owner authorization" is superseded. **I2 was implemented and merged.**

- **I2 merged via PR #55** (`HydraCoreSystems/gm-commerce`): "Phase I Slice 2: ready_for_ai integration + durable bridge jobs".
- **Merge SHA:** `bef5a5d94aeab4f4b506eb116398a542c5f04886` (`gm-commerce/main`).
- What shipped (verified against the merged migration `20260808120000_phase_i_slice2_ready_for_ai_bridge_jobs.sql` and `app/actions.ts`):
  - `canonical_legacy_bridge_jobs` — a SEPARATE durable queue (per `(environment, legacy_table, legacy_key)`, statuses `pending/processing/done/failed/mismatch/retry/dead_letter`, lease columns, `attempt_count`/`max_attempts`, `available_at` backoff, `correlation_id`, `error_context`) + immutable `canonical_legacy_bridge_job_attempts` ledger.
  - `gmcom_mark_product_ready_for_ai` — guarded transition + enqueue in ONE transaction (enqueue failure rolls back the transition; later canonical failure never reverts it; no-op replay) + idempotent `gmcom_enqueue_legacy_bridge_job` + `gmcom_claim_legacy_bridge_job` (`FOR UPDATE SKIP LOCKED`, 300s lease, lease-expiry recovery, retry/dead-letter) + `gmcom_finish_legacy_bridge_job` (lease-token validated, 60s→1h exponential backoff).
  - **Every real `ready_for_ai` transition now atomically enqueues a durable identity-bridge job.** Request-triggered processing (best-effort after the transition, failure-isolated; plus a bounded manual drain server action `drainBridgeJobs`).
  - **Manual-drain authorization:** `drainBridgeJobs` is owner/co-owner-only via `lib/auth` `resolvePrincipal()` + `requireRole(principal, "co_owner")`; staff/service rejected (no new permission invented); the `/` button is hidden unless the trusted configured principal is authorized; the server-action check is the mandatory boundary.
- **I2 creates ProductConcept/SKU/SourceRecord identities only** — no CommercePackages, Claims, photos/evidence, or Phase H queue entries.
- Post-merge `main` CI for `bef5a5d` independently verified green: CI workflow run `31262417220` = success; Copilot `31262419177` = success.
- **I3 (existing identity backfill) has not begun** and remains pending design and owner authorization; a read-only I3 design inventory is in progress.

The authoritative current state is `STATUS.md` and `phase-i-slice-plan.md` (both updated for I2's delivery).

## 10. SUPERSEDING ADDENDUM (2026-08-08) — I3 delivered and merged

**Current-state correction:** §9's "I3 (existing identity backfill) has not begun and remains pending design and owner authorization" is superseded. **I3 was implemented and merged.**

- **I3 merged via PR #56** (`HydraCoreSystems/gm-commerce`): "Phase I Slice 3: existing identity backfill".
- **Merge SHA:** `285a2a01d229d09597028c332473f5e19cfc1eba` (`gm-commerce/main`).
- What shipped (verified against the merged migration `20260808150000_phase_i_slice3_identity_backfill.sql`, `lib/canonical/bridge/backfill.ts`, `app/actions.ts`, `app/page.tsx`):
  - **Existing eligible legacy products can now be backfilled into canonical ProductConcept/SKU/SourceRecord identity** through a required **dry-run → single-use real-run** workflow (an exact completed `dry_run` authorizes at most one real run; DB-enforced by `source_dry_run_id` NULL-safe CHECK + composite FK for the same environment + guard trigger + partial unique index; **no freshness rule** — a completed dry run has no expiry; the real run uses a new run id and re-evaluates current DB state).
  - Explicit four-status eligibility (`ready_for_ai`/`generating`/`review`/`published`), fail-closed for `intake`/`archived`/unknown.
  - Durable `canonical_legacy_backfill_runs` + immutable per-run `canonical_legacy_backfill_row_outcomes` (run-scoped unique, append-only trigger, no global uniqueness/upserts; counters increment only on real inserts).
  - Run identity/source immutability + a strict run state machine (INSERT must be running with no completed_at/cursor/counters; running/failed have completed_at NULL, completed has completed_at SET; a failed run accepts no outcomes until explicitly resumed; completed is terminal; forged completed runs rejected).
  - `record_outcome`/`finish_run` serialize via `FOR UPDATE` (genuine two-session race proven).
  - Keyset batching + per-row persisted cursor + resumability; owner/co-owner authorization (`runLegacyBackfillDryRun` / `promoteLegacyBackfill(dryRunId)`); UI shows promote buttons only for unconsumed completed dry runs.
- **Backfill is identity-only** — no CommercePackages, Claims, photo/evidence records, Phase H queue entries, publishing changes, or later Phase I work were created.
- Post-merge `main` CI for `285a2a0` independently verified green: CI workflow run `31281957101` = success.
- **I1, I2, and I3 are now all delivered and merged** (PR #54 `2c26b61`, PR #55 `bef5a5d`, PR #56 `285a2a0`). The identity backbone is complete. **The proposed next slice (I4: approved photo sets → canonical PhotoAsset, attached assets only) is a design recommendation only and is NOT authorized** — grounded in a read-only inventory of the merged code at `285a2a0`, and gated on seven owner decisions recorded in `phase-i-slice-plan.md` (photos-as-attached-assets; original-photo vs derivative identity unit; derivative path/metadata representation; canonical PhotoAsset `owner_approval_state='pending'` with legacy approval as source of truth; stable `storage_ref`; **durable per-photo bridge mapping via additive `legacy_table` CHECK widening**; ongoing approval-event integration before backfill — recommended, not authorized). The inventory establishes: `photo_sets.status='approved'` is the truthful approval event (not `products.photos_confirmed_at`); photos are attached/commerce assets, not Claims or Evidence; `canonical_photo_assets` has no `status`/`approved_at` and cannot truthfully carry approval.

The authoritative current state is `STATUS.md` and `phase-i-slice-plan.md` (both updated for I3's delivery).

## 11. SUPERSEDING ADDENDUM (2026-08-09) — I4 delivered and merged; CI workflow-size incident corrected

**Current-state correction:** §10's "proposed I4 is a design recommendation only and is NOT authorized" is superseded. **I4 was implemented and merged.**

- **I4 merged via PR #57** (`HydraCoreSystems/gm-commerce`): "Phase I Slice 4: ongoing approved-photo bridge".
- **Merge SHA:** `e58766e67e5877d8e34187846d63476b7a63c4f9` (`gm-commerce/main`).
- **Post-merge CI:** workflow run `31307655608` = success (independently pulled 2026-08-09); Copilot run `31307656060` = success.
- What shipped (verified against the merged migration `20260808200000_phase_i_slice4_approved_photo_bridge.sql`, `lib/canonical/bridge/processor.ts`, `app/photos/actions.ts`):
  - **Atomic approval + durable photo-job enqueue:** `gmcom_mark_photo_set_approved` guards `needs_review → approved` and enqueues one durable `photo_sets` bridge job in one transaction; **a later processing failure never reverses legacy approval**.
  - **One canonical PhotoAsset per original legacy photo** (preserving `storage_ref` = original path and the new `original_content_hash`; **no files copied or moved**); **derivatives remain linked representations** in the new `canonical_photo_asset_derivatives` table; **canonical approval stays `pending`** (legacy approved photo set remains the source of truth).
  - **Permanent per-photo mappings:** additive `legacy_table` CHECK widening to `photo_assets` + `legacy_approved_at`/`legacy_approved_by` provenance; run/outcome ledger stays separate operational history.
  - **Replay validates the full state:** `record_purpose`, approval state, storage/hash/linkage, full derivative state (missing/extra/duplicate), one shared correlation across assets/mappings/derivatives; partial mappings are never silently repaired (mismatch fails visible); deterministic lock order + two-session concurrency proven.
  - **Fail-closed processor dispatch:** `products` → identity RPC, `photo_sets` → photo RPC; any other `legacy_table` is never routed and finishes `failed`/`bridge_unsupported`.
- **CI workflow-size incident and permanent correction (part of this merge):** PR #57's branch CI produced no runs because `.github/workflows/ci.yml` was **529,068 bytes** — above GitHub's **512,000-byte per-workflow file limit**, which rejects the workflow before any run is created (silent). Fix: the large Phase I live-Postgres SQL heredocs and race scripts were extracted into checked-in `supabase/live/` (`i1-identity-bridge-live.sql`, `i1-identity-bridge-race.sh`, `i2-ready-for-ai-bridge-jobs-live.sql`, `i3-identity-backfill-live.sql`, `i3-backfill-race.sh`, `i4-approved-photo-live.sql`, `i4-photo-race.sh`) and invoked via short `psql -f`/`bash` steps preserving exact order/assertions/env; `supabase/ci-workflow-size.test.ts` enforces a **450 KB** guard. `ci.yml` is now **414,483 bytes**. Extracted tests were run exactly as CI invokes them (I1 12 → I2 18 → I3 15 → I4 20 + all races) on one fresh PG15 before merge and passed in CI on merged `main`.
- **I4 exclusions (all hold):** no historical photo backfill; no Claims or Evidence; no CommercePackage; no Phase H queue population; no publishing changes; no later Phase I work.
- **I5 (historical approved-photo backfill using the I4 bridge machinery) is now the likely next slice — a design recommendation only, NOT authorized.** See `phase-i-slice-plan.md`. The master completion map (`COMPLETION.md`, added 2026-08-09) records the overall finish line, the remaining slice sequence, owner decisions by earliest blocking slice, and measurable exit criteria for Phase I and for GM Commerce as a whole.

The authoritative current state is `STATUS.md`, `phase-i-slice-plan.md`, and `COMPLETION.md` (all updated for I4's delivery).

## 12. SUPERSEDING ADDENDUM (2026-08-09) — Owner-confirmed third output path: the permanent Listings Spreadsheet

The owner confirmed a durable output requirement that **replaces any proposed "downloadable-file" framing of the third output path** (including the older "copy-and-paste sales sheet" / per-listing exportable-document notion — the authoritative record is now this decision, `phase-i-slice-plan.md` decision 12, and `COMPLETION.md`).

- The review/publishing workflow must offer **three mandatory operational destination choices: Shopify, Etsy, and Listings Spreadsheet**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** **Etsy remains scheduled under ROADMAP Milestone 4, which is now explicitly classified as required for overall GM Commerce completion — not optional future work.**
- **Listings Spreadsheet** = **one permanent master Google Sheet**; selecting it **appends one new row to the same configured sheet** and must **not create a new spreadsheet per listing**.
- The user can identify the **intended external destination** on the row (e.g. **Facebook, Palm Street, Whatnot, auction, website, email, or other**).
- **Append-only operational output** — existing rows are never overwritten. Each row preserves the canonical **CommercePackage ID and version**, **export ID**, **timestamp**, **trusted actor**, **intended destination**, **approval state**, **listing fields**, **price recommendation**, and **photo references**.
- **Idempotent** — retries/double-clicks create no duplicate rows; a later **intentional** export of a newer package version may create a new row.
- The **Google Sheet ID and credentials come from trusted server configuration** and never reach the browser.
- Because a Google Sheets write **cannot share a database transaction**, implementation must use a **durable export job/outbox** with retry, idempotency, failure reporting, and an **immutable export ledger** — the database update and the Google Sheet update are **never one atomic transaction** and that must not be claimed.
- **CSV download is optional backup functionality only**, not the primary requirement.

**Status: confirmed requirement — recorded, not implemented, not authorized to implement yet.** It constrains the CommercePackage/output work and will be reflected in the slice design when that work is scoped (no new slice is invented by recording it). Overall completion requires a **proven working path for each of the three mandatory destination choices (Shopify, Etsy, and the Listings Spreadsheet)**; completion testing must demonstrate that an **approved canonical CommercePackage can be routed successfully through Shopify, through Etsy, and through the Listings Spreadsheet without Phil or Crystal rewriting or reformatting the approved content**. A listing is **not** required to be sent to all three destinations simultaneously; for each approved listing Phil or Crystal chooses the intended route.

The authoritative current state is `STATUS.md`, `phase-i-slice-plan.md`, and `COMPLETION.md` (all updated for the confirmed requirement).
