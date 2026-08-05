# Handoff — GMCOM-004: Connect live Skrybix intake to GM Commerce

## Task

- Issue: GMCOM-004 (https://github.com/HydraCoreSystems/gm-commerce-hq/issues/4)
- Objective: Integrate the verified Skrybix commerce-selection endpoint
  (GMCOM-003) with GM Commerce so selected plant SKUs enter the workflow
  without manual entry.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `001aa61` ("Wire up live Skrybix commerce intake
  (GMCOM-004)"). **Still not pushed to GitHub** — same standing blocker as
  GMCOM-002 (no push credentials in this environment).

## Work Completed

Confirmed GMCOM-003 actually merged this time (checked directly, not
assumed): `origin/master` in `skrybix-webapp` includes PR #1
("Add selected commerce handoff"), which adds `commerce_selected_at` /
`commerce_acknowledged_at` columns on `cuttings`, a
`CommerceSelectionControl` checkbox in the Cuttings UI, and two new
endpoints: `GET /api/commerce/v1/plants` (Bearer-authenticated, returns
only selected-and-unacknowledged cuttings) and
`POST /api/commerce/v1/plants/:cuttingId/acknowledge`. Read the actual
contract from `skrybix-webapp/README.md`'s "GM Commerce handoff" section
and the route source, not from the issue text alone.

Rewrote GM Commerce's Skrybix integration against that real contract:

- `lib/sources/skrybix.ts` — `getPendingSkrybixRecords()` (Bearer auth via
  `COMMERCE_EXPORT_KEY`, calls the real endpoint) and
  `acknowledgeSkrybixRecord()`. Completely replaces the old
  `sku-registry`/`SKU_REGISTRY_KEY` stand-in from GMCOM-002.
- `app/select/actions.ts` — `importSkrybixRecord()`: idempotent on
  `(source_system, source_record_id)`. If no row exists, inserts then
  acknowledges. If a row already exists but was never acknowledged
  upstream (a prior run's ack call failed), retries only the acknowledge
  call — never re-inserts. Skrybix is only told "acknowledged" after our
  own insert has committed, never before.
- `app/select/page.tsx` — Skrybix section now lists whatever Skrybix
  reports as pending (selected, unacknowledged), each with an Import
  button; no more "browse all inventory and select" model, since
  selection now happens in Skrybix itself. The Product SKU Generator
  section is unchanged (out of scope for this issue).
- `supabase/schema.sql` — added `source_record_id` (the idempotency key,
  paired with `source_system`; equals `sku` for both sources today, kept
  as a separate column since the source contract treats them as distinct
  concepts) and `source_acknowledged_at` (Skrybix only), plus a unique
  index on `(source_system, source_record_id)`.
- `README.md` — fully rewritten. It was still describing GMCOM-002's
  original bulk-import design (a stale gap from that session, not this
  one); now documents the real select → import → intake → review →
  publish workflow and the idempotency mechanics in detail, per this
  issue's explicit "document how selected items prevent repeated
  treatment as new" requirement.
- `.env.example` / `.env.local` — `SKU_REGISTRY_KEY` renamed to
  `COMMERCE_EXPORT_KEY` to match Skrybix's actual env var name. Read only
  server-side (`lib/sources/skrybix.ts`, never a Client Component, never a
  `NEXT_PUBLIC_` var) — never logged, never sent to the browser.
- Local, stale, never-committed `sku-registry` prototype code sitting in
  `skrybix-webapp`'s working tree (left over from GMCOM-002's session) was
  stashed (`git stash`, not deleted) before pulling the real merged PR, to
  avoid losing anything by accident. It's fully superseded by the real
  `commerce/v1` implementation and doesn't need to be recovered.

## Verification

**Confirmed working:**

- `npm run build` compiles clean, zero type errors.
- Read the actual deployed... attempted to. See blocker below — this is
  where verification stopped.

**Not yet tested — genuinely blocked, not skipped:**

- **The live Skrybix endpoint is not actually reachable as deployed.**
  `curl https://skrybix-webapp.vercel.app/api/commerce/v1/plants` returns
  `307` redirecting to `/login`, both for the list endpoint and the
  acknowledge endpoint — meaning Skrybix's own session middleware is
  intercepting these requests before they ever reach the route handler.
  This contradicts the code actually merged: the pulled `middleware.ts`
  (matching commit `83a7d2a`) correctly excludes
  `api/commerce/v1/plants` from its matcher. **My working theory: Vercel's
  "Redeploy" action on an existing deployment rebuilds that same pinned
  commit — it does not automatically pick up new commits unless a fresh
  deployment is triggered from the latest one.** If "Skrybix redeployed"
  meant clicking Redeploy on a pre-merge deployment, this would produce
  exactly this symptom. Worth checking the Vercel dashboard's Deployments
  tab for which commit SHA is actually live in Production — it should
  read `83a7d2a` (or later), not `c27f06e` or earlier.
  - I could not investigate further myself — no Vercel access in this
    environment.
- **No real cutting has been imported end-to-end.** Blocked entirely on
  the above — there's no way to test against real data until the live
  endpoint actually excludes itself from the login redirect.
- `COMMERCE_EXPORT_KEY`'s real value (configured in Skrybix's Vercel
  Production env per Phil) was never shared into GM Commerce's own
  `.env.local` — even once the deployment issue is resolved, that value
  is still needed before real testing can happen.

## Decisions Made

- No manual-workaround was attempted (per explicit instruction) — the
  deployment issue is left for Phil to resolve via Vercel, not routed
  around here.
- Kept `source_record_id` as a column distinct from `sku`, even though
  they're equal for both sources today, because the Skrybix contract
  itself models them as separate concepts (`sourceRecordId` vs `sku`) —
  cheap to keep them distinct now rather than needing a migration later if
  they ever diverge.

## Remaining Work

- Confirm the Production deployment in Vercel is actually serving commit
  `83a7d2a` (or later) of `skrybix-webapp`, and redeploy from the correct
  commit if not.
- Once confirmed reachable, share the real `COMMERCE_EXPORT_KEY` value
  (from Skrybix's Vercel Production env) into `gm-commerce/.env.local`.
- Apply the additive schema migration (`source_record_id`,
  `source_acknowledged_at`, unique index) to the real Supabase table —
  given to Phil directly in chat.
- Then: select at least one real cutting in Skrybix, import it from GM
  Commerce's `/select` page, and confirm it lands in `/pipeline` with
  status `intake` — the actual "verify one real cutting end-to-end" step,
  still outstanding.
- Push `gm-commerce` to GitHub (still outstanding from GMCOM-002).
- Update GitHub Issue #4 directly — I have no GitHub API access in this
  environment, so this needs to happen via GitHub Desktop/web the same way
  every other Issue update has.

## Blockers or Risks

- **Live Skrybix deployment likely stale** — see Verification above. This
  is the one thing blocking end-to-end verification; the integration code
  itself is complete and builds clean.
- No GitHub push/API credentials (standing blocker, unchanged).
- `gm-commerce` still isn't pushed to GitHub.

## Recommended Next Action

Phil: check the Vercel dashboard for `skrybix-webapp`'s Production
deployment commit SHA. If it's not `83a7d2a` or later, trigger a real
redeploy from the latest commit (not the "Redeploy" button on an old
deployment). Once `curl .../api/commerce/v1/plants` returns `401` (not a
`307` redirect) even with a garbage bearer token, that confirms it's
live — share the real `COMMERCE_EXPORT_KEY` value at that point and I can
complete the one remaining acceptance criterion (a real cutting verified
end-to-end) immediately.
