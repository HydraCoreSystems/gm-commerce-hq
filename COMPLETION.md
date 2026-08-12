# GM Commerce — Master Completion Map

> Plain-English answer to: **What must be finished before GM Commerce is considered operationally complete?**
>
> Owner-facing. Derived strictly from `ROADMAP.md`, `PRODUCT_RESET_2026-08-03.md`, `phase-i-slice-plan.md`, `DECISIONS.md`, and `STATUS.md` — no new capabilities, dates, SourceCategories, Claims, evidence rules, or lifecycle states are invented here. Last updated: 2026-08-12 (Phase I complete through PR #68; Phase 0 Slices 1–3 merged via PRs #69–#71).

## The finish line in one sentence

GM Commerce is operationally complete when the real product pipeline — select a product, add and approve its photos, generate a listing, review it, publish it to its chosen destination — runs day to day **with a complete canonical mirror**: every real product has permanent identity and photo records in the canonical system, every approved listing becomes a reviewable commerce package in the review shell, and the workflow offers **three mandatory operational destination choices — Shopify, Etsy, and the permanent Listings Spreadsheet — with a proven working path for each** (Phil or Crystal chooses the intended route for each approved listing; a listing is not required to be sent to all three), and no routine step falls back to Phil or Crystal when the system can do it (the governing completion test in `PRODUCT_RESET_2026-08-03.md`).

## What is already finished

**The real pipeline (GMCOM-001–012).** Product selection from Skrybix and the Product SKU Generator, SKU-named photo folders, human photo confirmation, AI listing generation, human review, and Shopify draft publishing — verified end to end against a real Shopify store.

**The canonical system (Phases A–H).** The 18 canonical entity tables, the evidence/claims model, compliance checks, recommendation services, vision analysis, owner-editable policies and learned rules, and the read-only review shell. Phase A reconciled the migration ledger and schema with the live database.

**Phase I — the legacy-to-canonical bridge (I1–I5, CommercePackage, claims, drift, routing, and all three destination adapters — complete).**

- **I1** (PR #54): any product that reaches `ready_for_ai` gains canonical ProductConcept/SKU/SourceRecord identity atomically, with a permanent mapping.
- **I2** (PR #55): every real `ready_for_ai` transition also enqueues a durable bridge job; request-triggered processing, leases, retries, dead-lettering, and an owner-only manual drain.
- **I3** (PR #56): existing products can be backfilled into canonical identity through a required dry-run → single real-run workflow, with an immutable outcome ledger and no duplicates.
- **I4** (PR #57): approved photo sets are bridged into canonical PhotoAssets.
- **I5** (PR #58): historical approved-photo backfill (dry-run promotion, I4 reuse, idempotency) — implemented and merged, no longer a design recommendation only.
- **CommercePackage creation** (PR #59): canonical CommercePackages are created at generation finalization — the slice that populated the review shell.
- **CommercePackage content assembly** (PR #60): exact-version listing content, photo references, and field lineage.
- **Truthful claims mapping** (PR #61): AI-content claims + field lineage, `under_review` only, no fabrication.
- **Drift monitoring** (PR #62): bounded resumable read-only scans, no repair.
- **Review-shell display** (PR #63): assembled canonical package content shown in the review shell.
- **Atomic canonical review decisions** (PR #64): approve/reject atomicity, retry, rollback, fail-closed.
- **Canonical destination routing + outbox** (PR #65): durable enqueue, lifecycle, ledger, idempotency, eligibility.
- **Listings Spreadsheet adapter** (PR #66): durable export ledger, lease-protected claim, atomic completion, error allowlist.
- **Canonical Shopify draft adapter** (PR #67): lease-protected claim, atomic marketplace success, immutable attempts, error allowlist.
- **Canonical Etsy draft adapter** (PR #68): create-intent-before-remote-call, `needs_confirmation` parking, single-use recreate, immutable resolution events.

**Phase 0 Slice 1 — environment and legacy-access hardening (PR #69):** fail-closed environment propagation (no `'production'` defaults), legacy-table RLS/grants, hardened legacy-writing RPCs. Delivered and merged; post-merge CI green (run `31565668557`, 24/24 jobs, with the documented deferred-drift skip).

**Phase 0 Slice 2 — current-version invariant and regeneration safety (PR #70, merge `9cc5f6a`, migration `20260816000000_phase0_slice2_current_version_safety.sql`):** one authoritative current canonical CommercePackage per environment and SKU. Completed and merged; post-merge CI green (run `31605446557`, 24/24 jobs).

- **Current-version invariant** — one current package per environment+SKU, deterministic highest-version convergence (parent `canonical_skus` row serialization; lock order `canonical_skus → canonical_commerce_packages → canonical_destination_requests`; partial unique index as defense in depth). Higher versions atomically supersede older current packages; lower versions remain historical/`superseded`; equal-version bridge replay is idempotent; the previous package stays authoritative until its replacement successfully materializes.
- **Regeneration authority-transfer safety** — stale approval/enqueue/claim/retry/success paths fail closed; nonterminal requests for superseded packages become terminal `failed`/`package_superseded`; historical packages, decisions, requests, and audit events are preserved.
- **Shopify pre-side-effect dispatch authorization** and **Listings Spreadsheet pre-side-effect dispatch authorization** — `gmcom_authorize_destination_dispatch` is called immediately before each external write; the unavoidable database-to-external-API micro-window remains documented (authorization is placed immediately before writes; post-call validation remains defense in depth).
- **Etsy stays implemented but not launch-active and fail-closed** — Phase 0 Slice 2's generic database protections cover Etsy requests; Etsy application-level pre-side-effect authorization is **intentionally deferred** to the Etsy activation phase (the missing protection is not optional). Resume only when Phil explicitly begins and authorizes Etsy activation. Authoritative technical checklist: `gm-commerce/docs/canonical-etsy-draft.md` §10.

**Phase 0 Slice 3 — destination-request deduplication (PR #71, merge `851be71`, migration `20260817000000_phase0_slice3_destination_dedup.sql`):** at most one ACTIVE operational destination request per delivery intent. Completed and merged; post-merge CI green (run `31615866708`, 24/24 jobs).

- **Destination-request double-submission deduplication** — delivery-intent identity `(environment, commerce_package_id, destination, intended_external_destination, custom_destination_label)`; partial unique index for ACTIVE operational requests; sequential and concurrent duplicate submissions (different correlation IDs included) converge to the existing request; active requests take precedence over terminal history; terminal-only history follows the established no-redelivery rule; exact-correlation replay stays idempotent and mismatched correlation reuse fails closed; superseded packages remain ineligible via Slice 2; Etsy remains inactive and fail-closed; Etsy recreate and same-row retry behavior are preserved.
- **Upgrade reconciliation** — keeps the oldest ACTIVE request; discarded duplicates become terminal `failed`/`duplicate_intent` (distinct from `package_superseded`, which remains reserved for actual package supersession); the surviving request is recorded in audit metadata. Historical requests and events are preserved. Deliberate redelivery is not implemented and requires a future explicit owner-authorized mechanism.

## What is true right now (current end state)

- `gm-commerce/main` is at **`851be71`** (merge of PR #71, Phase 0 Slice 3); post-merge CI green (workflow run **`31615866708`**, 24/24 jobs).
- The review shell is populated by **real canonical CommercePackages** (created at generation finalization, assembled with exact-version content and photo references, and displayed in the review shell). The earlier "the review shell has no real CommercePackages" statement no longer holds.
- Every new `ready_for_ai` product automatically gains canonical identity (I1 + I2). Existing products can be backfilled on demand (I3). Every approved photo set gains permanent canonical PhotoAssets (I4); historically approved photo sets can be backfilled (I5).
- AI-content Claims are bridged truthfully (`under_review` only, no fabrication). Drift is monitored (read-only, no repair).
- All three destination adapters — **Shopify** (PR #67), **Etsy** (PR #68), and the **permanent Listings Spreadsheet** (PR #66) — are **implemented** and write through the canonical destination outbox (PR #65).
- Legacy remains the working source pipeline; canonical approval is never fabricated.

## Implemented vs. launch-ready — the remaining finish line

"Implemented" does not mean "launch-ready." The following launch conditions remain **incomplete** and are part of the finish line:

| # | Remaining launch item | What it is | Status |
|---|---|---|---|
| 1 | **Current-version invariant and regeneration safety** | One authoritative current canonical CommercePackage per environment and SKU; deterministic highest-version convergence under concurrent materialization; stale approval/enqueue/claim/retry/success fail closed; nonterminal requests for superseded packages become terminal `failed`/`package_superseded` (review finding D1/High). | **Fixed and merged** (Phase 0 Slice 2, PR #70, `9cc5f6a`). |
| 2 | **Destination-request deduplication** | At most one ACTIVE operational destination request per delivery intent `(environment, commerce_package_id, destination, intended_external_destination, custom_destination_label)`; sequential and concurrent duplicate submissions converge; upgrade reconciliation records duplicates as `failed`/`duplicate_intent` (review finding D2/High). | **Fixed and merged** (Phase 0 Slice 3, PR #71, `851be71`). |
| 3 | **Automatic workers, retries, and queue maintenance** | No automatic background consumer exists; processors are manual server actions (review finding C3/High — a product gap). Required for launch per owner decision 3 (processing/retries/queue/destination creation must be automatic). | **Not implemented.** |
| 4 | **Batch processing (Phase 2)** | REQUIRED before launch. **Mandatory owner-design checkpoint:** must not be designed or implemented until Phil approves how it should work. | **Not begun; design gated on Phil.** |
| 5 | **Google Drive → Supabase Storage photo architecture** | Google Drive = human master; Supabase Storage = application copies/derivatives; OneDrive = independent backup/archive (owner decision). | **Not implemented.** |
| 6 | **Shopify draft-only end-to-end launch verification** | Shopify launches first; a draft-only end-to-end launch verification against the real store has not been executed as the launch gate. | **Not executed.** |
| 7 | **Etsy configuration and launch readiness** | Etsy stays fail-closed until its token store and policy source are configured and verified (owner decision). The Etsy draft adapter is implemented (PR #68) but is not launch-ready. Etsy application-level pre-side-effect dispatch authorization is also deferred to the Etsy activation phase — the missing protection is not optional (authoritative checklist: `gm-commerce/docs/canonical-etsy-draft.md` §10). | **Not configured/verified; authorization deferred.** |
| 8 | **Dependency-safe legacy cutover** | Legacy retired only after dependency-ordered cutover (inventory dependencies/capabilities → map canonical replacements → migrate and verify → stop new legacy writes/routing → confirm no canonical dependency → retire legacy runtime paths). No premature deletion. | **Not begun.** |
| 9 | **Deterministic recovery → constrained AI recovery → owner escalation** | Recovery ladder: deterministic automation → constrained AI "fresh-eyes" agent → owner escalation. | **Not implemented as an operational ladder.** |

## Confirmed owner requirement — the permanent Listings Spreadsheet (mandatory third output path)

The owner confirmed (2026-08-09) that the review/publishing workflow must offer **three mandatory operational destination choices — Shopify, Etsy, and Listings Spreadsheet**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** The third one is **one permanent master Google Sheet**, not a new spreadsheet (or downloadable file) per listing.

- Selecting **Listings Spreadsheet** must **append one new row to the same configured Google Sheet**. It must **not create a new spreadsheet for each listing**.
- The user must be able to identify the **intended external destination** for the row — e.g. **Facebook, Palm Street, Whatnot, auction, website, email, or other**.
- The sheet is **append-only operational output**: existing rows are never overwritten.
- Each row preserves: the canonical **CommercePackage ID and version**, **export ID**, **timestamp**, **trusted actor**, **intended destination**, **approval state**, **listing fields**, **price recommendation**, and **photo references**.
- **Idempotent**: retries or double-clicks must not create duplicate rows. A later **intentional** export of a newer package version may create a new row.
- The **Google Sheet ID and credentials come from trusted server configuration** and never reach the browser.
- Because a Google Sheets write **cannot share a database transaction**, the implementation must use a **durable export job/outbox** with retry, idempotency, failure reporting, and an **immutable export ledger**. The database update and the Google Sheet update are **never one atomic transaction** — that must not be claimed.
- **CSV download is optional backup functionality only**, not the primary requirement.

**Status: the Listings Spreadsheet adapter is IMPLEMENTED (PR #66, durable export ledger, lease-protected claim, atomic completion, error allowlist) through the canonical destination outbox (PR #65).** Launch readiness for the Listings Spreadsheet path is part of the automatic-processing launch condition above and has not been demonstrated end to end with real approved listings.

## Required vs. optional, plainly

Three distinct categories:

**1. Implemented but not yet launch-ready.** Phase I is fully implemented (identity, photo, CommercePackage, claims, drift, review decisions, routing, and all three destination adapters). Phase 0 Slices 1–3 (environment/legacy-access hardening; current-version invariant and regeneration safety; destination-request deduplication) are merged. The remaining launch items in the table above are open.

**2. Required launch work (not yet complete):** the seven open items in the "Implemented vs. launch-ready" table above — automatic workers/retries/queue maintenance; batch processing (owner-design gated); Google Drive → Supabase Storage photo architecture; Shopify draft-only launch verification; Etsy configuration and launch readiness (including the deferred pre-side-effect authorization); dependency-safe legacy cutover; the deterministic/AI/owner recovery ladder.

**3. Genuinely optional enhancements after completion** (each separately authorized; never part of the required finish line): CSV download (backup functionality only); sale and inventory coordination (ROADMAP Milestone 5); public editorial publishing (`PRODUCT_RESET` §17/§18); broader marketing/analytics/automated repricing (ROADMAP "Version 2 and Later"); GMCOM-014 real-export validation.

**Governing completion test (confirmation):** the governing completion test in `PRODUCT_RESET_2026-08-03.md` requires the system to have a **proven working path for each of the three mandatory destination choices**, and completion testing must demonstrate that an **approved canonical CommercePackage can be routed successfully through Shopify, through Etsy, and through the Listings Spreadsheet without Phil or Crystal rewriting or reformatting the approved content**. A listing is **not** required to be sent to all three destinations simultaneously; for each approved listing Phil or Crystal chooses the intended route. If reaching any of the three requires Phil or Crystal to re-key, re-format, or rewrite the approved listing content, the workflow is not complete.

## Owner decisions that gate the remaining launch work

- **Batch processing (Phase 2):** **mandatory owner-design checkpoint** — must not be designed or implemented until Phil approves how it should work.
- **Current-version invariant / regeneration safety:** no new owner decision identified; **fixed and merged** in Phase 0 Slice 2 (PR #70, `9cc5f6a`).
- **Destination-request deduplication:** no new owner decision identified; **fixed and merged** in Phase 0 Slice 3 (PR #71, `851be71`).
- **Automatic workers/retries/queue:** owner decision 3 already requires automatic processing; implementation remains.
- **Legacy cutover:** owner decision — dependency-ordered cutover only, no premature deletion (recorded in `DECISIONS.md`).
- **Photo storage:** owner decision — Google Drive human master → Supabase Storage app copies/derivatives → OneDrive backup/archive (recorded in `DECISIONS.md`).
- **Etsy launch:** owner decision — Etsy stays fail-closed until token store + policy source are configured and verified.
- **Recovery ladder:** owner decision — deterministic automation → constrained AI fresh-eyes recovery → owner escalation.

## Measurable exit criteria

**Phase I is complete when:** every slice in `phase-i-slice-plan.md` is delivered and merged — this is now **true** (I1–I5, CommercePackage creation, content assembly, claims mapping, drift hardening, review decisions, routing, and the three destination adapters).

**Launch readiness (operational launch gate) is met when:**

1. Phase 0 Slice 2 (current-version invariant + regeneration safety) is fixed and merged — **satisfied** (PR #70, `9cc5f6a`).
2. Phase 0 Slice 3 (destination-request deduplication) is fixed and merged — **satisfied** (PR #71, `851be71`).
3. Automatic background processing (workers, retries, queue maintenance, destination creation) is implemented (owner decision 3).
4. Batch processing (Phase 2) is designed per Phil's approval and implemented.
5. Google Drive → Supabase Storage photo architecture is implemented.
6. Shopify draft-only end-to-end launch verification passes (Shopify launches first).
7. Etsy is configured and verified (token store + policy source), the deferred pre-side-effect dispatch authorization is implemented per `gm-commerce/docs/canonical-etsy-draft.md` §10, Etsy end-to-end draft verification passes, and Phil explicitly authorizes activation — only then is Etsy's fail-closed gate lifted.
8. The recovery ladder (deterministic → constrained AI fresh-eyes → owner escalation) is in place.
9. The governing completion test passes: a proven working path for each of the three destination choices with no Phil/Crystal rewriting, and no routine step falls back to Phil or Crystal when the system can handle it.

**GM Commerce (overall) is operationally complete when:** the launch gate above is met, the canonical mirror runs the real pipeline day to day, and the genuinely optional enhancements (CSV download backup, sale/inventory coordination, editorial publishing, V2 items) remain open only if separately authorized.

## What happens after completion

After operational completion there is **no automatic next phase**. The system enters **normal operation** (the real pipeline and the canonical mirror run together), **monitoring** (CI, bridge jobs, dead-letters, export ledger, drift), and **maintenance** (defect fixes, schema care, provider limits). **Enhancements — including sale/inventory coordination, editorial publishing, and V2 features — are made only when the owner separately authorizes them**, each with its own design, implementation, review, and CI verification, per the standing discipline in the Phase H handoff. (Etsy is no longer an enhancement: it is a required operational output via ROADMAP Milestone 4.)

## Estimate of remaining required work before launch

- **Phase 0 Slice 2** (current-version invariant + regeneration safety) — **completed and merged** (PR #70, `9cc5f6a`).
- **Phase 0 Slice 3** (destination-request deduplication) — **completed and merged** (PR #71, `851be71`).
- **Automatic background processing** — one implementation slice (launch requirement, owner decision 3).
- **Batch processing (Phase 2)** — REQUIRED before launch, **gated on Phil's design approval**.
- **Google Drive → Supabase Storage photo architecture** — one implementation slice.
- **Shopify draft-only end-to-end launch verification** — a verification/launch-gate activity, not a new feature.
- **Etsy configuration + verification** — a configuration/verification activity on the implemented adapter.
- **Dependency-safe legacy cutover** — a later, separately sequenced cutover (not Phase 0 Slice 3).
- **Recovery ladder** — part of the automatic-processing/launch hardening work.

The exact slice count can change as the remaining launch work is planned; this is an estimate, not a commitment.
