# Handoff — GMCOM-006: Build the guided intake experience

## Task

- Issue: GMCOM-006 (https://github.com/HydraCoreSystems/gm-commerce-hq/issues/6)
- Objective: Transform GM Commerce's intake foundation into a guided,
  polished, business-user-facing workflow from imported SKU through a
  durable "Ready for AI" state — for Phil, Crystal, and eventually
  employees, not developers.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `a3eb9ab` ("Build the guided intake experience
  (GMCOM-006)"). **Not yet pushed** — this repo has no remote configured
  at all yet (`git remote -v` returns nothing); standing blocker, no
  GitHub push credentials in this environment.

## Work Completed

**The home page (`/`) is now the guided intake experience** — replacing
the old developer-style `/pipeline` list. Each product in intake renders
as a card showing:

- Identity: product name (prominent), SKU (a styled tag, not a raw
  technical ID), and a source badge (Skrybix / SKU Generator).
- A three-item readiness checklist (photo folder set up → photos added
  and confirmed → ready for AI), each marked done/next/pending with a
  checkmark or step number — never raw database status strings.
- Exactly one primary action at a time, matching whichever checklist item
  is next: **Set Up Photo Folder** → **I've Added the Photos** → **Mark
  Ready for AI**. Completed steps show as confirmed state, not competing
  buttons.
- An optional, visually secondary "Add a note" section (`<details>`,
  collapsed by default) that never competes with the primary action.

**New durable state: `ready_for_ai`.** Added to the `status` check
constraint alongside the existing `intake`/`review`/`published`/`archived`
values. Reached only via `markReadyForAI()` (`app/actions.ts`), which:

- Is gated **server-side**, not just by a disabled button — re-fetches
  the record and throws if `photo_folder_path` or `photos_confirmed_at`
  is missing, so the transition can't be reached by any path other than
  the UI actually enforcing both conditions first.
- Is idempotent — guarded by `.eq("status", "intake")`, so a duplicate or
  late click on an already-advanced record is a harmless no-op.
- Explicitly does **not** call `lib/stub-ai.ts` or anything else — this
  issue is intake only, per its own scope boundary.

**New column: `photos_confirmed_at`.** A human attestation ("I've Added
the Photos"), deliberately separate from any automatic filesystem check —
a folder can exist and be empty, or contain files nobody's actually
looked at, so this is a conscious confirmation, not an inference.
`lib/photo-root.ts` gained `countPhotos()`, shown alongside the
confirmation as a helpful cross-check ("We found 2 photos there
already") — never the gate itself. The confirmation is reversible (an
"undo" link) so a premature confirmation can be corrected honestly.

**Visual design pass** (`app/globals.css` rewritten): real elevation
(layered shadows, not flat borders), a proper type scale, gradient-backed
nav bar and stat tiles, refined badges/checklist/button styles, and
responsive breakpoints for desktop/laptop/tablet (including a mobile
table-to-stacked-rows fallback for `/select`). Applied consistently across
all four pages, not just the new home page — `/select` and `/review` got
matching page headers and empty-state treatment with no change to their
underlying logic.

**Preserved from GMCOM-004:** `/select`'s Skrybix import and Product SKU
Generator selection flows are untouched functionally. `/pipeline` now
redirects to `/` (old links/bookmarks keep working) rather than being
deleted outright.

## Verification

**Confirmed working, `npm run build` clean:**

- Used the real GMCOM-004 cutting (`HY-LOB01-C04`, still sitting in
  `intake` from the prior session) to test the **folder-creation** step
  for real: clicked "Set Up Photo Folder," confirmed a real folder was
  created at `C:\Users\pwach\OneDrive\Pictures\All Product Photos\HY-LOB01-C04`
  on disk, and the checklist/UI updated correctly.
- **Deliberately did not** fake the photo-confirmation or Ready-for-AI
  steps on that real cutting — no real photos exist for it yet, and
  attesting otherwise would have written false data onto a real business
  record. Instead:
- Inserted an isolated scratch product (`TEST-QA-001`, never touching
  Skrybix or the SKU Generator's real selection flows) to exercise the
  remaining steps: photo-folder fallback-creation, the "I've Added the
  Photos" confirmation, the "undo" link, and the gated "Mark Ready for
  AI" button (which only appeared once both conditions were true). All
  worked correctly, verified via the rendered page state at each step,
  not just by reading the code.
- Deleted the scratch product row and its test folder afterward — the
  real cutting was left exactly where it honestly belongs: photo folder
  confirmed, photos not yet confirmed, one real product still correctly
  showing in "Need your attention."
- Confirmed `/select` (Skrybix + SKU Generator sections), `/review`
  (empty-state), and the `/pipeline` → `/` redirect all still work with no
  server errors.
- **Screenshots: not yet captured.** The Browser pane wasn't displayed on
  Phil's end during this session, so screenshot compositing timed out
  every attempt (confirmed via repeated retries, not a one-off glitch).
  Functional verification above used `read_page`/`get_page_text` instead,
  which don't require the pane to be visually rendered. Screenshots are
  outstanding — see Remaining Work.

## Decisions Made

- `ready_for_ai` sits between `intake` and `review` in the status
  lifecycle. The existing stub-AI-triggering logic from GMCOM-002 (which
  used to run on the old `/pipeline`'s "Process" button) was **not**
  carried forward as part of this issue — advancing to `ready_for_ai` is
  now the terminal action for guided intake, and picking those records up
  for AI processing is explicitly the next issue's work, not this one's.
- Chose a human-attested `photos_confirmed_at` column over relying solely
  on `countPhotos()`, per the issue's explicit "honest filesystem
  verification (human confirmation acceptable for V1)" allowance — shown
  together, not as alternatives.
- `/pipeline` was redirected rather than deleted, to avoid breaking any
  existing links without needing to ask whether any exist.

## Remaining Work

- **Capture and attach screenshots** — needs Phil to open the Browser
  pane; I can complete this the moment it's visible, without any further
  code changes.
- **Push `gm-commerce` to GitHub** — no remote configured at all yet;
  needs Phil to create the GitHub repo and push via GitHub Desktop (the
  standing blocker from GMCOM-002/004, now three commits deep locally).
- Update GitHub Issue #6 — no GitHub API access available to Claude in
  this environment.

## Blockers or Risks

- No GitHub push/API credentials in this environment (standing, unchanged).
- Screenshot capture depends on the Browser pane being visible on Phil's
  side — a session/environment condition, not a code issue.

## Recommended Next Action

**Build the real AI listing-generation step**, now that `ready_for_ai` is
a durable, honestly-gated state with real records able to reach it: pick
up `ready_for_ai` products, generate real (or still-stub, if cost/scope
isn't confirmed yet) listing content, and move them to `review`. Per
`gm-commerce-hq`'s AI-direction notes, confirm scope and ongoing cost with
Phil before wiring in any real model call — this is genuinely new
infrastructure, not an extension of anything already built.
