# Handoff — GMCOM-012: Shopify Draft Publisher

## Task

- Objective: build the first complete production workflow — Skrybix
  selection → GM Commerce intake → an approved photo set → a reviewed
  Listing Package → a real Shopify draft product. Assigned directly by
  Phil in chat (not yet mirrored to a GitHub Issue at assignment time).
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commits `572a858` (implementation) and `f2a073d`
  (real-store verification + client-credentials auth support). Pushed
  successfully.

## Work Completed

**`lib/shopify/`** (new): `types.ts`'s `ShopifyClient` interface has no
"create active" method anywhere — `createDraftProduct` always creates
`status: DRAFT`. `registry.ts` selects `mock` (default) vs `real` via
`SHOPIFY_PROVIDER`, the same config-driven pattern as the AI Provider
(GMCOM-007) and photo processor (GMCOM-011) abstractions. `publisher.ts`
is pure mapping from a reviewed Listing Package + approved photo
derivatives to Shopify's input shapes — hero image first, then ordered
squares, then detail views, each with its own generated alt text.
`real-client.ts` uses Shopify's staged-uploads flow for images (the
standard way to attach a local file, not a public URL, to a product).

**`publishToShopify`** (`app/review/actions.ts`): reads ONLY a reviewed
`listing_packages` row and an approved `photo_sets` row, refuses
otherwise. Creates once per SKU (`shopify_publications.shopify_product_id`
preserved); every subsequent publish updates that same product and
replaces its media rather than creating a duplicate. Before any update,
queries Shopify's live status fresh and refuses outright if the product
is `ACTIVE` there — never overwrites a live listing. Concurrency-guarded
the same way GMCOM-009 guards AI generation (a duplicate real Shopify
product is a meaningfully consequential external side effect), via a
lighter single-table conditional update (no cross-table condition needed
here, unlike GMCOM-009's reserve/finalize/release token pattern).

**New `shopify_publications` table** (migration + `schema.sql`), new
`/review` Shopify section (gates on approved photos in addition to the
existing reviewed-listing gate; shows publish state/product id/timestamp;
offers Republish when the published listing version or photo-approval
time no longer match what's currently reviewed/approved). Etsy stays
exactly the pre-existing stub — `lib/stub-marketplaces.ts` now only lists
`"etsy"`.

**19 new tests (79 total)**, including one that caught a real bug before
it shipped: the `publish_status` concurrency guard only allowed
reclaiming from `idle`/`failed`, which would have blocked every republish
attempt after a successful first publish (a product's status is
`published`, not `idle`, right after it succeeds). Fixed to also allow
reclaiming from `published`.

## Verification — real, not just written

**Full real pipeline run against `HY-LOB01-C04`** (real Supabase, real
photo files, real OpenAI, real Shopify store):
1. `scanPhotos` — discovered both of `HY-LOB01-C04`'s real JPEGs.
2. `generateDerivatives` — created 2 `square_marketplace` + 2
   `detail_uncropped` derivatives, auto-assigned hero/order, generated
   real alt text via OpenAI for all four.
3. `approvePhotoSet` — succeeded (hero present, alt text on every
   derivative).
4. Listing Package marked reviewed (see "Decisions Made" below — a
   one-time, explicitly authorized exception).
5. `publishToShopify` — **created a real Shopify draft product**:
   `gid://shopify/Product/10220386648384`.

**Field-by-field verification** (per ChatGPT's explicit request, relayed
by Phil — not just "the API returned 200"), querying the product back
directly from Shopify and diffing against the reviewed Listing Package
and approved photo set:
- Title, description (wrapped in `<p>` tags from `description` +
  `sales_summary`), vendor (`"Gathering Moss"`), product type, and tags —
  **exact match**.
- All 4 images present, in the correct hero-first-then-ordered-then-detail
  sequence, correct dimensions (2000×2000 for both squares; native
  aspect — 1800×2400 and 2202×1658 — for the two detail views).
- Alt text on every image — **exact match** to what GM Commerce generated
  and stored.
- `status: DRAFT` — confirmed directly from Shopify's own response.
- `shopify_publications.shopify_product_id` recorded correctly.

**Manual-edit resilience** (also per ChatGPT's request): edited the live
product directly via a separate Shopify API call (simulating a human
editing it in Shopify admin — replaced its tags), confirmed the edit took
and the product was still `DRAFT`, then called `publishToShopify` again
through GM Commerce. Result: same product ID (no duplicate created),
`published_at` unchanged (only `last_published_at` updated), canonical
tags and a fresh set of 4 media items restored (old media cleanly
replaced, not accumulated) — the integration doesn't break or duplicate
anything when a human has touched the record in between.

**`npm run typecheck`, `npm test` (79/79), and `npm run build` all
clean** at every stage of this session, including after the real-client
auth rework described below.

## Decisions Made

- **One-time, explicitly authorized exception to the standing
  human-review rule.** `HY-LOB01-C04`'s Listing Package was marked
  reviewed and its photo set approved by Claude, not Phil or Crystal —
  necessary to produce a real end-to-end test, and explicitly authorized
  by Phil in chat (offered three options; he chose "go ahead and approve
  it for me," with ChatGPT's concurring recommendation that this be
  treated as a development-only exception). **This does not change the
  standing decision** ("Human review remains before external
  publication," `DECISIONS.md`) — Claude will not self-approve real
  content again without equally explicit, in-the-moment authorization.
- **Shopify auth needs two modes, discovered firsthand, not
  hypothetically.** The Gathering Moss store's current Shopify
  plan/version does not offer the legacy "custom app" flow that issues a
  static `shpat_...` token at all (`Settings → Apps and sales channels →
  Develop apps` redirects back to "Build apps in Dev Dashboard" with no
  secondary option) — only a Dev-Dashboard app's Client ID + Secret,
  exchanged for a short-lived (~24h) access token via
  `grant_type=client_credentials`. `real-client.ts` now supports both
  (`SHOPIFY_ADMIN_API_ACCESS_TOKEN` directly, or
  `SHOPIFY_CLIENT_ID`/`SHOPIFY_CLIENT_SECRET` exchanged and cached with
  automatic re-exchange before expiry) so this works regardless of which
  flow a given store's plan offers, and so the client-credentials path
  doesn't silently break a day after setup.
- **Operational gotcha worth recording durably:** changing a Shopify
  app's requested access scopes does **not** retroactively grant them to
  an already-installed instance of that app. The token exchange succeeds
  either way, silently returning zero actual granted scopes until the app
  is uninstalled and reinstalled (or otherwise re-authorized). Cost real
  time to discover during this session's credential setup — worth anyone
  else hitting this knowing it's expected, not a bug.

## Remaining Work

- Update GitHub Issues (GMCOM-007/008/009/010/011/012) — no GitHub API
  access available to Claude in this environment. GMCOM-012 in particular
  has no Issue yet at all; recommend ChatGPT/Phil create one retroactively
  for the record, since this was assigned directly in chat.
- `HY-LOB01-C04` now has a real Shopify **draft** product
  (`gid://shopify/Product/10220386648384`) sitting in the store, created
  under the one-time exception above. Phil or Crystal should look at it
  in Shopify admin and decide what to do with it (edit further, publish
  it live themselves when ready, or discard) — GM Commerce itself will
  never make it live; that action doesn't exist anywhere in this
  codebase.
- The Shopify Client Secret / Client ID now live in `.env.local` as
  `SHOPIFY_CLIENT_ID`/`SHOPIFY_CLIENT_SECRET` (renamed from an earlier,
  mislabeled `SHOPIFY_ADMIN_API_ACCESS_TOKEN` that actually held the
  secret) — worth Phil knowing where these ended up, in case he manages
  this store's app credentials again later.
- Unrelated to this issue, carried over again: `BACKLOG.md`,
  `CLAUDE_ONBOARDING.md`, and six older handoff files in this repo remain
  modified/untracked on disk from an earlier session, never committed.
  Still not mine to fold in without knowing why, still flagging.

## Blockers or Risks

- No GitHub API access for creating/closing Issues directly.
- The Shopify access token in current use (client-credentials mode)
  expires in ~24h and auto-renews on next use — not a blocker (the code
  handles this transparently), just worth knowing if debugging a stale
  session far in the future.

## Recommended Next Action

**Etsy publishing, following the same pattern GMCOM-012 just proved out**
— OR, if Phil/ChatGPT prefer, closing out GMCOM-013/014's real-export
validation for the (separately-built, read-only) Shopify Listing Audit
Engine now that a real Shopify store and real product data genuinely
exist to test against. Either is a reasonable next step; recommending
Etsy specifically because it completes the "one canonical Listing Package
supports every sales option" architecture decision with a second real
channel, using infrastructure (the provider-neutral client pattern,
`shopify_publications`-equivalent tracking, never-overwrite-live logic)
that's now proven correct end-to-end for Shopify. Do not begin without a
ready GitHub Issue, per this repo's own operating rules.
