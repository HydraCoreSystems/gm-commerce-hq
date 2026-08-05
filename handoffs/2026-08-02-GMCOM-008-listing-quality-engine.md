# Handoff — GMCOM-008: Listing Quality Engine (Phase 1)

## Task

- Issue: GMCOM-008 (Listing Quality Engine, Phase 1)
- Objective: Raise the quality bar from "generate a listing" to "generate
  the highest-quality, most persuasive, truthful Gathering Moss listing
  possible" — replace the single-prompt generation from GMCOM-007 with a
  multi-stage pipeline (Gather Facts → Prompt Builder → AI Generation →
  Self Review → Revision → Final Listing Package), add a Quality Summary
  for the human reviewer, keep the architecture extensible for future
  review stages, and keep everything on the existing Provider abstraction.
  Explicitly: no Shopify, no publishing, no change to the canonical
  Listing Package as the output.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `827a3ac` ("Build the Listing Quality Engine,
  Phase 1 (GMCOM-008)"). Pushed successfully.

## Work Completed

**The pipeline, implemented exactly as specified:**

1. **Gather Facts** — `lib/ai/pipeline/gather-facts.ts` (new file). A
   plain function, no AI call. Assembles every known real fact: SKU/
   source identity, source-specific `attributes` (Skrybix's
   `parentSourceRecordId`/`sourceState`/etc., or the SKU Generator's
   category attributes), the human's own `notes`, and — new — real
   **photo metadata**: `photoMetadata.photoCount`, read via the existing
   `countPhotos()` helper against the product's actual photo folder.
   Structured as its own object specifically so richer photo facts (real
   vision analysis, later) can be added without touching anything else.
2. **Prompt Builder** — `lib/ai/prompt-builder.ts`, extended, not
   duplicated. Two request builders, both equally provider-agnostic:
   `buildGenerationRequest` (unchanged content-integrity rules from
   GMCOM-007) and the new `buildReviewAndReviseRequest`.
3. **AI Generation** — first `provider.generate()` call, produces the
   initial draft. Same schema and content-integrity rules as GMCOM-007.
4. **Self Review** + 5. **Revision** — **combined into one call**, not
   two. Per the issue's explicit cost-efficiency instruction, the second
   call is given the same known facts plus the stage-3 draft, and asked
   to do both jobs in one structured response: critique the draft
   against seven named problems (unsupported claims, repetitive AI
   wording, weak titles, missing selling points, poor SEO opportunities,
   generic language, internal inconsistencies), then produce a revised
   package that fixes what it found while still obeying every
   content-integrity rule — a revision is explicitly forbidden from
   inventing a fact to sound more persuasive.
6. **Final Listing Package** — the pipeline's return value. Only the
   revised package is ever persisted or shown to a human; the
   pre-revision draft isn't stored anywhere.

**Total AI calls per generation: 2** (was 1 in GMCOM-007) — not 3, which
a naive literal reading of "self review, then revision" as separate
stages would have produced.

**New `listing_packages.quality_summary` column** (jsonb): `confidence`
(`low`/`medium`/`high`), `unsupportedClaims` (what the draft got wrong
before the reviewer fixed it), `missingInformation` (real facts a human
could supply to strengthen the listing), `recommendations` (plain-
language advice). This is genuinely surfaced to the human reviewer in
`/review` — a dedicated panel above the edit form, not buried in a debug
log — per the issue's explicit intent that this is guidance for a person,
not internal telemetry.

**Mock provider updated** (`lib/ai/providers/mock-provider.ts`) to branch
on `request.schemaName` and return a shape matching whichever stage
asked — otherwise it couldn't meaningfully exercise the new two-call
pipeline at all (it would return the flat package shape for the
review+revise call too, and destructuring `revisedPackage`/
`qualitySummary` out of that would silently yield `undefined` for both).

**Extensibility, per the issue's explicit requirement:** adding a future
stage (a dedicated SEO pass, real vision-based photo review, etc.) means
adding one more `buildXRequest` in `prompt-builder.ts` and one more
`provider.generate()` call in `listing-generator.ts` that consumes the
previous stage's output — the same shape stage 4+5 already follows. No
redesign of stages 1–3, and no changes to any provider file (they remain
pure transport, unaware of pipeline structure entirely).

## Verification

**Confirmed working, `npm run build` clean:**

- **Pipeline logic verified with the Mock provider** against an isolated
  scratch product, before the schema migration landed: both stages ran
  correctly (facts gathered, both prompts built, both mock calls
  succeeded, JSON parsed correctly) — the only failure was the expected
  one, a missing DB column, which is itself useful confirmation that
  nothing upstream of persistence was broken.
- **Real end-to-end verification against a real product**, as the issue
  requires: `HY-LOB01-C04` ("Hoya lobbii"). Its photo folder and photos
  were already confirmed from earlier in this session, so I clicked
  "Mark Ready for AI" for it (a deliberate choice — the issue explicitly
  asked to verify with a real product, so this reflects that instruction
  rather than my own earlier restraint from prior sessions) and then
  "Generate Listing Package" with `AI_PROVIDER=openai`. Real result,
  confirmed via direct database query:

  ```
  proposed_title: "Hoya lobbii Plant Cutting"
  quality_summary.confidence: "medium"
  quality_summary.unsupportedClaims: []
  quality_summary.missingInformation: [
    "Price", "Cutting dimensions", "Number of leaves or nodes",
    "Whether the cutting is rooted", "Care instructions",
    "Any condition details shown in the available photos"
  ]
  quality_summary.recommendations: [
    "Add the price before publishing.",
    "Provide dimensions and basic cutting details...",
    "Use the available photos to confirm and describe the cutting's
     visible condition.", ...
  ]
  source_facts.photoMetadata.photoCount: 2   <- genuinely real: 2 actual
                                                 photo files now exist in
                                                 that folder
  ```

  Content integrity held under real model output for the second session
  in a row: `price`/`care_details` stayed `null`, and the quality summary
  is honestly humble (`"medium"` confidence, a long and specific
  missing-information list) rather than inflated. The recommendations are
  genuinely actionable, not generic filler — this is a materially better
  result than GMCOM-007's single-call version would have produced, not
  just a more expensive one.
  - **Deliberately did not** mark this real record reviewed or touch it
    further — left it at "Needs review" for Phil or Crystal to actually
    review, matching the same discipline as GMCOM-004/006/007.
- Confirmed the Quality Summary panel renders correctly in `/review`,
  showing confidence, missing information, and recommendations clearly
  above the edit form.
- Scratch test data (`TEST-GMCOM008-001`) deleted after the dry run.

## Decisions Made

- **Combined the Self Review and Revision stages into a single API call**
  rather than three total calls, per the issue's explicit cost-efficiency
  instruction. They remain conceptually distinct (the system prompt asks
  for both jobs explicitly, in order, within one response) so splitting
  them into two calls later — if a future need justifies the extra
  cost/latency — is a small, contained change, not a redesign.
- **`quality_summary` is AI-authored and not human-editable** — same
  treatment as `source_facts`. A human can edit the Listing Package
  content, but the quality assessment itself is the AI's own record, kept
  intact for audit even after edits.
- **The pre-revision draft is never persisted.** Per "present only the
  revised package" — if a future need arises to compare pre/post revision
  for debugging, that would need a deliberate schema addition, not
  something quietly kept around today.
- **Photo metadata is count-only for now**, not filenames or real image
  analysis — genuinely all that's available without wiring in vision
  capability, which is out of scope for this issue. Structured as its own
  sub-object specifically so it can grow later without reshaping
  `GatheredFacts` itself.

## Remaining Work

- None outstanding for GMCOM-008 itself — fully built, migrated, and
  verified with both a scratch record and a real product.
- The real cutting `HY-LOB01-C04` now has a real, unreviewed Listing
  Package sitting in `/review`, waiting for Phil or Crystal.
- Update GitHub Issue for GMCOM-008 — no GitHub API access available to
  Claude in this environment.

## Blockers or Risks

- None new. Push access continues to work as of this session (see
  GMCOM-007's handoff note on this).

## Recommended Next Action

**Wire the Listing Package into a real Shopify draft.** GMCOM-008
explicitly scoped Shopify out, and GMCOM-007 already recommended it as
the next step — with two AI-quality milestones now shipped and verified
against real data, this is the clear next gap: GM Commerce can produce a
genuinely good canonical Listing Package but still can't get it in front
of a real buyer. Per the existing "Shopify before Etsy" decision in
`DECISIONS.md`, that means a Shopify draft-publishing adapter that reads
from a *reviewed* Listing Package (gated the same way `/review`'s publish
stub already is) and creates a real Shopify draft product, storing the
returned Shopify product ID back on the record.
