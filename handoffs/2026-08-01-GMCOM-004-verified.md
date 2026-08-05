# Handoff — GMCOM-004: Connect live Skrybix intake to GM Commerce (VERIFIED)

## Task

- Issue: GMCOM-004 (https://github.com/HydraCoreSystems/gm-commerce-hq/issues/4)
- Objective: Integrate the verified Skrybix commerce-selection endpoint
  (GMCOM-003) with GM Commerce so selected plant SKUs enter the workflow
  without manual entry.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `001aa61` ("Wire up live Skrybix commerce intake
  (GMCOM-004)"). **Still not pushed to GitHub** — standing blocker, see
  below.

## Work Completed

See `handoffs/2026-08-01-GMCOM-004-integration-built-verification-blocked.md`
for the full implementation writeup (`lib/sources/skrybix.ts`,
`app/select/actions.ts`'s idempotent `importSkrybixRecord`,
`supabase/schema.sql`, README rewrite). No code changed this session —
the two things blocking verification were both infrastructure, resolved
by Phil:

1. Skrybix's Vercel Production deployment was stale (serving a pre-merge
   commit); Phil confirmed it now serves commit `413189e`. Verified myself
   via `curl .../api/commerce/v1/plants` — returns `401` with the expected
   JSON error body instead of a `307` redirect to `/login`.
2. GM Commerce's own `products` table migration (`source_record_id`,
   `source_acknowledged_at`, unique index) had NOT actually been applied —
   what Phil had run at that point was Skrybix's own migration
   (`commerce_selected_at`/`commerce_acknowledged_at` on `cuttings`), a
   different migration on a different Supabase project. Caught this by
   querying the real `products` table directly rather than trusting the
   "migration applied" report at face value; gave Phil the correct SQL a
   second time, targeted explicitly at GM Commerce's own project
   (`wcrcllhvgbhykbonopzx`), which he then ran successfully.

`COMMERCE_EXPORT_KEY` was added directly to `gm-commerce/.env.local` by
Phil himself, not pasted in chat — I never saw or handled the plaintext
value beyond it existing in that file.

## Verification

**Confirmed working — real cutting, real round trip, checked
independently at every layer, not just "the UI looked right":**

1. Phil selected a real cutting, `HY-LOB01-C04` ("Hoya lobbii"), for GM
   Commerce in Skrybix's own Cuttings UI.
2. Confirmed via direct `curl` to `GET /api/commerce/v1/plants` (using the
   real `COMMERCE_EXPORT_KEY`) that it appeared in the pending list before
   import.
3. Clicked **Import** on GM Commerce's `/select` page. First attempt
   failed with `column products.source_acknowledged_at does not exist` —
   the real error surfaced immediately rather than silently succeeding
   with partial data, confirming the missing-migration diagnosis above
   was correct, not a guess.
4. After the correct migration was applied, retried Import — succeeded.
5. Queried GM Commerce's real `products` table directly (not just the
   rendered page): row exists, `sku` = `HY-LOB01-C04`, `source_system` =
   `skrybix`, `source_record_id` = `HY-LOB01-C04`, `status` = `intake`,
   `source_acknowledged_at` set, `attributes` correctly carrying
   `parentSourceRecordId` (`HY-LOB01`), `sourceState` (`active`),
   `sourceSelectedAt`, `sourceCreatedAt`.
6. Queried Skrybix's own `cuttings` table directly (its own Supabase
   project, separate from GM Commerce's) and confirmed
   `commerce_acknowledged_at` is now set there too, timestamped
   consistently with the import (`selected` 00:34:33 → GM Commerce's
   `source_acknowledged_at` 00:38:21.715 → Skrybix's
   `commerce_acknowledged_at` 00:38:23.701, all UTC) — the acknowledgement
   genuinely round-tripped, this isn't just GM Commerce's own copy looking
   right in isolation.
7. GM Commerce's `/select` page immediately reflected "Nothing selected in
   Skrybix right now" — confirming Skrybix's pending-list filter correctly
   excludes now-acknowledged records.
8. GM Commerce's `/pipeline` page shows the record correctly: "HY-LOB01-C04
   — Hoya lobbii (Skrybix, cutting)", photo folder "not yet confirmed" —
   the correct waiting-for-photos/notes state, exactly as required.
   **Deliberately did not click Confirm/Create Photo Folder or Process on
   this real record** — those are out of scope for this issue (AI listing
   generation is explicitly excluded), so it was left exactly at the
   required intake stage rather than pushed further for the sake of a more
   "complete-looking" demo.

**Not exercised this session (by design, not oversight):**

- The retry-without-duplicating branch (existing row, not yet
  acknowledged) wasn't forced/tested against real data — doing so would
  have meant deliberately breaking the acknowledge call against a real
  business record. That logic was verified by code review during
  implementation, not by an artificial failure injection against
  production data.
- Product SKU Generator's non-plant flow is unaffected by this issue and
  wasn't re-tested (its log is still empty — same outstanding item as
  before, unrelated to this issue).

## Decisions Made

- No new architectural decisions this session. Confirmed a process point:
  when a status report says "the migration was applied," verify against
  the actual schema before proceeding, not just before implementation —
  this caught a real, easy-to-make mix-up (two similarly-named migrations
  on two different Supabase projects, run within the same conversation)
  that would otherwise have surfaced as a confusing runtime error blamed
  on the code rather than the data layer.

## Remaining Work

- **Push `gm-commerce` to GitHub.** Still the standing blocker — commit
  `001aa61` exists locally only. Phil, via GitHub Desktop.
- Update GitHub Issue #4 to closed/verified — no GitHub API access
  available to Claude in this environment.
- Generate at least one real SKU through the Product SKU Generator's own
  live app — its `sku-log.json` is still empty in both known copies; this
  predates GMCOM-004 and is unrelated to it, but still outstanding.

## Blockers or Risks

- `gm-commerce` still isn't pushed to GitHub — everything in this and the
  prior GMCOM-004 handoff exists only in this local commit until that
  happens.
- No GitHub API/push credentials in this environment (standing, unchanged).

## Recommended Next Action

**GMCOM-005 direction (non-plant intake) already exists per Copilot's
completed work** — worth the project manager confirming its scope lines
up with this session's plant-side result before sequencing further.
Beyond that, the next concrete GM Commerce issue should be: **build the
real AI listing-generation step** (replacing `lib/stub-ai.ts`), since
intake for both plant and non-plant products now demonstrably works
end-to-end and that stub is the next thing actually blocking a usable
Review Queue. Per `gm-commerce-hq`'s own AI-direction notes, confirm scope
and ongoing cost with Phil before wiring in a real model call — that's a
real new integration (API key, per-call cost), not an extension of
anything already built.
