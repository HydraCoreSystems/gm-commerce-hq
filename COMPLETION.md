# GM Commerce — Master Completion Map

> Plain-English answer to: **What must be finished before GM Commerce is considered operationally complete?**
>
> Owner-facing. Derived strictly from `ROADMAP.md`, `PRODUCT_RESET_2026-08-03.md`, `phase-i-slice-plan.md`, `DECISIONS.md`, and `STATUS.md` — no new capabilities, dates, SourceCategories, Claims, evidence rules, or lifecycle states are invented here. Last updated: 2026-08-09 (Phase I Slice 4 merged via PR #57; owner confirmed the permanent Listings Spreadsheet as the third output path and classified Shopify, Etsy, and the Listings Spreadsheet as mandatory operational outputs — Etsy remains scheduled under ROADMAP M4, now required).

## The finish line in one sentence

GM Commerce is operationally complete when the real product pipeline — select a product, add and approve its photos, generate a listing, review it, publish it to its chosen destination — runs day to day **with a complete canonical mirror**: every real product has permanent identity and photo records in the canonical system, every approved listing becomes a reviewable commerce package in the review shell, and the workflow offers **three mandatory operational destination choices — Shopify, Etsy, and the permanent Listings Spreadsheet — with a proven working path for each** (Phil or Crystal chooses the intended route for each approved listing; a listing is not required to be sent to all three), and no routine step falls back to Phil or Crystal when the system can do it (the governing completion test in `PRODUCT_RESET_2026-08-03.md`).

## What is already finished

**The real pipeline (GMCOM-001–012).** Product selection from Skrybix and the Product SKU Generator, SKU-named photo folders, human photo confirmation, AI listing generation, human review, and Shopify draft publishing — verified end to end against a real Shopify store.

**The canonical system (Phases A–H).** The 18 canonical entity tables, the evidence/claims model, compliance checks, recommendation services, vision analysis, owner-editable policies and learned rules, and the read-only review shell. Phase A reconciled the migration ledger and schema with the live database.

**Phase I — the legacy-to-canonical bridge (I1–I4).**
- **I1** (PR #54): any product that reaches `ready_for_ai` gains canonical ProductConcept/SKU/SourceRecord identity atomically, with a permanent mapping.
- **I2** (PR #55): every real `ready_for_ai` transition now also enqueues a durable bridge job; request-triggered processing, leases, retries, dead-lettering, and an owner-only manual drain.
- **I3** (PR #56): existing products can be backfilled into canonical identity through a required dry-run → single real-run workflow, with an immutable outcome ledger and no duplicates.
- **I4** (PR #57): approved photo sets are bridged into canonical PhotoAssets (details under Phase I below).

## What is true right now (current end state)

- `gm-commerce/main` is at **`e58766e`** (merge of PR #57); post-merge CI green (workflow run **`31307655608`**).
- Every new `ready_for_ai` product automatically gains canonical identity (I1 + I2). Existing products can be backfilled on demand (I3). Every approved photo set gains permanent canonical PhotoAssets with durable per-photo mappings (I4).
- Legacy remains the working source pipeline; canonical approval is never fabricated (canonical PhotoAssets stay `pending`; the legacy approved photo set remains the source of truth).
- **The review shell still has no real CommercePackages to show.** Only a canonical CommercePackage can appear in the Phase H queue, and no slice has created one yet. This is the single biggest remaining gap.

## What must still be finished

All remaining work derives from `phase-i-slice-plan.md` and `PRODUCT_RESET_2026-08-03.md` §7. The smallest likely remaining slices, in dependency order:

| # | Slice (working name) | What it does | Blocked on |
|---|---|---|---|
| 1 | **I5 — historical approved-photo backfill** | Reuse the I4 photo-bridge machinery to backfill photo sets that were approved before I4 existed, using the established dry-run → single-use real-run backfill pattern. **Design recommendation only — not authorized.** | Owner confirmation (no hard design blocker) |
| 2 | **CommercePackage creation** | Create a canonical commerce package at a truthful lifecycle point (e.g., when a listing is finalized or enters review). This is the slice that finally populates the review shell. | Owner decision 10 — creation timing |
| 3 | **CommercePackage content assembly** | Fill the package's content, offers, and per-field lineage. | Content-availability decision + per-field lineage rules (`PRODUCT_RESET` §7) |
| 4 | **Claims mapping for AI-generated listing content** | Map AI-generated listing copy to canonical Claims. | **Owner decision 8 — a defensible SourceCategory/evidence design (the only hard owner-gated blocker)** |
| 5 | **Drift monitoring + operational reconciliation hardening** | Detect when a legacy value changed after it was bridged; alert and reconcile. | No owner decision; engineering design (thresholds/alerting) |

**Optional future enhancements — genuinely optional, not required for operational completion:** sale and inventory coordination (ROADMAP Milestone 5); public editorial publishing (`PRODUCT_RESET` §17/§18); broader marketing/analytics/automated repricing (ROADMAP "Version 2 and Later"); GMCOM-014 real-export validation (needs a current Shopify CSV from Phil).

### Confirmed owner requirement — the permanent Listings Spreadsheet (mandatory third output path)

The owner confirmed (2026-08-09) that the review/publishing workflow must offer **three mandatory operational destination choices — Shopify, Etsy, and Listings Spreadsheet**. **For each approved listing, Phil or Crystal chooses the intended route; a listing is not required to be sent to all three destinations simultaneously.** The third one is **one permanent master Google Sheet**, not a new spreadsheet (or downloadable file) per listing. This replaces any earlier framing of the third output path as a per-listing downloadable/exported file.

- Selecting **Listings Spreadsheet** must **append one new row to the same configured Google Sheet**. It must **not create a new spreadsheet for each listing**.
- The user must be able to identify the **intended external destination** for the row — e.g. **Facebook, Palm Street, Whatnot, auction, website, email, or other**.
- The sheet is **append-only operational output**: existing rows are never overwritten.
- Each row preserves: the canonical **CommercePackage ID and version**, **export ID**, **timestamp**, **trusted actor**, **intended destination**, **approval state**, **listing fields**, **price recommendation**, and **photo references**.
- **Idempotent**: retries or double-clicks must not create duplicate rows. A later **intentional** export of a newer package version may create a new row.
- The **Google Sheet ID and credentials come from trusted server configuration** and never reach the browser.
- Because a Google Sheets write **cannot share a database transaction**, the implementation must use a **durable export job/outbox** with retry, idempotency, failure reporting, and an **immutable export ledger**. The database update and the Google Sheet update are **never one atomic transaction** — that must not be claimed.
- **CSV download is optional backup functionality only**, not the primary requirement.

**Status: confirmed requirement — recorded here, not implemented, and not authorized to implement yet.** It constrains the CommercePackage/output work (phase plan decisions 12–13); no new slice is being invented by recording it.

## Required vs. optional, plainly

Three distinct categories:

**1. Phase I completion requirements** (the legacy-to-canonical bridge): the five slices in the table above — I5 historical approved-photo backfill; CommercePackage creation; CommercePackage content assembly; claims mapping; drift monitoring/reconciliation hardening. Phase I completion does not by itself finish GM Commerce.

**2. Later roadmap work required for overall operational completion** (beyond Phase I):
- **Etsy adapter (ROADMAP Milestone 4) — reclassified as required, not optional.** Shopify, Etsy, and the permanent Listings Spreadsheet are all mandatory operational outputs; Etsy is counted exactly once (here). Etsy remains scheduled under ROADMAP M4, which is now explicitly classified as required for overall GM Commerce completion.
- **The permanent Listings Spreadsheet output path** (confirmed requirement, phase-plan decision 12): append-only, idempotent, trusted-server-config credentials only, written through a durable export job/outbox with an immutable export ledger — never claimed atomic with the database.

**3. Genuinely optional enhancements after completion** (each separately authorized; never part of the required finish line): CSV download (backup functionality only); sale and inventory coordination (ROADMAP Milestone 5); public editorial publishing (`PRODUCT_RESET` §17/§18); broader marketing/analytics/automated repricing (ROADMAP "Version 2 and Later"); GMCOM-014 real-export validation.

**Governing completion test (confirmation):** the governing completion test in `PRODUCT_RESET_2026-08-03.md` requires the system to have a **proven working path for each of the three mandatory destination choices**, and completion testing must demonstrate that an **approved canonical CommercePackage can be routed successfully through Shopify, through Etsy, and through the Listings Spreadsheet without Phil or Crystal rewriting or reformatting the approved content**. A listing is **not** required to be sent to all three destinations simultaneously; for each approved listing Phil or Crystal chooses the intended route. If reaching any of the three requires Phil or Crystal to re-key, re-format, or rewrite the approved listing content, the workflow is not complete.

## Owner decisions that genuinely block each slice

- **I5:** no hard design blocker. Requires owner confirmation that historical photo backfill is wanted, and acceptance of I4's seven decisions as implemented. Ordering already follows the decided production-before-backfill rule (owner decision 6).
- **CommercePackage creation:** **owner decision 10** — when in the lifecycle a package is created.
- **Content assembly:** a content-availability decision and a per-field lineage resolution rule (`PRODUCT_RESET` §7).
- **Claims mapping:** **owner decision 8 — the only hard, owner-gated blocker.** A defensible SourceCategory/evidence design must be approved before any AI-content Claim is created. No new SourceCategory is invented.
- **Drift hardening:** no owner decision; a design decision on thresholds/alerting.

## Measurable exit criteria

**Phase I is complete when:**

1. Every slice in `phase-i-slice-plan.md` is delivered and merged — I1–I5, CommercePackage creation, CommercePackage content assembly, claims mapping, and drift hardening.
2. Every eligible legacy product and every approved photo set has permanent canonical identity/mapping, verified by live-Postgres coverage in CI.
3. The review shell is populated by **real** canonical CommercePackages (not fixtures).
4. Canonical approval is never fabricated, no cross-environment leakage, and CI is green on `gm-commerce/main`.

**GM Commerce (overall) is operationally complete when:**

1. The real pipeline runs end to end with the canonical mirror complete — identity, photo records, CommercePackage, review — and **proven working paths for all three mandatory destination choices: Shopify, Etsy, and the permanent Listings Spreadsheet** (completion testing routes an approved canonical CommercePackage through each without Phil or Crystal rewriting or reformatting the approved content; a listing is not required to be sent to all three).
2. The governing completion test passes: no routine step falls back to Phil or Crystal when the system can handle it.
3. Monitoring and maintenance processes are in place (CI, bridge-job visibility, dead-letter visibility, export-ledger visibility, drift detection).
4. The Phase I exit criteria above and the required roadmap work (Etsy / ROADMAP M4; the Listings Spreadsheet output) are delivered; genuinely optional enhancements (CSV download backup, sale/inventory coordination, editorial publishing, V2 items) remain open only if separately authorized.

## What happens after completion

After operational completion there is **no automatic next phase**. The system enters **normal operation** (the real pipeline and the canonical mirror run together), **monitoring** (CI, bridge jobs, dead-letters, export ledger, drift), and **maintenance** (defect fixes, schema care, provider limits). **Enhancements — including sale/inventory coordination, editorial publishing, and V2 features — are made only when the owner separately authorizes them**, each with its own design, implementation, review, and CI verification, per the standing discipline in the Phase H handoff. (Etsy is no longer an enhancement: it is a required operational output via ROADMAP Milestone 4.)

## Estimate of remaining required implementation slices/work items

- **Phase I:** approximately **5** remaining implementation slices (I5, CommercePackage creation, CommercePackage content assembly, claims mapping, drift monitoring/hardening). This is an **estimate**: the number can shrink if slices are combined and can grow if a slice reveals unexpected schema work (as I3 and I4 did).
- **Required for overall operational completion (beyond Phase I):** approximately **2** remaining work items — the **Etsy adapter (ROADMAP M4, counted exactly once)** and the **permanent Listings Spreadsheet output path** (confirmed requirement, decision 12). The spreadsheet path may fold into the CommercePackage output slice (then ≈1).
- **Total required remaining:** approximately **7** slices/work items (≈6 if the Listings Spreadsheet output folds into the CommercePackage output slice). Clearly an estimate.
- **Genuinely optional (separate from completion, not counted above):** CSV download (backup only), sale/inventory coordination (M5), editorial publishing, V2 marketing/analytics, GMCOM-014 CSV validation.
