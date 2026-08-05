# Handoff — GMCOM-007: Canonical AI Listing Package generator + Provider abstraction

## Task

- Issue: GMCOM-007 (https://github.com/HydraCoreSystems/gm-commerce-hq/issues/7)
- Objective: Replace the stub processing step with the first real
  AI-powered Listing Package generator — a durable, editable, canonical
  package, not a marketplace listing. Mid-implementation, Phil made an
  explicit architectural decision: GM Commerce must never be locked to a
  single AI vendor — build a Provider abstraction first, implement OpenAI
  as the first real provider, leave Anthropic wired but untested, keep a
  zero-cost Mock provider for dev/testing.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `5ee06f9` ("Build the canonical AI Listing
  Package generator (GMCOM-007)"). **Pushed successfully** — note for
  the project record: this is the first time in this project's history
  that a push from this environment has succeeded. Prior sessions
  (GMCOM-002/004/006) repeatedly hit "no push credentials"; that appears
  resolved now. Worth confirming with Phil whether something changed on
  his end (e.g. a credential helper now available) so `STATUS.md`'s
  standing blocker note can be corrected going forward rather than
  reflexively repeated.

## Work Completed

**The Provider abstraction**, exactly matching the requested shape:

```
GM Commerce → Listing Generator → Prompt Builder → AI Provider interface
                                                      ├── OpenAI Provider
                                                      ├── Anthropic Provider
                                                      └── Mock Provider
```

- `lib/ai/provider-types.ts` — the `AIProvider` interface. One method,
  `generate(request) → { rawJson, providerName, model }`. A provider's
  only job is transport: take an already-built prompt + schema, call its
  model, hand back raw text. It never sees or influences business logic.
- `lib/ai/prompt-builder.ts` — the **one** canonical prompt-building
  pipeline. Content-integrity rules, tone, and the Listing Package JSON
  Schema are defined **once** here and handed identically to whichever
  provider is active. No provider file duplicates any of this.
- `lib/ai/provider-registry.ts` — the only place that decides which
  vendor is active, purely from the `AI_PROVIDER` environment variable.
  Defaults to `mock` if unset, so nothing ever calls a paid API by
  accident.
- `lib/ai/providers/mock-provider.ts` — zero-cost, no network call,
  obviously-fake labeled content, schema-conformant. Requires no
  credentials at all.
- `lib/ai/providers/openai-provider.ts` — **real, tested** (see
  Verification). Uses the Responses API (`client.responses.create`) with
  `text.format: {type: "json_schema", ...}` for structured output.
  Default model `gpt-5.6-luna` (OpenAI's cost-optimized tier — same
  reasoning as running Anthropic at low effort), overridable via
  `OPENAI_MODEL`.
- `lib/ai/providers/anthropic-provider.ts` — carried over from before
  this decision, refactored onto the new interface. Structurally
  complete, **left untested against the real API per instruction** — the
  Mock provider is what's actually been exercised for now (see below);
  should work as soon as a key is available.
- `lib/ai/listing-generator.ts` — rewritten to talk **only** to
  `provider-registry.ts` / `prompt-builder.ts`. No vendor SDK import
  anywhere outside the three provider files.

**New `listing_packages` table** (see prior handoff draft — table
already existed in the DB before this pivot; unchanged by the provider
work): `reviewed_at` is the durable "a human approved this" marker.
`triggerListingGeneration` (`app/actions.ts`) refuses to regenerate once
it's set; `unlockForRegeneration` (`app/review/actions.ts`) is a separate,
explicit action required first. Old stub-AI columns on `products`
(`listing_title`/`listing_description`/`listing_tags`/`processed_at`)
were dropped as fully superseded — this and the table itself were
already migrated in the earlier part of this session, before the
architecture pivot; no further schema change was needed for the provider
work itself.

**UI**: home page's "Ready for AI" section gained a "Generate Listing
Package" button per product. `/review` was rebuilt around the real
`listing_packages` row: editable title/short title/description/sales
summary/tags/category/price/care details, a collapsible "What the AI
knew about this product" transparency panel (the literal `source_facts`
given to the model), a "Reviewed"/"Needs review" badge, "Mark Reviewed",
and "Unlock for regeneration". Publish (still a stub) is gated behind
having marked the package reviewed.

## Verification

**Confirmed working, `npm run build` clean:**

- Full lifecycle tested end-to-end against an **isolated scratch
  product** (`TEST-GMCOM007-001`, inserted directly at `ready_for_ai`,
  deleted afterward) with `AI_PROVIDER=mock`:
  1. Clicked "Generate Listing Package" → real Supabase row created,
     confirmed by direct query: correct schema, `model: "mock:mock-v1"`,
     `source_facts` correctly capturing the product's real attributes and
     notes.
  2. Edited the title via the Review Queue form → confirmed the new value
     persisted in the database, not just rendered client-side.
  3. Clicked "Mark Reviewed" → `reviewed_at` set, UI badge and publish
     gate updated correctly.
  4. **Attempted to regenerate while reviewed** → server threw the
     expected refusal (`"This listing package has already been reviewed
     and approved..."`), confirmed via server logs; confirmed via direct
     query that the edited title and `reviewed_at` were **untouched** —
     the guard is real, not just a UI-level disable.
  5. Clicked "Unlock for regeneration" → `reviewed_at` cleared, edited
     title still intact (unlocking doesn't touch content).
  6. Regenerated again → succeeded this time, confirmed via a new
     `generated_at` timestamp and fresh Mock content — the guard only
     blocks while actually reviewed, not permanently.
- **OpenAI provider: verified for real, end-to-end.** Once
  `OPENAI_API_KEY` was added, ran a real generation against an isolated
  scratch product (`TEST-GMCOM007-OPENAI-001`) with `AI_PROVIDER=openai`:
  a real `client.responses.create` call to `gpt-5.6-luna` succeeded
  (~4s), and the persisted `listing_packages` row was inspected directly:

  ```
  proposed_title: "6-Inch Glazed Ceramic Pot"
  short_title:    "Glazed Ceramic Pot"
  description:    "A 6-inch pot with a glazed ceramic finish, suitable
                    for plant display or other small home uses."
  category:       "Pot"
  tags:           ["pot", "ceramic", "glazed"]
  price:          null   <- correctly withheld, no real price data given
  care_details:   null   <- correctly withheld, no real care data given
  model:          "openai:gpt-5.6-luna"
  ```

  Content-integrity rules held under real model output, not just in the
  prompt text: the model used only the facts actually given (size,
  glazed-ceramic note, POT category) and left `price`/`care_details` null
  rather than inventing anything. Scratch data deleted afterward.
  One incidental fix along the way: a stale `.next` build cache (left
  over from an earlier `npm run build` verification pass) was making the
  dev server serve cached data after the `.env.local` provider switch —
  `rm -rf .next` before restarting `npm run dev` resolved it. Worth
  remembering for future sessions: after switching `AI_PROVIDER` (or any
  env var) mid-session, clear `.next` before assuming a UI discrepancy is
  a code bug.
- **Anthropic provider: untested, per explicit instruction.** Same code
  path as the working GMCOM-007-draft-1 Anthropic call from earlier in
  this session, refactored onto the new interface — should work, but "OK
  in isolation" isn't "verified," so treating it as unverified until it's
  actually run.
- Cleaned up all scratch data (product row, listing package row)
  afterward — no test artifacts remain in the real database.

## Decisions Made

- **AI_PROVIDER defaults to `mock`.** A missing/unset config value should
  never result in an unexpected real API charge. Explicit opt-in
  (`AI_PROVIDER=openai` or `=anthropic`) is required to spend anything.
- **Model selection is a `DEFAULT_MODEL` constant per provider,
  overridable via `OPENAI_MODEL` / `ANTHROPIC_MODEL`.** Keeps the common
  case (just pick a provider) simple while still allowing a model swap
  without a code change.
- **Provenance recorded as `"{providerName}:{model}"` in the existing
  `listing_packages.model` column** (e.g. `openai:gpt-5.6-luna`,
  `mock:mock-v1`) rather than adding two new columns — non-secret
  metadata only, sufficient to answer "what generated this" without a
  schema change.
- **The provider interface intentionally does not touch content
  integrity at all.** A provider that received business rules baked into
  its own call would violate "no duplicated prompt logic between
  providers" the moment a second real provider existed — everything
  content-related lives once, in `prompt-builder.ts`.

## Remaining Work

- `AI_PROVIDER` is set to `openai` in `.env.local` now that it's
  verified — this is the live default going forward for real day-to-day
  use.
- Anthropic provider remains untested until a key is supplied — not
  urgent per Phil's own sequencing.
- Update GitHub Issue #7 — no GitHub API access available to Claude in
  this environment.

## Blockers or Risks

- No GitHub API access for updating Issues directly (standing, unchanged) —
  **but note the push blocker itself appears resolved this session**, see
  the branch note above.

## Recommended Next Action

GMCOM-007 is fully complete and verified — real AI generation works,
content integrity holds under real model output, and the regeneration
guard is proven. The natural next issue is wiring the Listing Package
into an actual marketplace draft (Shopify first, per
`gm-commerce-hq/DECISIONS.md`'s existing "Shopify before Etsy" decision).
Before starting that, it's worth Phil (or Crystal) actually clicking
"Mark Ready for AI" on the real cutting `HY-LOB01-C04` — folder and
photos are both already confirmed as of this session — and running one
real generation on genuine business data, not just scratch test records,
so the first real Listing Package in the system is a real product.
