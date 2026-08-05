# Handoff — GMCOM-002: Build the GM Commerce application foundation

## Task

- Issue: GMCOM-002 (https://github.com/HydraCoreSystems/gm-commerce-hq/issues/2)
- Objective: Build the first working version of GM Commerce — the
  application that ties Skrybix and the Product SKU Generator together
  around a select → intake → review → publish workflow, per the revised
  GMCOM-002 direction (the original issue asked for a bulk-import display;
  Phil relayed a revision from ChatGPT mid-session replacing that with a
  human-driven selection pipeline — see "Decisions Made" below).
- Repository: `HydraCoreSystems/gm-commerce` (new — application code lives
  here, not in `gm-commerce-hq`, per instruction). **Not yet pushed to
  GitHub** — see Blockers.
- Branch: `main`, one commit so far ("Initial commit: GM Commerce
  application foundation (GMCOM-002)").

## Work Completed

Built a Next.js 14 (App Router) + TypeScript + Supabase/Postgres app at
`C:\Users\pwach\OneDrive\Documents\GitHub\gm-commerce`, following the same
stack and conventions as `skrybix-webapp` (plain CSS with the shared
Gathering Moss palette, server-only Supabase client via service-role key,
no ORM).

**Central product model** (`supabase/schema.sql`): one `products` table,
keyed by `sku`, covering both source types (`source_system` /
`source_type`), the pipeline stage (`status`: intake → review → published
→ archived), photo folder path, notes, AI-stub listing fields
(title/description/tags), and marketplace fields (target marketplaces,
marketplace IDs as jsonb). Real table created in the project's own new
Supabase project (`wcrcllhvgbhykbonopzx`, a separate Supabase account
under a co-owner's email, per Phil's explicit choice to keep it out of his
personal account).

**Workflow implemented end-to-end, matching the revised spec exactly:**

1. **`/select`** — lists products available from Skrybix
   (`lib/sources/skrybix.ts`, calls the existing authenticated
   `sku-registry` endpoint — never touches Skrybix's database) and from
   the Product SKU Generator (`lib/sources/sku-generator.ts`, reads its
   local `data/sku-log.json` directly — it has no network API). Each row
   not already in GM Commerce gets a **Select for Commerce** button.
   **There is no text field anywhere in this app that accepts a
   manually-typed SKU** — the only way a SKU enters the `products` table
   is via this selection action, which always copies a SKU that already
   exists in one of the two source systems.
2. **`/pipeline`** (intake) — for each selected-but-not-yet-processed
   product: a **Confirm / Create Photo Folder** action
   (`lib/photo-root.ts` + `app/pipeline/actions.ts`) that verifies the
   Product SKU Generator's already-known folder for non-plant products, or
   creates a fresh one (named by SKU) under the same shared photo root for
   Skrybix-origin products, which have no folder concept at all today. A
   notes field. A **Process** button, disabled until the folder step is
   done, that runs the AI stub and moves the product to Review.
3. **`/review`** — shows the (clearly-labeled) stub listing title/
   description/tags, marketplace checkboxes, and a **Publish (stub)**
   button that records which marketplaces were chosen and a placeholder
   ID for each (`STUB-SHOPIFY-<sku>` etc.) — no real Shopify/Etsy call.
4. **`/`** — dashboard with pipeline-stage counts and a plain-English
   walkthrough of the workflow.

**Stubs, clearly isolated and labeled** (per "use placeholder
implementations... those modules can initially be stubs"):
`lib/stub-ai.ts` (listing generation), `lib/stub-marketplaces.ts`
(publishing). Both return obviously-fake content (`[STUB TITLE] ...`,
`STUB-SHOPIFY-...`) so nothing could be mistaken for real output.

## Verification

**Confirmed working (tested, not just written):**

- `npm run build` compiles clean, zero type errors, after every code
  change.
- Full workflow click-tested in a real browser against the real Supabase
  project: selected a product, confirmed its photo folder (real
  filesystem check/creation), saved notes, clicked Process (real stub AI
  content generated and persisted), published from the Review Queue with
  a marketplace chosen (real stub marketplace IDs persisted), and
  confirmed the dashboard counts updated correctly at every stage. No
  server errors at any step.
- The empty/unconfigured real state was also verified deliberately: with
  `SKU_REGISTRY_KEY` unset and the Product SKU Generator's real
  `sku-log.json` genuinely empty, `/select` correctly shows "not
  configured" for Skrybix and "no SKUs generated yet" for the SKU
  Generator, rather than erroring or showing fake data.
- Test data used for the click-through above was synthetic and lived only
  in a scratch temp directory (never in the real Product SKU Generator's
  data files) and was deleted from the real Supabase table afterward — the
  live database is currently empty, as it honestly should be.

**Not yet tested / genuinely unknown:**

- **No real Skrybix data has been ingested or displayed.**
  `SKU_REGISTRY_KEY` is not set anywhere I have access to (checked
  `skrybix-webapp/.env.local` directly — it's blank there too), so the
  Skrybix half of `/select` has never shown a real plant SKU. This also
  means Copilot's planned Skrybix-side "selection handoff" has no
  integration point to test against yet — that's a real dependency, not
  just a config gap.
- **The Product SKU Generator currently has zero real logged SKUs** in
  either the `Documents\GitHub` or `Desktop` copies (`sku-log.json` is
  `[]` in both, counters unchanged since the GMCOM-001 baseline) — worth
  flagging since Phil indicated real-world validation was done; I didn't
  find evidence of it in either copy's log. Not treating this as
  "broken," just noting the discrepancy for whoever picks this up next.
- Only one product has ever gone through the pipeline (the deleted test
  row) — never tested with more than one product in flight at once, or
  with a Skrybix-origin (mother/cutting) product specifically (no real one
  was available to select).
- No automated tests exist — same posture as the SKU Generator, manual
  verification only.

## Decisions Made

- **GMCOM-002 was substantially revised mid-session.** The original issue
  text asked for a read-only display seeded by a bulk import script. Phil
  relayed a written revision from ChatGPT replacing that with the
  select → intake → review → publish workflow this handoff describes,
  specifically to guarantee "GM Commerce must never require manual SKU
  entry." Recommend the project manager reconcile the GitHub Issue's
  visible text with this revision so future sessions don't start from the
  stale version.
- **New Supabase project, new account.** Phil created a fresh Supabase
  account under a co-owner's email specifically so GM Commerce's database
  isn't lumped in with his personal Supabase usage. Project ref
  `wcrcllhvgbhykbonopzx`. Worth recording since it's a durable fact about
  where this data lives.
- **Photo folders for Skrybix-origin products** are created under the same
  shared root the Product SKU Generator already writes into (read from its
  `data/config.json`), not a separate GM-Commerce-only root — reduces
  proliferation of photo locations. Overridable via
  `GM_COMMERCE_PHOTO_ROOT` if that ever needs to diverge.
- **The bulk-import script from the original GMCOM-002 approach was
  deleted**, not kept alongside the new workflow — it modeled a different,
  now-superseded interaction (import everything vs. select specific
  items), and keeping both would have meant two ways to get a SKU into the
  table.
- Confirmed with Phil directly: the "manual SKU keying" concern was about
  ensuring GM Commerce itself never has a type-in-a-SKU field, not about
  Skrybix's own (separately flagged) manual Mother_ID entry. This app has
  no such field anywhere.

## Remaining Work

To close GMCOM-002's acceptance criteria fully:

- **Push this repo to GitHub.** It's committed locally
  (`C:\Users\pwach\OneDrive\Documents\GitHub\gm-commerce`, branch `main`)
  but I have no GitHub push credentials in this environment — Phil needs
  to create `HydraCoreSystems/gm-commerce` (private, matching
  `gm-commerce-hq`'s posture) and push via GitHub Desktop.
- Set `SKU_REGISTRY_KEY` in Skrybix's own environment (both
  `skrybix-webapp/.env.local` and its Vercel config) and share the value
  into `gm-commerce/.env.local`, so `/select` can show real plant SKUs
  and this can get tested against real Skrybix data for the first time.
- Generate at least one real SKU through the Product SKU Generator's own
  live app (not by hand-editing its JSON files) so `/select` has real
  non-plant data to test against too.

## Blockers or Risks

- No GitHub API/push credentials in this environment (same standing
  blocker as GMCOM-001) — this repo needs manual creation + push via
  GitHub Desktop before it's visible to Copilot or ChatGPT.
- Copilot's Skrybix-side "selection handoff" endpoint doesn't exist yet —
  `lib/sources/skrybix.ts` currently reads the existing full
  `sku-registry` feed and treats "not yet in GM Commerce's table" as the
  proxy for "available." That's a reasonable stand-in, but once Copilot's
  real selection-scoped endpoint exists, this file is where to swap it in
  — flagging so it isn't mistaken for the final integration.
- Next.js 14.2.35 has known unpatched CVEs (same deferred, shared risk
  already noted for `skrybix-webapp` and `hydrocloud-webapp` — not
  addressed here either, consistent with treating it as an accepted,
  tracked risk rather than blocking this issue on a major-version
  upgrade).

## Recommended Next Action

**GMCOM-003 — Wire up the real Skrybix selection integration.** Once
Copilot's selection-handoff endpoint exists on the Skrybix side: set
`SKU_REGISTRY_KEY` end-to-end, point `lib/sources/skrybix.ts` at whatever
the real endpoint turns out to be (a filtered `sku-registry` or a new
route), and run the full select → intake → review → publish workflow
against at least one real plant product and one real non-plant product, to
close the "real Skrybix data has never been tested" gap called out above.
This is a small, concrete, testable next step — not a new milestone — and
should be sequenced before any real AI or Shopify/Etsy work, since neither
of those is useful until real products can actually reach the pipeline.
