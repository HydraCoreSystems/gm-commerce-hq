# Handoff — GMCOM-001: Document and validate the Product SKU Generator baseline

## Task

- Issue: GMCOM-001 (per `CLAUDE_ONBOARDING.md` — no corresponding GitHub Issue was
  visible/editable from this session; no GitHub API/credentials available here,
  so this handoff could not be cross-posted to Issues directly. Recommend the
  project manager or Phil mirror this into the actual Issue if one exists.)
- Objective: Establish an accurate, reproducible baseline of the Product SKU
  Generator so the next implementation decision is grounded in working code,
  not memory of past sessions.
- Repository: `HydraCoreSystems/product-sku-generator`
- Branch: `main`

## Work Completed

- The Product SKU Generator's repository location is now recorded:
  `HydraCoreSystems/product-sku-generator` (private), pushed 2026-08-01.
  Single commit so far ("Initial commit: Product SKU Generator") containing
  the full working tree: `server.js`, `categories.json`, `data/*.json`,
  `public/` (UI), `README.md`, `package.json`, launcher scripts. `node_modules/`
  is correctly gitignored.
- Reviewed the app end-to-end against its own `README.md` and `server.js` to
  confirm the baseline described below is accurate as of today, not a stale
  summary from an earlier session.

## Verification

**Confirmed working (tested, not just written):**

- SKU format `{PREFIX}-{4-CHAR-CODE}-{3-DIGIT-SEQ}` across all 9 configured
  categories: PAL, POT, SOIL, SUP, APP, TAG, KEY, GC, HP.
- Plant Pals animal-name shortening (`ANIMAL_SHORT_CODES` lookup + fallback
  to first-4-letters/padded-X for unknown animals) — verified via sandboxed
  test generations (e.g. known animal → correct curated code; unknown animal
  → correct fallback code).
- Counter persistence in `data/counters.json` — currently matches real
  Shopify state as of 2026-08-01 (PAL=69, POT=31, SOIL=7, SUP=5, APP=4,
  TAG=1, KEY=8, GC=4, HP=1), reconciled during today's full-catalog SKU
  retrofit so the next SKU generated in any category won't collide with an
  already-issued Shopify SKU.
- In-app category management (add/edit/remove categories without editing
  `categories.json` by hand).
- Sequence-per-category behavior (one running counter per prefix, ignoring
  attribute value) — this is an intentional design choice, not a bug.

**Not yet tested / genuinely unknown:**

- **Real photo-folder creation has only been tested against a dummy path in
  a sandbox**, not against the real configured photo root
  (`C:\Users\pwach\OneDrive\Pictures\All Product Photos`, per
  `data/config.json`). Nobody has generated a SKU through the live app and
  confirmed the folder actually lands there correctly.
- The 6 newer categories (SOIL, SUP, APP, TAG, KEY, GC, HP) have never been
  exercised through the app's own UI. Their counters were set by direct
  edit to `data/counters.json` to match Shopify's retrofit — the app itself
  has never generated a SKU in any of these categories.
- No automated tests exist. All verification to date has been manual/sandbox
  spot-checks.
- Counter/history survival across an actual app restart on Phil's real
  machine hasn't been explicitly observed (should be true by construction —
  it's just a JSON file read/write — but "should be true" isn't the same as
  "watched it happen").
- No concurrency protection (fine for a single local user clicking one
  button at a time, per `README.md`'s own reasoning — flagging only so it's
  a documented assumption, not an accidental gap).

## Decisions Made

No new architectural decisions. This handoff reinforces two decisions
already recorded in `DECISIONS.md`:

- "Product SKU Generator local JSON is interim" — still accurate. Nothing
  about today's work upgrades it to a permanent data store.
- Worth naming explicitly for the project manager: the "SKU Repository"
  referenced elsewhere (Shopify catalog + this generator + Skrybix) exists
  today only as a Cowork live-view artifact that merges three sources on
  request, plus a one-time exported spreadsheet snapshot. There is no
  database yet. This matches — and is the reason for — Milestone 1's
  "central product record model" not being started yet. Flagging so it
  isn't mistaken for already-solved.

## Remaining Work

To close Milestone 0 for real (per `ROADMAP.md`'s own exit criteria):

- Run one real end-to-end generation through the live app for a category
  other than PAL (e.g. a POT or SUP SKU for an upcoming real product) and
  confirm the folder is created in the correct real OneDrive location.
- Confirm the app survives a real restart with counters intact (should be
  trivial to observe the next time it's used normally).

## Blockers or Risks

- No GitHub Issues access, GitHub API access, or push credentials exist in
  this session's environment — all GitHub writes this session went through
  Phil manually via GitHub Desktop. Any workflow that assumes an AI can
  autonomously open/close Issues or push commits will need either a
  credentialed integration or continued manual relay through Phil.
- No live connection between this session and ChatGPT exists — coordination
  currently depends on Phil copy-pasting between the two. Worth the project
  manager knowing this isn't automatic yet.

## Recommended Next Action

Do **not** start Milestone 1 yet. First close the real-world validation gap
above — it's a 5-minute task for Phil (generate one real SKU in a
not-yet-used category, confirm the folder lands correctly) and it's the
literal "Immediate Priority" already stated in `PROJECT_MANAGER.md`.

Once that's confirmed, Milestone 1 — Central Commerce Foundation is the
correct next module: a small durable backend (product record + listing
status + marketplace IDs), most likely following the same Next.js +
Supabase pattern already proven in `skrybix-webapp`, since that pattern is
already validated, low-risk, and matches this project's own stated
preference for reusing what already works rather than introducing a new
stack.
