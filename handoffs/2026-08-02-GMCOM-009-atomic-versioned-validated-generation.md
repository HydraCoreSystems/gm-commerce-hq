# Handoff — GMCOM-009: Make AI generation atomic, versioned, and runtime-validated

## Task

- Issue: GMCOM-009 — [HydraCoreSystems/gm-commerce-hq#8](https://github.com/HydraCoreSystems/gm-commerce-hq/issues/8)
- Objective: Harden the Listing Quality Engine (GMCOM-007/008) to prevent
  duplicate paid AI generation, protect human edits/approvals from
  overwrite, and validate all provider responses before persistence —
  reliability hardening before Shopify, not a feature addition.
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `main`, commit `81a5401` ("Make AI generation atomic, versioned,
  and runtime-validated (GMCOM-009)"). Pushed successfully.

## Work Completed

**1. Runtime schema validation (`lib/ai/validation.ts`, new).** Both AI
calls in the pipeline (`lib/ai/listing-generator.ts`) used to do
`JSON.parse(rawJson) as T` — a bare cast that enforced nothing at runtime.
Malformed JSON, a missing field, or a wrong type from any provider would
have been persisted straight into `listing_packages`. Now both calls go
through `parseAndValidate()`, backed by zod schemas that mirror
`ListingPackageDraft`, `QualitySummary`, and the combined review+revise
envelope exactly (a `satisfies z.ZodType<...>` check means the schema and
the TypeScript type can't silently drift apart). Invalid output throws a
`ProviderOutputError` with a field-level description of what was wrong
(e.g. `proposedTitle: String must contain at least 1 character(s)`) —
deliberately never the raw response text itself, per the issue's explicit
"never expose raw responses" boundary. The first draft is validated before
it's used to build the *second* (paid) call's prompt, so a bad first draft
fails fast rather than corrupting the second call too.

**2. Atomic generation locking (`supabase/schema.sql`, new SQL section;
`lib/ai/generation-lock.ts`, new).** Added a `generating` status and three
Postgres functions:
- `gmcom_reserve_generation(sku)` — claims a product for generation
  *before* any paid call is made. Uses `SELECT ... FOR UPDATE` inside the
  function so exactly one concurrent caller can win; a second caller for
  the same SKU sees `status = 'generating'` and is refused immediately,
  with zero provider calls made. Eligible from `ready_for_ai` (first
  generation) or from `review` with an unreviewed package (regeneration)
  or from a stale `generating` lock older than 15 minutes (crash
  recovery). Returns a random lock token the caller must present to
  finalize or release.
- `gmcom_finalize_generation(...)` — one transaction that archives the
  package being replaced into `listing_package_versions` (if one exists),
  upserts the new package with an incremented `version`, and releases the
  lock by moving status to `review`. Verifies the lock token still matches
  before touching anything, and refuses (raises) if the existing package
  is somehow already reviewed — belt-and-suspenders on top of what
  `gmcom_reserve_generation` already prevents.
- `gmcom_release_generation(sku, token)` — restores the product's
  pre-generation status on any failure (AI error, validation failure) so
  it's never stuck in `generating`.

`app/actions.ts`'s `triggerListingGeneration` now: fetch product → reserve
→ (on any failure) release and rethrow → (on success) finalize. This also
fixes a real pre-existing bug: the old guard required `product.status ===
"ready_for_ai"`, but nothing ever moves a product back to that status once
it reaches `review` — so `unlockForRegeneration`'s effect (clearing
`reviewed_at`) had no way to ever be followed by an actual regeneration.
`gmcom_reserve_generation` makes that path real (status = `review` +
unreviewed package is now a legitimate reservation case), tested in
`app/actions.test.ts`.

**3. Durable version history.** New `listing_package_versions` table
(append-only, one row per superseded version, including that version's own
`source_facts` and `quality_summary` — the full audit trail, not just the
copy fields) plus `listing_packages.version`. Populated automatically
inside `gmcom_finalize_generation`, never touched by application code
directly.

**4. Review/generate race guards (`app/review/actions.ts`).**
`markReviewed` and `unlockForRegeneration` now go through
`gmcom_mark_reviewed` / `gmcom_unlock_for_regeneration`, which refuse
while `status = 'generating'` for that SKU. Closes a real race: a
`/review` page rendered *before* a regeneration started could otherwise
still submit "Mark Reviewed" mid-flight, silently getting overwritten by
the incoming version with no record it ever happened.

**5. Providers untouched.** `lib/ai/providers/*` still do nothing but
transport a prompt to a vendor and hand back raw text — no validation or
locking logic leaked into them. `prompt-builder.ts` remains the one
canonical prompt/schema source for every provider.

**6. Tests (`vitest`, new dependency).** 26 tests across 4 files, all
passing:
- `lib/ai/validation.test.ts` (12) — valid input accepted; malformed JSON,
  missing fields, wrong types, out-of-range values, unexpected extra
  fields, and invalid enum values all rejected; confirms no raw provider
  text ever appears in a thrown error's message.
- `lib/ai/listing-generator.test.ts` (4) — happy path makes exactly 2
  provider calls; an invalid first draft aborts after 1 call (never wastes
  the second, paid call); an invalid review+revise envelope is rejected;
  unparseable JSON is rejected.
- `app/actions.test.ts` (6) — happy path reserves/generates/finalizes;
  ineligible product refused with zero provider calls; already-reviewed
  package refused with zero provider calls; unlocked regeneration
  succeeds and increments `version`; **two concurrent calls for the same
  SKU produce exactly one provider call** (the core "≤1 paid call"
  acceptance criterion); a failed generation releases the lock and a
  retry immediately succeeds.
- `app/review/actions.test.ts` (4) — `markReviewed`/`unlockForRegeneration`
  succeed normally, refuse while `status = 'generating'`.

`npm run build` is clean (TypeScript strict mode, all routes compile).

## Verification

**Tested, not just written:**
- All 26 unit tests pass (`npm test`).
- `npm run build` compiles cleanly with no type errors.
- The SQL was carefully re-read end-to-end for correctness (row-lock
  ordering, null-safe `IS DISTINCT FROM` token comparisons, `GET
  DIAGNOSTICS` row-count checks, idempotency of every statement) but —
  see below — **not executed against a real database this session.**

**NOT tested — genuinely outstanding, and the most important thing for
whoever picks this up next to know:**
- **The SQL migration has not been applied to the real Supabase project.**
  Claude has no way to execute arbitrary SQL against it in this
  environment (only `SUPABASE_URL`/`SUPABASE_SERVICE_ROLE_KEY` — REST/RPC
  access to functions that already exist, not a way to create them). Per
  the issue's own requirement, the full migration is provided below as
  one copy-and-paste block for Phil to run in the Supabase SQL editor.
- Until that migration runs, `AI_PROVIDER=openai` or `anthropic` +
  clicking Generate in the real app **will fail** (the `gmcom_*` functions
  won't exist yet) — this is expected, not a regression; `AI_PROVIDER`
  defaults to `mock` so nothing paid is at risk from this gap.
- Real Postgres-level row locking (the actual atomicity guarantee behind
  "concurrent Generate produces ≤1 paid call") was **not** exercised
  against a live database — `app/actions.test.ts`'s concurrency test uses
  an in-memory fake of the three `gmcom_*` RPC functions' documented
  contract, which proves the *application's* orchestration logic (reserve
  before calling, refuse on loss, release on failure) is correct given
  that contract, but doesn't prove Postgres itself honors it. The `SELECT
  ... FOR UPDATE` pattern is standard and well-understood, but per this
  project's own standard ("clearly distinguish work that was tested from
  work that was only written or reviewed"), this is disclosed as reviewed
  code, not verified behavior.
- No real generation was run against a live product this session (unlike
  GMCOM-007/008's handoffs) — blocked on the migration above.

## Decisions Made

- **Added `zod` as a dependency** for runtime schema validation. This is
  the literal mechanism the issue's requirement #1 asks for ("Add runtime
  schema validation"), not unrelated infrastructure — a hand-rolled
  validator would just be a worse version of the same thing.
- **Added `vitest`** as the project's first test runner (also a new
  dependency), for the same reason — requirement #8 explicitly asks for
  "focused tests," and none existed before this issue. Chosen over Jest
  for a lighter footprint on a project with no React-rendering tests to
  run (no `next/jest` setup needed) — everything tested here is pure
  server-side logic.
- **Generation locking uses a `generating` product status + three Postgres
  functions (RPC), not just conditional `UPDATE ... WHERE` calls from
  Supabase-js.** The regeneration-eligibility check genuinely spans two
  tables (`products.status` and `listing_packages.reviewed_at`), which
  can't be expressed as a single atomic Supabase-js `.update().eq()` call.
  A Postgres function with `SELECT ... FOR UPDATE` was the correct tool for
  real cross-table atomicity, not a workaround.
- **A 15-minute stale-lock reclaim window**, rather than a background
  sweeper job. This is a single small internal tool with realistically one
  or two concurrent users — a sweep job would be exactly the kind of
  "unrelated infrastructure" the issue's boundaries warn against. The
  reclaim check inside `gmcom_reserve_generation` costs nothing extra and
  covers the real failure mode (a crashed request) without new moving
  parts.
- **Fixed the pre-existing `ready_for_ai`-only regeneration guard** rather
  than leaving it in place. This isn't scope creep or a UI redesign —
  it's a correctness bug directly inside what GMCOM-009 was asked to
  harden ("Server enforces atomic unlock-before-regeneration" is an
  explicit acceptance criterion, and the old code made that path dead and
  unreachable regardless of locking). No UI element changed; this was
  purely a server-side eligibility check.
- **Editing a package while a regeneration is in flight
  (`saveListingEdits`) was deliberately left unguarded.** The issue's
  "Required Work" list names protecting *reviewed* packages and the
  review/generate race specifically, not edits-during-generation. Since
  `status` leaves `'review'` the instant a regeneration is reserved, the
  product also disappears from `/review`'s query on any fresh page load —
  the same narrow stale-page window as the mark-reviewed race, but for
  edits instead. If it happens, nothing is lost: `gmcom_finalize_generation`
  archives whatever the latest edited content is at archive time before
  overwriting it. Flagging this as a known, accepted residual risk rather
  than silently expanding scope to guard it too.

## SQL Migration — copy and paste into the Supabase SQL editor

This is the exact block appended to `supabase/schema.sql` (safe to run the
whole file instead if preferred — every statement here is idempotent).

```sql
create extension if not exists pgcrypto;

alter table products
  add column if not exists generation_started_at   timestamptz,
  add column if not exists generation_lock_token    uuid,
  add column if not exists generation_prior_status  text;

alter table products drop constraint if exists products_status_check;
alter table products add constraint products_status_check
  check (status in ('intake', 'ready_for_ai', 'generating', 'review', 'published', 'archived'));

alter table listing_packages
  add column if not exists version integer not null default 1;

create table if not exists listing_package_versions (
  id                 bigserial primary key,
  sku                text not null references products(sku) on delete cascade,
  version            integer not null,
  source_system      text not null,
  source_record_id   text not null,
  proposed_title     text not null,
  short_title        text,
  description        text not null,
  sales_summary      text not null,
  tags               text[] not null default '{}',
  category           text,
  price              numeric,
  care_details       text,
  source_facts       jsonb not null default '{}'::jsonb,
  quality_summary    jsonb not null default '{}'::jsonb,
  model              text not null,
  generated_at       timestamptz not null,
  reviewed_at        timestamptz,
  superseded_at      timestamptz not null default now()
);

create index if not exists listing_package_versions_sku_idx
  on listing_package_versions (sku, version desc);

create or replace function gmcom_reserve_generation(p_sku text)
returns table(ok boolean, reason text, previous_status text, lock_token uuid)
language plpgsql
as $$
declare
  v_status      text;
  v_started_at  timestamptz;
  v_prior       text;
  v_reviewed_at timestamptz;
  v_token       uuid;
begin
  select p.status, p.generation_started_at, p.generation_prior_status
    into v_status, v_started_at, v_prior
    from products p
   where p.sku = p_sku
   for update;

  if not found then
    return query select false, 'not_found', null::text, null::uuid;
    return;
  end if;

  if v_status = 'generating' then
    if v_started_at is not null and v_started_at > now() - interval '15 minutes' then
      return query select false, 'already_generating', v_status, null::uuid;
      return;
    end if;
  elsif v_status = 'ready_for_ai' then
    v_prior := v_status;
  elsif v_status = 'review' then
    select lp.reviewed_at into v_reviewed_at from listing_packages lp where lp.sku = p_sku;
    if v_reviewed_at is not null then
      return query select false, 'already_reviewed', v_status, null::uuid;
      return;
    end if;
    v_prior := v_status;
  else
    return query select false, 'not_eligible', v_status, null::uuid;
    return;
  end if;

  v_token := gen_random_uuid();

  update products
     set status = 'generating',
         generation_started_at = now(),
         generation_lock_token = v_token,
         generation_prior_status = v_prior,
         updated_at = now()
   where sku = p_sku;

  return query select true, 'reserved', v_prior, v_token;
end;
$$;

create or replace function gmcom_finalize_generation(
  p_sku               text,
  p_lock_token        uuid,
  p_source_system     text,
  p_source_record_id  text,
  p_proposed_title    text,
  p_short_title       text,
  p_description       text,
  p_sales_summary     text,
  p_tags              text[],
  p_category          text,
  p_price             numeric,
  p_care_details      text,
  p_source_facts      jsonb,
  p_quality_summary   jsonb,
  p_model             text
)
returns boolean
language plpgsql
as $$
declare
  v_status       text;
  v_token        uuid;
  v_existing     listing_packages%rowtype;
  v_next_version integer;
begin
  select p.status, p.generation_lock_token into v_status, v_token
    from products p
   where p.sku = p_sku
   for update;

  if not found or v_status <> 'generating' or v_token is distinct from p_lock_token then
    return false;
  end if;

  select * into v_existing from listing_packages where sku = p_sku for update;

  if v_existing.sku is not null then
    if v_existing.reviewed_at is not null then
      raise exception 'listing_packages.% is already reviewed; refusing to overwrite', p_sku;
    end if;

    insert into listing_package_versions (
      sku, version, source_system, source_record_id, proposed_title, short_title,
      description, sales_summary, tags, category, price, care_details,
      source_facts, quality_summary, model, generated_at, reviewed_at
    ) values (
      v_existing.sku, v_existing.version, v_existing.source_system, v_existing.source_record_id,
      v_existing.proposed_title, v_existing.short_title, v_existing.description,
      v_existing.sales_summary, v_existing.tags, v_existing.category, v_existing.price,
      v_existing.care_details, v_existing.source_facts, v_existing.quality_summary,
      v_existing.model, v_existing.generated_at, v_existing.reviewed_at
    );
    v_next_version := v_existing.version + 1;
  else
    v_next_version := 1;
  end if;

  insert into listing_packages (
    sku, source_system, source_record_id, proposed_title, short_title, description,
    sales_summary, tags, category, price, care_details, source_facts, quality_summary,
    model, version, generated_at, reviewed_at, updated_at
  ) values (
    p_sku, p_source_system, p_source_record_id, p_proposed_title, p_short_title, p_description,
    p_sales_summary, p_tags, p_category, p_price, p_care_details, p_source_facts, p_quality_summary,
    p_model, v_next_version, now(), null, now()
  )
  on conflict (sku) do update set
    source_system    = excluded.source_system,
    source_record_id = excluded.source_record_id,
    proposed_title    = excluded.proposed_title,
    short_title       = excluded.short_title,
    description       = excluded.description,
    sales_summary     = excluded.sales_summary,
    tags              = excluded.tags,
    category          = excluded.category,
    price             = excluded.price,
    care_details      = excluded.care_details,
    source_facts      = excluded.source_facts,
    quality_summary   = excluded.quality_summary,
    model             = excluded.model,
    version           = excluded.version,
    generated_at      = excluded.generated_at,
    reviewed_at       = null,
    updated_at        = now();

  update products
     set status = 'review',
         generation_started_at = null,
         generation_lock_token = null,
         updated_at = now()
   where sku = p_sku;

  return true;
end;
$$;

create or replace function gmcom_release_generation(p_sku text, p_lock_token uuid)
returns boolean
language plpgsql
as $$
declare
  v_status text;
  v_token  uuid;
  v_prior  text;
begin
  select p.status, p.generation_lock_token, p.generation_prior_status
    into v_status, v_token, v_prior
    from products p
   where p.sku = p_sku
   for update;

  if not found or v_status <> 'generating' or v_token is distinct from p_lock_token then
    return false;
  end if;

  update products
     set status = coalesce(v_prior, 'ready_for_ai'),
         generation_started_at = null,
         generation_lock_token = null,
         generation_prior_status = null,
         updated_at = now()
   where sku = p_sku;

  return true;
end;
$$;

create or replace function gmcom_mark_reviewed(p_sku text)
returns boolean
language plpgsql
as $$
declare
  v_count integer;
begin
  update listing_packages lp
     set reviewed_at = now(), updated_at = now()
   where lp.sku = p_sku
     and exists (
       select 1 from products p where p.sku = lp.sku and p.status <> 'generating'
     );
  get diagnostics v_count = row_count;
  return v_count > 0;
end;
$$;

create or replace function gmcom_unlock_for_regeneration(p_sku text)
returns boolean
language plpgsql
as $$
declare
  v_count integer;
begin
  update listing_packages lp
     set reviewed_at = null, updated_at = now()
   where lp.sku = p_sku
     and exists (
       select 1 from products p where p.sku = lp.sku and p.status <> 'generating'
     );
  get diagnostics v_count = row_count;
  return v_count > 0;
end;
$$;
```

## Remaining Work

- **Phil: run the SQL block above (or the whole `supabase/schema.sql`
  file) in the Supabase SQL editor for the GM Commerce project.** Nothing
  in this issue works against the real app until this runs.
- After that, a real generation (and ideally a real regeneration, to
  exercise `listing_package_versions` for real) against a live product
  would close the "not tested against real Supabase" gap above — the same
  discipline as GMCOM-007/008's real-product verification. Not done this
  session; flagging as the natural first check for whoever picks up the
  next issue, or a short follow-up if Phil wants it confirmed before
  moving on.
- Update GitHub Issue #8 to closed/done — no GitHub API access available
  to Claude in this environment.

## Blockers or Risks

- No GitHub API access for closing Issue #8 directly.
- No way to execute the migration or verify against the live Supabase
  project in this session (see Verification above) — this is the one
  real gap in an otherwise complete, tested implementation.
- If the Postgres functions above ever need to be granted execute
  permission explicitly (some Supabase project configurations restrict
  function execution more tightly than table access) and the app's
  `SUPABASE_SERVICE_ROLE_KEY` calls start failing with a permission error
  after the migration runs, that would need a `grant execute on function
  gmcom_... to service_role;` follow-up — flagging as a possibility, not
  a known issue, since the existing schema.sql never needed explicit
  grants for tables either.

## Recommended Next Action

**GMCOM-010 (Issue #9) — Establish migrations, CI, and safe operational
baseline.** Note: while this session's work was in flight, ChatGPT (the
project manager) pushed three commits to this repo defining a *different*
next step than this handoff originally recommended — GMCOM-007/008's
"Shopify next" recommendation is now superseded. See
`handoffs/2026-08-02-photo-pipeline-priority.md`, `PHOTO_PIPELINE_SCOPE.md`,
and the "Photos are a first-class commerce asset" entry in `DECISIONS.md`:
photos are now a required pre-publishing stage (GMCOM-011, Issue #10), and
the explicit sequencing is **GMCOM-009 → GMCOM-010 → GMCOM-011 (photos) →
Shopify**, not GMCOM-009 → Shopify.

GMCOM-010 (already filed, [Issue #9](https://github.com/HydraCoreSystems/gm-commerce-hq/issues/9))
was scoped specifically not to overlap this issue's work: versioned
Supabase migration files (this session's own migration was necessarily a
single hand-authored copy-paste SQL block for Phil to run manually — the
exact gap GMCOM-010 exists to close for every future schema change), a CI
workflow, a pinned Node version, safe structured server logging, server-
action input validation and photo-path traversal protection, and a
deployment/runbook document. It explicitly excludes AI generation/
concurrency/versioning logic (this issue's own territory), Shopify/Etsy,
and authentication.
