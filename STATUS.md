# Current Project Status

_Last updated: 2026-08-02_

## Current Milestone

**GMCOM-009 complete — AI generation is now atomic, versioned, and
runtime-validated. One real gap remains: Phil needs to run the migration
SQL against the live Supabase project before generation works again.**
Both AI calls' output is now runtime-checked (zod) against the canonical
schemas before persistence; concurrent Generate clicks for the same SKU
can produce at most one paid provider call (atomic reserve-before-call,
backed by three new Postgres functions); every regeneration archives the
package it replaces into a new `listing_package_versions` table inside
the same transaction as the new write; and `markReviewed`/
`unlockForRegeneration` are now guarded against acting while a
generation is in flight. Also fixed a real pre-existing bug: regeneration
was unreachable because a product never returned to `ready_for_ai` once
reviewed, so `unlockForRegeneration`'s effect had no way to ever take.
26 new unit tests (first test suite in this repo — added `vitest`), all
passing; `npm run build` clean. **Not yet verified against the real
Supabase project** — Claude has no way to execute the migration itself
this session; full detail, and the exact copy-paste SQL block, in
`handoffs/2026-08-02-GMCOM-009-atomic-versioned-validated-generation.md`.

## Previous Milestone

**GMCOM-008 complete and verified — the Listing Quality Engine is live.**
Listing generation is now a multi-stage pipeline (Gather Facts → Prompt
Builder → AI Generation → Self Review + Revision → Final Listing
Package), still behind the same AI Provider abstraction. Verified against
a real product: `HY-LOB01-C04` ("Hoya lobbii") now has a real, genuinely
useful Listing Package + Quality Summary sitting in `/review`, awaiting
Phil or Crystal's actual review.

## Completed

- **GMCOM-009 — AI generation hardened: atomic (reserve/finalize/release
  locking, ≤1 paid call under concurrent Generate clicks), versioned
  (`listing_package_versions`, append-only), and runtime-validated (zod
  schemas reject malformed/invalid provider output before persistence).
  Also fixed regeneration being unreachable in the old code. Commit
  `81a5401` on `main`, pushed. 26 tests, `npm run build` clean. SQL
  migration not yet applied to the live project — see handoff.**
- GM Commerce Headquarters repository created.
- Core mission and Version 1 direction established.
- Existing Skrybix repository identified: `HydraCoreSystems/skrybix-webapp`.
- Product SKU Generator built by Claude and documented.
- Product SKU Generator repository location recorded and pushed:
  `HydraCoreSystems/product-sku-generator`.
- GMCOM-001 baseline documented. Milestone 0 real-world validation
  completed by Phil.
- **GMCOM-002 — GM Commerce application foundation built.**
- New Supabase project provisioned for GM Commerce under a co-owner's
  separate account (project ref `wcrcllhvgbhykbonopzx`).
- **GMCOM-003 — merged.** Skrybix commerce selection handoff, production-
  deployed.
- **GMCOM-004 — verified end-to-end with a real cutting** (`HY-LOB01-C04`).
- **GMCOM-006 — guided intake experience built and verified.**
  `HY-LOB01-C04` has had its photo folder and photos confirmed since this
  milestone.
- **GMCOM-007 — canonical AI Listing Package generator + Provider
  abstraction, fully verified including a real OpenAI call.** Vendor-
  agnostic architecture: `GM Commerce → Listing Generator → Prompt
  Builder → AI Provider interface → {OpenAI, Anthropic, Mock}`. Anthropic
  provider complete but intentionally untested per instruction.
- **GMCOM-008 — Listing Quality Engine (Phase 1), commit `827a3ac` on
  `main`, pushed.** Single-prompt generation replaced with a multi-stage
  pipeline: Gather Facts (new — now includes real photo-folder metadata,
  `photoCount`) → Prompt Builder → AI Generation → Self Review + Revision
  (combined into one API call for cost efficiency, per explicit
  instruction — 2 total AI calls per generation, not 3) → Final Listing
  Package. New `listing_packages.quality_summary` column (confidence,
  unsupported claims caught, missing information, recommendations),
  surfaced directly to the human reviewer in `/review`. Verified twice:
  pipeline logic with the Mock provider (updated to return matching
  shapes per pipeline stage) against a scratch record, then a real
  generation against `HY-LOB01-C04` via OpenAI — confidence came back
  honestly "medium," zero unsupported claims, and specific actionable
  guidance (price, dimensions, rooted status, "use the available photos
  to confirm condition"). Real record deliberately left at "Needs
  review," not approved by Claude. Full detail in
  `handoffs/2026-08-02-GMCOM-008-listing-quality-engine.md`.
- **Found and fixed a real bug in this repository** (prior session):
  `DECISIONS.md` had unresolved git conflict markers from an incomplete
  stash-pop; merged cleanly.

## Active Work

- **Phil to run the GMCOM-009 migration SQL against the live Supabase
  project** (copy-paste block in
  `handoffs/2026-08-02-GMCOM-009-atomic-versioned-validated-generation.md`)
  — generation is blocked in the real app until this runs.
- Phil or Crystal to actually review `HY-LOB01-C04`'s real Listing
  Package in `/review` — edit, approve, or discard as they see fit.
- Update GitHub Issues for GMCOM-007/008/009 — no GitHub API access
  available to Claude in this environment.
- Generate at least one real SKU through the Product SKU Generator's live
  app — its `sku-log.json` was still empty as of the last check.

## Current Blockers

- No GitHub API access for updating Issues directly.
- No live channel between Claude and ChatGPT exists yet; coordination
  currently depends on Phil relaying between them.
- AI provider usage limits (Anthropic/OpenAI/etc.) are not automatically
  visible to the project manager.

## Next Highest-Priority Task

**GMCOM-010 — Wire the Listing Package into a real Shopify draft** (per
the existing "Shopify before Etsy" decision in `DECISIONS.md`). The
reliability hardening GMCOM-009 explicitly asked to complete before
Shopify work is done — this is the clear next gap: read from a *reviewed*
package, create a real Shopify draft product, and store the returned
Shopify product ID back on the record. Does not strictly require the
GMCOM-009 migration to be applied first (publishing only reads an
already-reviewed package), but Phil should run that migration soon
regardless since generation itself is blocked until he does.

## AI Capacity

| Contributor | Current role | Capacity status | Current assignment |
|---|---|---|---|
| ChatGPT | Project manager / reviewer / contributor | Available | Sequence Shopify draft publishing issue (GMCOM-010) |
| Claude | Primary hands-on builder | Available | GMCOM-009 complete; awaiting next assignment |
| GitHub Copilot | GitHub-native implementation contributor | Available | Completed GMCOM-003, GMCOM-005 |
| Phil | Product owner / workflow validator | Available as schedule permits | Run GMCOM-009 migration SQL; review `HY-LOB01-C04`'s real Listing Package |

Capacity status should be updated whenever a provider limit is reached or resets.
