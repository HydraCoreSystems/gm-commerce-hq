# Handoff — GMCOM-011: Photo preparation and approval pipeline

## Task

- Issue: GMCOM-011 — [HydraCoreSystems/gm-commerce-hq#10](https://github.com/HydraCoreSystems/gm-commerce-hq/issues/10)
- Objective: Build the Version 1 GM Commerce photo pipeline so raw product
  uploads become truthful, standardized, optimized, human-approved
  commerce assets before any Shopify/Etsy draft is created.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `9ee8eec` ("Build the photo preparation and
  approval pipeline (GMCOM-011)"). Pushed successfully.
- Preceded by a live smoke test of GMCOM-009's generation path (see below)
  and confirmation that GMCOM-010's reconciliation (migrations, CI,
  `.nvmrc`, `lib/server-input.ts`/`lib/server-logger.ts`) landed cleanly —
  both required by the assignment before starting this issue.

## Part 1 — GMCOM-009 live smoke test (prerequisite, done first)

Ran a real generation against `HY-LOB01-C04` on the reconciled `main`
(commit `9dc82d8`), calling the actual exported `reserveGenerationSlot` /
`generateListingPackage` / `finalizeGeneration` functions directly (real
Supabase project, real OpenAI call) rather than through the UI, since
regeneration still has no UI entry point (a known, previously-disclosed
limitation). All required checks passed:

- Reservation: `ok: true`, `reason: "reserved"`, real UUID lock token,
  `previousStatus: "review"`.
- Real OpenAI call succeeded (`gpt-5.6-luna`), returned a genuine,
  validated quality summary.
- Finalization succeeded; product reached `status: "review"`.
- Version incremented 1 → 2; the old v1 row was archived into
  `listing_package_versions` with its original `generated_at` preserved.
- `quality_summary` and `source_facts` persisted correctly on the new row.
- No lock remained afterward (`generation_started_at`/
  `generation_lock_token` both null).
- `/review` reloads with the correct v2 content — confirmed via an
  **isolated** dev server copy, not the one already running in this
  checkout.

**One real, non-code finding:** my first check of `/review` (via the
checkout's own dev server) showed *stale v1 content* even though the
database genuinely had v2. Root cause: another chat's dev server was
already running against this same folder, and two concurrent `next dev`
processes sharing one `.next` build directory corrupted each other's
output (the other server was independently returning a broken 500 at the
time, referencing production runtime files under `next dev` — a symptom
of the same corruption). Not a GMCOM-009/010 bug. Re-verified cleanly
against a fully isolated copy (own `node_modules`, own `.next`, own port).
Worth knowing: **running two `next dev` instances against the same
checked-out folder is not safe** — flagging for whoever owns operational
conventions, not fixing myself (out of scope for GMCOM-011).

Never called `markReviewed` — `HY-LOB01-C04` is exactly where GMCOM-008
left it, "Needs review," just on v2 now with v1 fully recoverable.

## Part 2 — GMCOM-011 photo pipeline

### Work completed

**Provider-neutral processing interface** (`lib/photos/types.ts`,
`processor-registry.ts`, `processors/local-processor.ts`) — the exact
same architectural pattern as the AI Provider abstraction (GMCOM-007):
one interface (`ImageProcessor`), one registry keyed by an env var
(`PHOTO_PROCESSOR`, default and only current value `local`), one
implementation file that's the sole place `sharp`/`heic-convert` are
imported. A future cloud image service or AI-vision-assisted processor
is a new class here, not a rewrite of the pipeline or server actions.

**HEIC/HEIF — no manual conversion required.** `heic-convert` (backed by
`libheif-js`, libheif compiled to WASM) decodes HEIC/HEIF into a real
intermediate that `sharp` then processes identically to any other source.
Chosen specifically because sharp's own prebuilt binaries exclude HEIF
support (licensing) and `heic-convert` needs no native build step on
Phil's Windows machine or in CI — the alternative (a custom libvips build
with libheif) would have traded a user-side conversion step for an
operator-side build dependency, which the issue's own requirement ("never
require the user to rename or pre-convert the file") rules out just as
much.

**Deterministic inspection, no ML model** (`lib/photos/image-
analysis.ts`): EXIF orientation + sRGB color-profile normalization,
Laplacian-variance blur estimate, exposure mean via `sharp`'s own
`.stats()`, and a 64-bit average-hash for near-duplicate flagging
(Hamming distance, first-seen-wins). All well-established, explainable
techniques — matching Version 1's explicit preference for conventional
deterministic processing over generative/ML methods.

**"Standardized Gathering Moss commerce backgrounds" = a mat, not a
background swap.** The `square_marketplace` derivative pads a non-square
photo onto a solid, configurable brand-color canvas
(`derivative-standard.ts`, default warm cream `#F6F4EE`) — the real
background actually behind the product in the photo is never touched,
segmented, or replaced. True background replacement is exactly the
generative/object-removal category GMCOM-011's own truthfulness
guardrails require to be "explicit, previewed, and human-approved" and
"kept out of the default Version 1 path" — this reading is deliberate and
explained in `local-processor.ts`'s header comment. A
`detail_uncropped` derivative is produced only when the source aspect
ratio is far enough from square (configurable tolerance) that a square
crop would genuinely lose product information, per the issue's own
conditional wording.

**Alt text reuses the existing AI Provider abstraction**
(`lib/photos/alt-text.ts`) rather than introducing new AI infrastructure —
one small, batched (one call per photo set, not per image), factual,
runtime-validated (zod, same `parseAndValidate` used for Listing
Packages) request, mock by default so nothing paid runs unless
`AI_PROVIDER` is already configured. Text-only, not vision: the model is
given real product facts (name, source type/category, hero-vs-detail
role) but never the pixels, so it can't invent visual details it never
saw — a deliberate Version 1 boundary, not an oversight (the scope doc
itself frames vision as a later addition).

**Schema** (`supabase/migrations/20260802020000_photo_pipeline.sql`,
appended to `schema.sql`): `photo_sets` (one row per SKU, the aggregate
approval state — `pending` → `processing` → `needs_review` → `approved`),
`photo_assets` (one row per discovered original — never mutated, only its
inspection metadata recorded), `photo_derivatives` (one row per generated
derivative — hero/order/alt-text live here since those are properties of
what gets published, not of the raw upload). Concurrency uses plain
guarded conditional updates (`.eq("status", expected)`), not GMCOM-009's
full reserve/finalize/release token machinery — deliberately lighter,
since photo processing is local and unpaid (the real risk GMCOM-009
guarded against was duplicate *paid* AI calls; here the risk is wasted
CPU and confusing UI state, which a guarded conditional update already
prevents). Approval requires a hero image and non-empty alt text on every
derivative; once approved, `photo_sets.status` requires an explicit
unlock to reprocess — mirroring `listing_packages.reviewed_at`
deliberately, for the same reason.

**Derivatives never share a folder with originals.** They're written to
`_derivatives/<sku>/`, a sibling of the SKU's own photo folder (never a
subfolder of it), so a plain top-level, non-recursive folder scan can
never mistake generated output for a newly-discovered source photo.

**UI** (`/photos` index, `/photos/[sku]` guided review, plus a
non-blocking "Prepare Photos" link added to the home page's intake
checklist — doesn't change what "Mark Ready for AI" requires): discovered
photos with warnings/duplicate flags, before/after previews (HEIC/HEIF
originals show a labeled placeholder instead of an unrenderable inline
image — most browsers can't display HEIC directly; the derivative is the
real proof decoding worked), a hero/order picker (number inputs + radio
buttons, not drag-and-drop — a deliberate simplification: this app has no
client-side interactive components yet, and "choose/reorder" doesn't
require drag specifically), per-derivative alt-text editing, and
approval/unlock. Images are served through `app/photos/image/route.ts`,
which only ever accepts an opaque row id and looks up the real path from
the database — no request ever supplies a filesystem path, so there's no
path-traversal surface to defend against in the first place.

### Verification

**Automated — 60 tests total (34 new for GMCOM-011), `npm run typecheck`
and `npm run build` both clean:**
- `lib/photos/image-analysis.test.ts` — perceptual hash stability/
  distinctness, Hamming distance correctness, sharpness scoring.
- `lib/photos/discovery.test.ts` — format classification, OS-junk
  skipping, non-recursive scanning (derivatives folder never picked up as
  a source), stable content hashing.
- `lib/photos/pipeline.test.ts` — near-duplicate flagging logic.
- `lib/photos/processors/local-processor.test.ts` — decode pass-through
  for JPEG/PNG/WebP, unsupported-format rejection, dimension/hash
  reporting, underexposure warning, square derivative sizing, the
  aspect-ratio-conditional `detail_uncropped` rule, no-upscaling
  guarantee.
- `app/photos/actions.test.ts` — the approval gate's business rules
  (requires a hero, requires alt text on every derivative, requires at
  least one derivative) and the unlock state-machine guard.

**Real, not just written — file-level verification against real files**
(Supabase persistence for this part is the one remaining gap; see below):
- **Real plant photo set**: both of `HY-LOB01-C04`'s actual JPEGs
  (4000×3000 and 2202×1658) — discovered, inspected (blur/exposure
  computed, zero warnings), derivative generated and written to a real
  sibling folder, originals confirmed byte-for-byte unchanged
  (SHA-256 before/after), and the two genuinely different photos
  correctly *not* flagged as near-duplicates.
- **Real HEIC/HEIF source**: a real iPhone HEIC photo (3024×4032),
  processed purely mechanically — decoded via `heic-convert+sharp`
  without any manual conversion, dimensions correctly detected, both a
  `square_marketplace` and (aspect ratio 0.75, past the tolerance) a
  `detail_uncropped` derivative produced as real, viewable JPEGs, original
  confirmed byte-for-byte unchanged. Its depicted content was never
  examined, described, or logged anywhere — only technical facts.
- **Background mat, confirmed pixel-level**: a synthetic wide (800×200,
  saturated green) test image produced a square derivative where the
  letterboxed corners are the configured near-neutral cream mat and the
  center remains the original, fully-saturated, uncropped content —
  concretely proving this is a frame around the untouched photo, not a
  background replacement.
- All scratch test files/folders were copies in an isolated temp
  directory, never touching `HY-LOB01-C04`'s real photo folder or the
  original personal HEIC file, and were deleted after the run.

**Not yet verified — the one real gap, same shape as GMCOM-009's:**
Claude has no way to execute DDL against the live Supabase project in
this environment. The migration below has **not** been applied, so:
- No Supabase-persisted verification (`photo_sets`/`photo_assets`/
  `photo_derivatives` rows, the actual server actions end-to-end) has
  been done yet.
- `/photos` and `/photos/[sku]` were checked against the live app (an
  isolated dev server copy, learning from Part 1's shared-`.next`
  lesson): `/photos` renders correctly and lists `HY-LOB01-C04` (tolerates
  the missing `photo_sets` table gracefully, showing "Not started");
  `/photos/[sku]` correctly shows a controlled error — `Couldn't load
  photo data: Could not find the table 'public.photo_assets' in the
  schema cache` — rather than crashing. That confirms the error-handling
  path works, but the real guided-review experience (thumbnails, hero
  picker, alt text) can't be shown until the migration runs.
- Pixel screenshots could not be captured this session (the Browser
  pane's screenshot tool errored: "the Browser pane is not displayed, so
  the page is not compositing frames") — verification above is by page
  text/console-log inspection instead, which is why the acceptance
  criteria's "screenshots" aren't literally attached to this handoff.

## Decisions Made

- **HEIC decoding via `heic-convert`/`libheif-js` (WASM), not a custom
  `sharp`/libvips build with libheif.** No native build step needed on any
  platform — matches "never require manual conversion" without trading it
  for an operator-side build dependency.
- **Background "standardization" implemented as a solid-color mat, not
  real background replacement.** True background swaps are explicitly
  gated by the issue's own truthfulness rules as requiring human
  preview/approval and being kept out of Version 1's default path; a mat
  is the truthful, deterministic interpretation that still delivers
  consistent, premium framing.
- **Alt text reuses the existing AI Provider abstraction rather than
  adding new AI infrastructure or a vision call.** One small, batched,
  factual, validated text-only request per photo set; mock by default.
- **Photo-set concurrency uses plain guarded conditional updates, not
  GMCOM-009's reserve/finalize/release token pattern.** Local, unpaid
  processing has a materially lower duplicate-cost risk than paid AI
  calls; a simpler mechanism is proportionate.
- **Hero/order UI uses number inputs and radio buttons, not drag-and-
  drop.** This app has no client-side interactive components yet;
  drag-and-drop would be a genuine new frontend pattern for a UX
  refinement, not a functional requirement ("choose/reorder" doesn't
  mandate a specific interaction model).

## Remaining Work

- **Phil: apply the migration below** (also appended to
  `supabase/schema.sql`) before the photo pipeline works against the real
  app.
- After that, a real Supabase-backed run — scan, generate, hero/order,
  alt text, approve — against `HY-LOB01-C04` (and ideally a real non-plant
  product, once one exists via the Product SKU Generator) would close the
  one remaining gap, the same way GMCOM-009's DB-level verification is
  still pending Phil's confirmation.
- Update GitHub Issues for GMCOM-009/010/011 — no GitHub API access
  available to Claude in this environment.
- (Unrelated to this issue, carried over from the GMCOM-009 handoff and
  still true): `BACKLOG.md`, `CLAUDE_ONBOARDING.md`, and six older
  handoff files in this repo remain modified/untracked on disk from an
  earlier session but were never committed. Left untouched again this
  session — not mine to fold in without knowing why they weren't
  committed, but flagging again since it's real, apparently-finished work
  at risk of being lost.

## Blockers or Risks

- No GitHub API access for closing Issues directly.
- No way to execute the migration or verify Supabase persistence/UI with
  real data in this session (see Verification above).
- Running two `next dev` processes against the same checked-out folder
  corrupts both (see Part 1) — worth a documented operational convention
  if multiple contributors regularly work in this exact checkout at once.

## SQL Migration — copy and paste into the Supabase SQL editor

Idempotent (`create table if not exists`, `create index if not exists`) —
safe to run even if part of it somehow already applied.

```sql
create table if not exists photo_sets (
  sku            text primary key references products(sku) on delete cascade,
  status         text not null default 'pending'
                   check (status in ('pending', 'processing', 'needs_review', 'approved')),
  approved_at    timestamptz,
  approved_by    text,
  last_scanned_at timestamptz,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);

create table if not exists photo_assets (
  id               uuid primary key default gen_random_uuid(),
  sku              text not null references products(sku) on delete cascade,
  original_filename text not null,
  original_path    text not null,
  source_format    text not null
                     check (source_format in ('jpeg', 'png', 'webp', 'heic', 'heif', 'unsupported')),
  content_hash     text not null,
  file_size_bytes  bigint,
  status           text not null default 'discovered'
                     check (status in ('discovered', 'processed', 'failed', 'unsupported')),
  warnings         jsonb not null default '[]'::jsonb,
  width            integer,
  height           integer,
  orientation      integer,
  color_profile    text,
  blur_score       numeric,
  exposure_mean    numeric,
  perceptual_hash  text,
  duplicate_of_id  uuid references photo_assets(id),
  discovered_at    timestamptz not null default now(),
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  unique (sku, original_path)
);

create index if not exists photo_assets_sku_idx on photo_assets (sku);

create table if not exists photo_derivatives (
  id                  uuid primary key default gen_random_uuid(),
  photo_asset_id      uuid not null references photo_assets(id) on delete cascade,
  sku                 text not null references products(sku) on delete cascade,
  derivative_type     text not null check (derivative_type in ('square_marketplace', 'detail_uncropped')),
  derivative_path     text not null,
  format              text not null check (format in ('jpeg', 'png')),
  width               integer not null,
  height              integer not null,
  file_size_bytes     bigint not null,
  processing_recipe   jsonb not null default '{}'::jsonb,
  background_treatment text,
  conversion_details   jsonb not null default '{}'::jsonb,
  order_index         integer,
  is_hero             boolean not null default false,
  alt_text            text,
  created_at          timestamptz not null default now(),
  updated_at          timestamptz not null default now(),
  unique (photo_asset_id, derivative_type)
);

create index if not exists photo_derivatives_sku_idx on photo_derivatives (sku, order_index);

create unique index if not exists photo_derivatives_one_hero_per_sku
  on photo_derivatives (sku)
  where is_hero;
```

## Recommended Next Action

**GMCOM-012 (or whatever number ChatGPT assigns) — apply and verify the
GMCOM-011 migration, then build the SKU-folder-readiness detector and
first real Shopify draft-publishing adapter that consumes *approved*
photo sets and *reviewed* Listing Packages together.** Per the
already-established sequencing (`handoffs/2026-08-02-photo-pipeline-
priority.md`: GMCOM-009 → GMCOM-010 → GMCOM-011 → Shopify), photos were
the last reliability/completeness prerequisite before marketplace
publishing. Both halves of a real listing — copy and photos — now exist
and are individually approvable; the next real gap is the same one
GMCOM-007/008/009's handoffs kept pointing at: getting a fully-reviewed
product (Listing Package reviewed *and* photo set approved) in front of
an actual buyer via a real Shopify draft product, storing the returned
Shopify product ID back on the record. Per the existing "Shopify before
Etsy" decision in `DECISIONS.md`.
