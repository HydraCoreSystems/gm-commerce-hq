# GM Commerce Working Release Standard

## Governing objective

Get a complete, usable GM Commerce workflow into Gathering Moss operations as quickly as possible, then improve it continuously from real use.

This does **not** authorize a half-built demo. A working release must complete the essential production path safely and honestly.

## Essential production path

1. Select an existing SKU from Skrybix or Product SKU Generator.
2. Create or confirm its photo folder.
3. Add raw product photos.
4. Standardize, optimize, review, and approve commerce images.
5. Generate a truthful, high-quality canonical Listing Package.
6. Review and edit the Listing Package.
7. Produce at least one usable sales output:
   - copy-ready Listing Sheet for manual channels, and
   - Shopify draft publishing as the first automated marketplace adapter.

A release is not considered operational until a real product can traverse this path without manual SKU re-entry or off-system reconstruction of listing content.

## Priority test for every proposed task

Before starting work, answer:

> Does this directly shorten the path to a usable production workflow, remove a blocker from that path, or prevent a realistic production failure?

If **yes**, prioritize it.

If **no**, place it behind the current working-release path unless Phil explicitly promotes it.

## Delivery policy

- Ship complete vertical slices rather than isolated foundations.
- Preserve truthfulness, source ownership, originals, human approval, and data integrity.
- Prefer the simplest implementation that can safely support real use.
- Defer nonessential scoring systems, generalized platforms, speculative automation, and polish that does not block use.
- Refine from observed real-world use after the workflow is operational.

## Team utilization rule

A limit or delay affecting one contributor must not stop the project.

- Claude: major implementation and complex UI/workflow integration.
- Copilot: independent engineering modules, tests, migrations, tooling, reviews, and hardening.
- ChatGPT: project management, product decisions, specifications, GitHub work, independent code/artifacts, acceptance criteria, and integration planning.
- Phil and Crystal: business and brand decisions that require Gathering Moss ownership.

Every active contributor must have a non-overlapping deliverable whenever useful work is available.

## Decision discipline

- Existing recorded decisions remain in force until Phil changes them or implementation evidence proves they cannot work.
- Engineering choices may be made by the assigned engineer inside issue boundaries.
- Business choices require Phil's decision after concise options and tradeoffs.
- Brand choices require Phil or Crystal's decision.
- Do not reopen settled choices as brainstorming.

## Output discipline

Project work should produce visible artifacts: code, tests, migrations, specifications, issues, pull requests, screenshots, verification reports, or usable output.

Discussion is supporting work, not the deliverable.

## Current working-release sequence

1. GMCOM-011 — Commerce image preparation and approval.
2. GMCOM-012 — Copy-ready Listing Sheet output.
3. Real internal use by Phil and Crystal.
4. Shopify draft adapter.
5. Refinement driven by production experience.
