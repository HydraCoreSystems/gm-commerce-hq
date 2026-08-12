# Decision Log

## 2026-07-31 — GitHub is the coordination hub

**Decision:** Use `HydraCoreSystems/gm-commerce-hq` as the permanent project-management headquarters and source of truth for GM Commerce.

**Reason:** AI conversations and usage windows are temporary. GitHub provides persistent documentation, issues, branches, commits, and handoffs that multiple contributors can use.

## 2026-07-31 — Repository remains private

**Decision:** Keep the headquarters repository private.

**Reason:** It may contain internal business processes, architecture, future strategy, operational weaknesses, and integration details. Claude, Copilot, and ChatGPT can use authenticated GitHub access rather than requiring public visibility.

## 2026-07-31 — Minimal human intervention is a core requirement

**Decision:** GM Commerce will optimize not only for speed and inventory accuracy but also for the fewest responsible human touches.

**Reason:** Gathering Moss needs leverage comparable to a larger company without adding a large staff.

## 2026-07-31 — Skrybix remains separate and authoritative for plants

**Decision:** Plant identities remain in Skrybix. The Product SKU Generator handles non-plant products during the initial phase.

**Reason:** Skrybix already has Mother_ID and Cutting_ID generation. Replacing or duplicating it would delay useful delivery and introduce inventory risk.

## 2026-07-31 — Product SKU Generator local JSON is interim

**Decision:** Accept local JSON counters and history for the single-user starter utility, but do not treat them as the permanent central commerce database.

**Reason:** The approach is appropriately simple now, but GM Commerce will eventually need a durable shared source of truth for products, listing state, and marketplace identifiers.

## 2026-07-31 — Shopify comes before Etsy

**Decision:** Prove the shared listing-draft model and Shopify draft-publishing workflow before implementing Etsy publication.

**Reason:** This keeps the first end-to-end workflow smaller and avoids designing two marketplace integrations before the central model is validated.

## 2026-08-01 — Existing SKUs enter GM Commerce from two source systems

**Decision:** GM Commerce never requires manual SKU re-entry. Plant records are selected in Skrybix; non-plant records are selected in Product SKU Generator. Each source sends the already-assigned SKU and source record into GM Commerce.

**Reason:** The SKU is the permanent business identity. Re-keying it would add labor and create avoidable inventory errors.

## 2026-08-01 — GM Commerce is a selection-driven pipeline, not a bulk importer

**Decision:** GM Commerce's `products` table only ever contains SKUs a
human has explicitly selected from Skrybix or the Product SKU Generator.
There is no bulk-import path and no screen anywhere that accepts a
manually-typed SKU.

**Reason:** The original GMCOM-002 issue described a read-only display
seeded by an import script; ChatGPT revised this mid-implementation
specifically to guarantee "GM Commerce must never require manual SKU
entry," and to model the real workflow (select → intake → review →
publish) rather than a passive mirror of both source systems' full
inventories. (Same principle as "Existing SKUs enter GM Commerce from two
source systems" above, recorded independently the same day.)

## 2026-08-01 — GM Commerce owns the downstream commerce workflow

**Decision:** After intake, GM Commerce owns photo-folder status, notes, AI processing state, listing drafts, review state, destination choices, marketplace relationships, and publishing state. Skrybix and Product SKU Generator remain authoritative only for their source identities and SKU creation.

**Reason:** This preserves clear source boundaries while giving the commerce workflow one durable home.

## 2026-08-01 — GM Commerce's Supabase project is a separate account

**Decision:** GM Commerce's database lives in a new Supabase account
(project ref `wcrcllhvgbhykbonopzx`) created under a co-owner's email, not
Phil's existing personal Supabase account that hosts Skrybix.

**Reason:** Phil's explicit choice, to keep GM Commerce's usage/billing
separate from his personal account.

## 2026-08-01 — GM Commerce creates its own photo folders for Skrybix-origin products

**Decision:** For plant products (which Skrybix doesn't organize into
photo folders at all), GM Commerce creates a folder named by SKU under the
same shared photo root the Product SKU Generator already writes into —
not a second, GM-Commerce-only photo root.

**Reason:** Avoids two parallel photo-folder locations for what should be
one conceptual "all product photos" root; the root is overridable via
`GM_COMMERCE_PHOTO_ROOT` if that ever needs to change.

## 2026-08-01 — One canonical Listing Package supports every sales option

**Decision:** GM Commerce is marketplace-agnostic. AI creates one canonical Listing Package for a SKU. Output adapters then use that package for Shopify, Etsy, both platforms, copy-and-paste sales sheets for Palm Street or Overgrown Oasis, and future sales channels.

**Reason:** Listing intelligence should be created once rather than rewritten independently for each channel. Some channels have APIs; others require a human-friendly document or copy-ready view. Both are output targets from the same package.

## 2026-08-01 — Human review remains before external publication

**Decision:** AI-generated copy and prepared photos enter a review queue. Phil or Crystal chooses the destination channel or channels and approves the result before publication during Version 1.

**Reason:** The system should remove repetitive work while preserving owner control over public listings and sales decisions.

## 2026-08-01 — Every source system uses the same commerce handoff pattern

**Decision:** Every authoritative source system must expose the same conceptual handoff to GM Commerce: a human selects an existing record, the source persists that selection, the source exposes only selected records through an authenticated interface, and GM Commerce consumes the handoff without creating or modifying the source identity.

**Reason:** Skrybix and Product SKU Generator manage different product types, but GM Commerce should receive both through one predictable intake pattern. A common handoff reduces duplicated logic, prevents manual SKU re-entry, and keeps each source authoritative for its own identities.

## 2026-08-01 — The interface must teach the workflow

**Decision:** GM Commerce must have a best-in-class, visually polished GUI that makes the next correct action obvious without requiring technical knowledge, written instructions, or institutional memory. The interface must be usable by Phil, Crystal, and future employees with minimal training.

**Reason:** GM Commerce is intended to become an operational system used repeatedly by different people. Its value depends not only on automation, but on reducing uncertainty, errors, and training time. The UI should communicate status, readiness, blockers, and the next action through layout, hierarchy, language, and feedback rather than expecting the user to understand the underlying architecture.

## 2026-08-02 — AI generation is reserved atomically before any paid call

**Decision:** `triggerListingGeneration` must atomically reserve a product
(a `generating` status, claimed via a Postgres function using `SELECT ...
FOR UPDATE`) before either paid provider call in the Listing Quality
Engine's pipeline is made. Reservation, not a post-hoc status update, is
the mechanism that prevents duplicate paid calls under concurrent Generate
clicks.

**Reason:** GMCOM-009's explicit acceptance criterion. The prior
implementation only flipped `products.status` at the very end of
generation, so two concurrent calls (a double-click, two tabs) could both
pass the eligibility check and both spend real provider calls before
either write landed.

## 2026-08-02 — Every regeneration is versioned, never silently overwritten

**Decision:** Any time a generation succeeds and a `listing_packages` row
already exists for that SKU, the row being replaced is archived into an
append-only `listing_package_versions` table (including its own
`source_facts` and `quality_summary`) inside the same transaction as the
new write. Application code never deletes or overwrites version history.

**Reason:** GMCOM-009's explicit requirement ("Add durable package
versioning to preserve prior versions during regeneration"). Regeneration
is real now (see the fix below) — losing the previous version silently on
regenerate would be a real data-loss risk once that path is actually used.

## 2026-08-02 — Regeneration requires the product to have reached review and been unlocked, not "ready_for_ai" again

**Decision:** A product is eligible for AI generation when either (a) its
status is `ready_for_ai` (first generation) or (b) its status is `review`
and its `listing_packages` row is unreviewed (`reviewed_at IS NULL` —
i.e., explicitly unlocked via `unlockForRegeneration`). Products never
return to `ready_for_ai` after reaching `review`.

**Reason:** The original GMCOM-007 guard only allowed generation from
`ready_for_ai`, which made `unlockForRegeneration`'s effect (clearing
`reviewed_at`) unreachable — nothing could ever follow it with an actual
regeneration. This closes that gap without any UI change: the eligibility
check is server-side only.

## 2026-08-02 — AI provider access is abstracted behind a swappable interface

**Decision:** GM Commerce never depends directly on any single AI vendor's SDK. A provider interface (`lib/ai/provider-types.ts`) sits between the Listing Generator and every vendor; the active provider (Mock, OpenAI, or Anthropic) is selected entirely through the `AI_PROVIDER` environment variable, never by editing code. Exactly one canonical prompt-building pipeline (`lib/ai/prompt-builder.ts`) and one Listing Package schema are shared by every provider — content integrity, tone, and business rules live there, never duplicated inside a provider implementation.

**Reason:** Phil's explicit architectural decision, made before any provider was implemented: GM Commerce must never be locked to one AI vendor. OpenAI was chosen as the first real provider because ChatGPT already serves as this project's coordinating PM, OpenAI's platform covers text/vision/structured-output/future capabilities in one place, and keeping Anthropic wired as a first-class option (not a fallback) gives a strong default today without a long-term vendor lock-in.

## 2026-08-02 — "Standardized commerce backgrounds" means a mat, not a background swap

**Decision:** GM Commerce's photo pipeline standardizes a non-square
photo's canvas by padding it onto a solid, configurable brand-color mat
(default a warm cream). It does not segment the product out of its real
photographed scene and composite a different background behind it.

**Reason:** True background replacement is a generative/object-removal
operation, and both `PHOTO_PIPELINE_SCOPE.md` and `standards/
PHOTO_STANDARD.md` require exactly that category of edit to be explicit,
previewed, and human-approved, and kept out of Version 1's default path.
A mat delivers the requested "premium, warm, clean, consistent" framing
without ever touching the real, truthful content of the photograph.

## 2026-08-02 — Photo alt text reuses the existing AI Provider abstraction, text-only

**Decision:** Alt text for approved photo derivatives is generated
through the same `AIProvider` interface and `AI_PROVIDER` switch already
built for Listing Packages — one small, batched (per photo set, not per
image), factual, runtime-validated request — rather than a new AI
integration or a vision-capable call.

**Reason:** Avoids introducing new AI infrastructure/cost for a
Version-1-scoped feature; batching keeps cost proportionate to the
feature's value. Text-only (facts, not pixels) keeps alt text honestly
limited to what GM Commerce actually knows, the same content-integrity
discipline already governing Listing Package generation.

## 2026-08-02 — Photo processing concurrency uses guarded conditional updates, not the GMCOM-009 token pattern

**Decision:** `photo_sets.status` transitions (pending → processing →
needs_review → approved) are guarded with plain conditional updates
(`.eq("status", expected)`), not the reserve/finalize/release lock-token
mechanism built for AI generation.

**Reason:** GMCOM-009's heavier mechanism exists specifically to prevent
duplicate *paid* AI calls. Photo processing is local and unpaid — the
real risk is wasted CPU and confusing UI state, which a guarded
conditional update already prevents. Matching engineering weight to
actual risk, not applying the same pattern everywhere by default.

## 2026-08-02 — Photos are a first-class commerce asset

**Decision:** GM Commerce must prepare, standardize, optimize, and review product photos before any marketplace draft is created. Photo work is not a secondary attachment step; it is part of the canonical commerce package and must follow Gathering Moss photo standards.

The photo pipeline must preserve originals, create standardized derivatives, avoid altering the product truthfully represented in the image, support human review, and produce marketplace-ready assets with known dimensions, orientation, crop, file type, quality, order, and status.

**Reason:** Listing copy alone cannot produce strong sales. Product photos are often the first and most influential part of a listing, and inconsistent or poorly prepared images would undermine the quality of the entire GM Commerce workflow. Shopify and Etsy publishing should consume approved photo assets rather than raw uploads.

## 2026-08-02 — Claude may self-approve real content only as an explicit, one-time, in-the-moment exception

**Decision:** For GMCOM-012's real end-to-end verification, Claude marked
`HY-LOB01-C04`'s Listing Package reviewed and its photo set approved
itself, rather than waiting for Phil or Crystal. This was explicitly
authorized by Phil in chat (offered as one of three options; he chose it,
with ChatGPT's concurring recommendation to treat it as a development-only
exception) — not a decision Claude made unilaterally. The standing rule
("Human review remains before external publication," recorded 2026-08-01
above) is unchanged: this was a single authorized exception for this
verification, not a new standing practice. Claude will not self-approve
real content again without equally explicit, in-the-moment authorization
each time.

**Reason:** Producing a genuine end-to-end verification (not a scratch/
fake product) required a real reviewed listing and a real approved photo
set, and none existed. The alternative — using throwaway test data instead
— would have proven the mechanics but not actually exercised the real
production path Phil explicitly asked to see working.

## 2026-08-02 — Shopify auth supports both a static token and a client-credentials exchange

**Decision:** `lib/shopify/real-client.ts` accepts either a static
long-lived Admin API token (`SHOPIFY_ADMIN_API_ACCESS_TOKEN`, the classic
custom-app `shpat_...` token) or Client ID + Secret
(`SHOPIFY_CLIENT_ID`/`SHOPIFY_CLIENT_SECRET`), the latter exchanged for a
short-lived (~24h) access token via `grant_type=client_credentials` and
automatically re-exchanged before it expires.

**Reason:** Discovered firsthand, not designed speculatively: Gathering
Moss's actual Shopify store/plan does not offer the legacy static-token
"custom app" flow at all — only the Dev-Dashboard client-credentials path
was available. Supporting both modes means this code works regardless of
which flow a given store's plan offers, and the token-refresh logic means
the client-credentials path doesn't silently stop working a day after
setup.

**Operational note for whoever manages Shopify app credentials next:**
changing an app's requested access scopes does NOT retroactively grant
them to an already-installed instance of that app — the token exchange
keeps succeeding, but every product-related call fails with silently-empty
granted scopes until the app is uninstalled and reinstalled (or otherwise
re-authorized) on the store. Cost real troubleshooting time to discover
during GMCOM-012; recording it here so it isn't rediscovered the hard way
again.

## 2026-08-03 — Phase B slice 1: RLS is real but not load-bearing for today's only caller; the TypeScript repository layer is

**Decision:** The 18 canonical entity tables (`gm-commerce` PR #5,
`supabase/migrations/20260803030000_phase_b_slice1_canonical_entities.sql`)
have RLS enabled and a real, environment-scoped `SELECT` policy for the
`authenticated` role, gated on a session GUC (`app.gmcom_caller_environment`)
that is fail-closed when unset. But `service_role` — the only role
`gm-commerce` actually uses (`lib/supabase.ts` holds only the service-role
key; there is no browser-side Supabase Auth user yet) — has Postgres
`BYPASSRLS`, so RLS structurally cannot gate it. The actual environment-
isolation enforcement for today's app is `lib/canonical/repository.ts`'s
`CanonicalEntityRepository`.

**Reason:** Two options existed: (a) claim RLS enforces isolation and
leave the service-role gap unstated, or (b) build the TypeScript-layer
enforcement as the real mechanism and state plainly that RLS is
defense-in-depth for a future caller, not today's protection. (a) would
have repeated exactly the kind of overstated-safety mistake §25 exists to
prevent (the `HY-LOB01-C04` "real data" mischaracterization this whole
reset traces back to) — a security claim that sounds stronger than what's
actually enforced. (b) was chosen. If a browser-side Supabase Auth path is
ever added for this app, the RLS policy is already live and correct and
needs no rework — only then would it become load-bearing.

**Update (2026-08-03, same day, correction pass after Codex's independent
review of PR #5):** Codex found the original version of this entry
correctly *named* the gap but didn't go far enough — the "load-bearing"
application path itself wasn't yet bound or protected against accidental
misuse. Three concrete fixes landed in response, and the trust boundary is
now stated precisely rather than just directionally:

1. **`CanonicalEntityRepository` is now the only door in, verified by
   audit, not just by convention.** `grep`ing `app/`, `components/`, and
   `lib/` for any reference to a `canonical_*` table name outside
   `lib/canonical/` itself returns nothing — no other application code has
   ever queried these tables directly. This was true before the
   correction pass too, but wasn't previously stated as a checked fact.
2. **Environment is now bound per repository instance at construction,
   never accepted per call.** The original version of this class took
   `callerEnvironment` as a parameter on every read — which is
   caller-*selected* scope, not caller-*bound* authorization; any call
   site could simply request `production`. Now, one `Environment` is
   bound at construction via `createCanonicalEntityRepository()`, which
   resolves it from trusted config (`GMCOM_ENVIRONMENT`, fail-closed on
   missing/invalid — see `environment-config.ts`), never from a request
   parameter. No method on a constructed instance accepts a different
   environment as an override.
3. **`createEntity` can no longer be used to overwrite its own protected
   columns.** The original version built the stored row as `{ id,
   ...recordContextColumns, ...fields }` — entity-specific `fields` spread
   *last*, so a caller (or a bug) could silently overwrite `id`,
   `environment`, approval/eligibility/retention, or timestamp columns.
   Fixed with two independent layers: an explicit reject
   (`rejectProtectedColumnOverrides`, checked first, adversarially tested
   against each of the 18 protected columns individually) and a
   corrected, protected-columns-spread-last row construction as a second,
   structurally independent backstop.

The trust boundary, stated precisely as of this correction: RLS is real,
enabled, and correct defense-in-depth for a future non-service-role
caller — but is not what protects today's data. What protects today's
data is `CanonicalEntityRepository` being the sole code path with table
access, each instance being bound to one environment at construction from
trusted config, and `createEntity` being unable to have its protected
columns overridden by caller-supplied `fields`.

**Second update (2026-08-03, same day, Codex's targeted re-review found
three further gaps in the above):**

1. **Point 2 above ("environment is bound per instance") was real, but the
   class's constructor was still public and unrestricted — any runtime
   code could bypass `createCanonicalEntityRepository()` and call `new
   CanonicalEntityRepository(supabase, "production")` directly, choosing
   its own environment. "Prefer the factory" was a comment, not an
   enforced boundary.** Fixed by actually separating the modules, not just
   documenting an intent: the real class moved to
   `lib/canonical/internal/repository-impl.ts`
   (`CanonicalEntityRepositoryImpl`), and `lib/canonical/repository.ts` —
   the module application code actually imports — re-exports it as a
   **type only** (`export type { CanonicalEntityRepositoryImpl as
   CanonicalEntityRepository }`). TypeScript erases type-only exports at
   compile time, so there is no runtime value of that name reachable
   through `repository.ts` at all; `new CanonicalEntityRepository(...)`
   does not compile for code that only imports the public module, and a
   runtime check (`lib/canonical/trust-boundary.test.ts`) confirms the
   compiled module's namespace has no such property. The one value export
   that produces an instance, `createCanonicalEntityRepository(supabase)`,
   takes no environment argument — there's nothing for a caller to
   override even if they wanted to. Tests get their own construction path,
   `lib/canonical/testing.ts`'s `__createRepositoryForTesting`, itself
   audited (grep, same mechanism as point 1's original audit) as never
   imported by `app/` or `components/`. This is now **structurally** the
   only path, not conventionally the recommended one.
2. **`createEntity` could fabricate genuine owner approval and true
   eligibility flags with no real `OwnerDecision` behind them.** The
   original fail-closed defaulting (this document's earlier ULID/RLS
   entries) only guaranteed *omitted* fields defaulted safely — it never
   stopped a caller from *explicitly* supplying `ownerApproval.approvalState:
   "genuine"` plus a fabricated reviewer/timestamp/decision-ref, or flipping
   an eligibility flag `true`, through ordinary entity creation. Fixed on
   two levels: `CreateEntityParams.context` is now typed as
   `CreatableRecordContextInput`, which removes `"genuine"` from the
   `approvalState` union entirely and narrows every eligibility flag's
   type to the literal `false` — a caller cannot even write the
   privileged values and have the code type-check. `rejectPrivilegedCreationContext()`
   is the runtime backstop for an `as any` bypass, adversarially tested
   against genuine approval and each of the three eligibility flags
   individually. A future migration/backfill needing to preserve
   pre-existing genuine approval is explicitly out of scope for slice 1
   and will need its own separate, privileged, audited operation — not a
   relaxation of this check.
3. **`environment` defaulted to `'production'` on all 18 tables** — an
   insert (direct SQL, a future bug, anything that reached the database
   without going through the bound-environment repository) that omitted
   `environment` silently became production data, the exact opposite of
   fail-closed and a direct contradiction of "environment is set
   explicitly at creation, never inferred." Fixed: the default is removed
   entirely; the column remains `NOT NULL` with no fallback, so an
   omitted `environment` now fails the insert outright. Live-verified in
   CI against real Postgres: an insert omitting `environment` fails with
   `not_null_violation`; the same insert with `environment` supplied still
   succeeds.

The trust boundary, restated precisely as of this second correction: RLS
remains defense-in-depth only. What protects today's data: (a)
`CanonicalEntityRepository` is the sole code path with table access,
structurally unreachable as a constructible value outside
`lib/canonical/internal/` and `lib/canonical/testing.ts`, both audited as
never imported by application code; (b) each instance is bound to one
environment at construction from trusted config, with no override
surface anywhere in its public methods; (c) `createEntity` cannot have
its protected columns overridden by `fields`, and cannot establish
genuine approval or true eligibility through `context` — both rejected,
not silently downgraded, and both unreachable at the type level for a
caller who doesn't bypass TypeScript; (d) `environment` itself is a
required column with no default, so even a write that bypasses the
repository entirely still cannot silently default into production.

## 2026-08-03 — ULID IDs are generated independently at both the Postgres and TypeScript layers, not shared via a library dependency

**Decision:** `gmcom_ulid()` (Postgres, `plpgsql`) and `lib/canonical/ulid.ts`'s
`generateUlid()` (TypeScript) are two separate implementations of the same
26-character Crockford-base32 ULID shape (48-bit ms timestamp + 80-bit
crypto-random entropy), rather than one being derived from or wrapping the
other, and rather than pulling in a `ulid` npm package.

**Reason:** The TypeScript repository layer (`lib/canonical/repository.ts`)
is the primary, sole ID-issuing path in normal application use — it always
supplies an explicit `id` on insert. `gmcom_ulid()` exists as a safety net
so a direct-SQL insert (discouraged by §1.1's database-change-control
policy, but not something the database itself can prevent) still gets a
compliant ID rather than an arbitrary or absent one. Keeping the two
implementations independent means a defect in one is not automatically a
defect in the other, and the TypeScript one can be unit-tested directly
(format, 10,000-call uniqueness, sortability) without needing a live
Postgres connection, which was not available in this session's environment
(no Docker daemon, no local Postgres) — the Postgres-side implementation's
own runtime behavior is instead verified live by CI's `schema-from-empty`
job, which does have real Postgres.

## 2026-08-03 — Canonical foreign keys are composite on (column, environment), not id alone

**Decision:** Every parent/child relationship among the 18 canonical
entity tables (`gm-commerce` PR #5) now uses a composite foreign key —
`foreign key (col, environment) references parent (id, environment)` —
instead of the original `references parent(id)`. Every canonical table
also carries a `unique (id, environment)` constraint, which is what makes
the composite FK possible (Postgres requires a unique constraint on
exactly the column tuple a composite FK references).

**Reason:** Codex's independent review of PR #5 found that the original,
id-only foreign keys let a row in one environment reference a parent row
in a different one — e.g. a `production` SKU could reference a `test`
ProductConcept — which is exactly the cross-environment leak §25 exists to
prevent, and it was enforced nowhere: not by the original id-only FK, and
not by application code (which never validated a parent's environment
before referencing it). A composite FK was chosen over a cross-table
CHECK constraint (which Postgres doesn't support directly) or a trigger:
it's declarative, enforced by Postgres's own constraint machinery rather
than custom trigger logic repeated across 17 relationships, and it's
NULL-safe by Postgres's default `MATCH SIMPLE` semantics — an optional,
unset reference (e.g. `InventoryItem` with no `sku_id`) remains exempt,
exactly like the plain FK it replaced. No new column was needed: every
canonical table already carried its own `environment` column, which now
doubles as the second half of every composite FK it participates in.
Live-verified in CI (`.github/workflows/ci.yml`'s `schema-from-empty` job)
against a real Postgres instance: a deliberately cross-environment insert
on both a required relationship (SKU → ProductConcept) and a nullable one
(InventoryItem → SKU) is confirmed to fail with `foreign_key_violation`,
and a same-environment insert on both is confirmed to still succeed.

## 2026-08-03 — Repository construction is bound by a private constructor + module-private runtime token, not a naming convention

**Decision:** `CanonicalEntityRepositoryImpl` (`gm-commerce`
`lib/canonical/internal/repository-impl.ts`) now has a TypeScript
`private` constructor that additionally checks a module-private runtime
`symbol` token (`CONSTRUCTION_TOKEN`, declared `const`, never exported).
The only way to obtain an instance, anywhere in the codebase, is the
static `CanonicalEntityRepositoryImpl.create(supabase)` method, which
resolves the bound environment internally from trusted configuration and
takes no environment argument. `lib/canonical/testing.ts` (a prior
round's environment-accepting test factory) is deleted; tests now go
through the exact same `createCanonicalEntityRepository(supabase)` path
as application code, temporarily controlling `process.env.GMCOM_ENVIRONMENT`
for the duration of a construction call.

**Reason:** Two prior rounds of this fix were each found insufficient by
Codex's review, in a useful, illustrative progression:

- **Round 1** bound environment per repository *instance* rather than
  accepting it per method call — real progress, but the class itself was
  still a plain public class, constructible anywhere with any environment.
- **Round 2** re-exported the class as a TYPE ONLY from
  `lib/canonical/repository.ts` (erased at compile time) and moved the
  real class to an `internal/` directory, auditing that `app/` and
  `components/` never imported it directly. This stopped *application
  code* from constructing an arbitrary-environment instance through the
  public module — but it was still a naming/import-location convention:
  the class itself remained a real, exported, constructible runtime value
  in `internal/repository-impl.ts`, and nothing stopped a future helper
  anywhere else in the tree (a different `lib/` subdirectory, a script, a
  job — none of which round 2's audit even scanned) from importing that
  exact file and calling `new` on it directly.
- **Round 3 (this entry)** replaces the naming convention with an actual
  construction boundary that holds regardless of which file attempts a
  bypass. The private constructor blocks the common case at compile time.
  The runtime symbol-token check is what makes it real against a
  *determined* bypass: TypeScript's `private` is erased at runtime, so a
  caller casting past it with `as any` could otherwise still call the
  compiled constructor — but they cannot supply the token, because a
  `symbol` is unique by identity, never by its description string. Even
  creating a second `Symbol("CanonicalEntityRepositoryImpl construction
  token")` with the exact same text does not produce a value equal to the
  real one; JavaScript's `Symbol()` guarantees this. The real token is
  never exported, attached to any object, or returned by any function —
  no code outside `internal/repository-impl.ts`'s own module scope has
  ever held a reference to it, so none can reproduce it.

Adversarially tested in `lib/canonical/internal/repository-impl.test.ts`,
which imports the real class directly (deliberately bypassing the
"don't import internal/" convention, to prove the boundary doesn't
depend on that convention being followed) and confirms: a guessed
2-argument constructor call throws; a forged token with a matching
description string throws; other guessed token values (`"production"`,
`{}`, `undefined`, `null`, `Symbol.for("production")`) all throw; and the
one real construction path, `CanonicalEntityRepositoryImpl.create(supabase)`,
succeeds and binds exactly the environment `resolveTrustedEnvironment()`
resolves, with no argument for any caller to have influenced.
`lib/canonical/trust-boundary.test.ts`'s import-location audit is kept
and broadened beyond `app/`+`components/` to `lib/` (excluding
`lib/canonical`'s own legitimate wiring), `test/`, `types/`, `supabase/`
— now explicitly documented as defense-in-depth on top of the real
mechanism, not the primary guarantee, since the guarantee no longer
depends on file location at all.

The trust boundary, restated precisely as of this third correction: RLS
remains defense-in-depth only. What protects today's data:
`CanonicalEntityRepositoryImpl` cannot be constructed by any code,
anywhere, with a caller-chosen environment, full stop — not because of
where a file lives, but because its own constructor refuses every caller
who cannot produce a value that exists only in its own module's closure.

## 2026-08-05 — Marketplace suitability resolves conflicts per marketplace key, not forced to a single marketplace winner

**Decision:** A subject may be suitable for multiple marketplaces independently. The marketplace-suitability recommendation service evaluates each marketplace separately; conflicts are resolved only within a single marketplace key (e.g., two claims disagreeing about "Shopify"). Distinct marketplaces (Shopify, Etsy, Amazon) are all surfaced as candidates rather than forced into a single winner.

**Reason:** A suitability recommendation that forces a single "best" marketplace would suppress valid multi-channel candidates. The claim model already supports independent suitability claims per marketplace, and the precedence machinery (owner override > verified > under_review > stale, with confidence/freshness tiebreakers) already resolves within-key conflicts. Preserving multi-marketplace candidates reflects the real business operation — Gathering Moss sells on multiple channels — without compromising the single-SelectionTrace-per-recommendation contract.

## 2026-08-05 — Photography recommendation reads vision claims + legacy photo-set state without creating intermediate photo-claim predicates

**Decision:** The Phase F Slice 5 photography recommendation service evaluates photography readiness by reading `vision.*` claims (from Phase D's vision provider contract) through the claim repository and reading `photo_sets`/`photo_assets`/`photo_derivatives` (from GMCOM-011's legacy photo pipeline) through direct Supabase queries. It does not create new claim predicates (e.g., `photo.hero_selected`, `photo.alt_text_complete`) to bridge the two layers. Each signal (hero, alt text, derivatives present, photo-set approved, vision analysis presence) is derived and surfaced in the recommendation value, not stored as an intermediate claim.

**Reason:** Creating photo-claim predicates would require a migration and broader architectural decisions about migrating GMCOM-011's legacy tables into the canonical claim model — work that belongs to the product reset's Phase B (canonical entities and claim model), not to Phase F's recommendation services. Reading both data sources directly follows the same pattern the compliance gate already uses (reading a derived outcome from a non-claim source) and avoids blocking the photography recommendation on a migration that has not been designed or approved. When the GMCOM-011-to-canonical bridge is eventually built, the photography service's data reading can be updated to consume photo claims rather than legacy table rows — but the recommendation value shape and the §18 SelectionTrace contract stay the same, so consumers of the recommendation are not broken by that future migration.

## 2026-08-05 — Photography recommendation reads hero from photo_derivatives.is_hero and derives blur/exposure/near-duplicate from vision claims, not from inspection_result columns

**Decision:** The Phase F Slice 5 brief specified `photo_sets.hero_photo_id` and `photo_assets.inspection_result` (JSONB with `blurScore`, `exposureMean`, `perceptualHash`) as data sources — these columns do not exist in the actual legacy schema. The implementation reads hero from `photo_derivatives.is_hero` (the real column) and derives blur/exposure/near-duplicate signals from vision claims (`unusableReason` for blur/exposure, `isDuplicateView` for near-duplicates) rather than inventing thresholds on non-existent inspection columns. Photo-set lookup is SKU-only (legacy tables have no environment column); ProductConcept subjects receive `not_applicable` photo-set signals while vision claims are still assessed.

**Reason:** The brief's column names were speculative, not verified against the actual schema. The implementation correctly adapted to what exists. Engineering the correct thing (read real columns, derive quality signals from already-persisted vision claims) is the correct adaptation. No data was invented, no thresholds were fabricated.

## 2026-08-05 — Merchandising live-CI precedence fixture uses verified-vs-under_review because Phase B capability gate blocks owner-override seeding

**Decision:** The Phase F Slice 7 live-Postgres CI job tests conflict resolution with verified-vs-under_review claims rather than owner-override claims. The Phase B capability gate (`rejectPrivilegedCreationContext`) fail-closes all owner-authority transitions, preventing insertion of a `canonical_owner_decisions` row from CI. Owner-override precedence is fully covered by unit tests (which use in-memory Claim fixtures, bypassing the DB gate).

**Reason:** The CI constraint is a Phase B architectural decision (privileged context cannot be fabricated), not a Slice 7 gap. Working around it would require relaxing the capability gate specifically for test infrastructure — a worse outcome than accepting that one precedence variant is unit-tested rather than integration-tested. All other precedence variants (verified-vs-under_review, confidence tiebreakers, freshness tiebreakers) are exercised in live Postgres.

## 2026-08-05 — SEO recommendation reads listing_packages.proposed_title and listing_packages.category (not title / product_type)

**Decision:** The Phase F Slice 6 brief specified `listing_packages.title` and `listing_packages.product_type` as data sources — these column names don't match the actual schema. The implementation reads `listing_packages.proposed_title` and `listing_packages.category` (the real column names) and does not read `listing_packages.seo_title` or `listing_packages.seo_description` (which exist only in the undocumented live-database migration).

**Reason:** Same pattern as F5's schema-correction: the brief's column names were speculative, the implementation correctly adapted to reality.

## 2026-08-05 — Photography signals are always emitted in a fixed documented order

## 2026-08-04 — "Current" on an append-only versioned table is derived from the un-superseded chain head, never a mutable status flag

**Decision:** Any canonical table whose rows must never be mutated or
deleted after insert (an append-only, fully-versioned history table) must
determine "the current row" by querying for the row that no other row's
`supersedes_<x>_id`-style pointer targets yet — never by adding a mutable
`effective_state`/`is_current`/`active` flag that gets flipped on the old
row when a new one is inserted. A guard trigger on insert additionally
rejects (does not silently rewrite) any insert that would branch the chain,
i.e. two different rows both claiming to supersede the same predecessor,
serialized via a row lock on the predecessor during that check.

**Reason:** Established by Phase C Slice 5 (Freshness, Revalidation, and
Promotion Gates — `gm-commerce` PR #14, merge commit
`b205ea79b325e96b301054c96f941a462b41ad10`) for
`canonical_freshness_policies` and its `gmcom_current_freshness_policy(...)`
function (`supabase/migrations/20260804030000_phase_c_slice5_freshness_revalidation.sql`).
A mutable "active" flag on an otherwise append-only table is a real
integrity hazard: it can be flipped incorrectly, flipped twice, or drift out
of sync with the actual insert history, and nothing then prevents "current"
from silently pointing at a stale or wrong row. Deriving "current" purely
from chain structure (no successor points at me yet) makes an incorrect
"current" state structurally impossible rather than merely policed by
application discipline — the same reasoning already applied to
environment-binding and to the private-constructor/token repository-
construction boundary above. This is recorded here as a reusable
architectural pattern: any future append-only versioned canonical table
(not just freshness policies) should default to this "derive current from
the un-superseded chain head" shape rather than reintroducing a mutable
current-state flag.

## 2026-08-12 — Exact-version package snapshots and current-version routing safety

**Decision:** A canonical CommercePackage is assembled from an **exact, versioned snapshot** of the approved listing content (content assembly, PR #60), and destination routing/review operate on that exact version. This preserves the exact reviewed version for every destination write and keeps the historical record immutable.

**Reason:** Regeneration supersedes the old package version rather than overwriting it, so the review-decision and routing paths must reference a specific version. This is the architecture that Phase I delivered (exact-version content assembly + immutable version history). **Finding D1/High (current-version invariant and regeneration safety) is resolved** by Phase 0 Slice 2 (PR #70, merge `9cc5f6a`); see the "Current-version authority is database-enforced" entry below.

## 2026-08-12 — Current-version authority is database-enforced (Phase 0 Slice 2)

**Decision:** Currentness of a canonical CommercePackage is structurally derived and database-enforced — not merely a UI status. Exactly one non-superseded operational package exists per (environment, SKU), enforced by the partial unique index `canonical_commerce_packages_one_current_per_sku_env` with a guard trigger that serializes materialization on the parent `canonical_skus` row (`SELECT ... FOR UPDATE`). Under the lock the current version is re-read, so concurrent replacement inserts deterministically converge to the **highest** package version (lock order: `canonical_skus → canonical_commerce_packages → canonical_destination_requests`, consistent with enqueue/approval). Authority transfers only when the replacement package successfully materializes: a committed regeneration keeps the previous package authoritative until the bridge inserts the new version, whose guard trigger atomically supersedes the old package — no supersession gap. Lower versions are preserved as historical/`superseded` and never displace the current package; equal-version bridge replay is idempotent; the partial unique index remains defense in depth. Superseded destination requests transition idempotently to terminal `failed`/`package_superseded` (including leased/processing requests) and are never claimed, retried, or recorded as succeeded; historical packages, decisions, requests, and audit events are preserved.

Shopify and Listings Spreadsheet require pre-side-effect dispatch authorization: `gmcom_authorize_destination_dispatch` is called immediately before each external write, and the post-call success/failure RPCs remain independent verification. Etsy application-level pre-side-effect authorization is **deferred** to the explicitly authorized Etsy activation phase (authoritative checklist: `gm-commerce/docs/canonical-etsy-draft.md` §10); Etsy remains implemented but not launch-active and fail-closed, and the shared database protections still cover Etsy requests.

Database-to-external-API atomicity is impossible; authorization is therefore placed immediately before each write, and post-call validation remains defense in depth. The remaining database-to-external-API micro-window is bounded to a single await boundary and is documented.

**Reason:** Phase 0 Slice 2 (PR #70, merge `9cc5f6a`, migration `20260816000000_phase0_slice2_current_version_safety.sql`) resolves finding D1/High deterministically at the database boundary rather than relying on application discipline. Verified: 2204 application tests, migration from empty + upgrade, consolidated schema, materialization race, PR CI, and post-merge CI run `31605446557` (24/24 jobs).

## 2026-08-12 — Destination-request correlation/idempotency and supersede semantics

**Decision:** Every destination request carries a correlation identity and the destination outbox enforces idempotent request lifecycle semantics (one durable enqueue per destination intent; lease-protected claims; immutable attempt/ledger records). A later intentional request for a newer package version may create a new request for that version.

**Reason:** Destination writes (Shopify/Etsy/Listings Spreadsheet) are durable outbox operations that must not duplicate on retry or double-click. **Finding D2/High (destination-request deduplication) is resolved** by Phase 0 Slice 3 (PR #71, merge `851be71`); see the "Destination-request deduplication is delivery-intent-based" entry below.

## 2026-08-12 — Destination-request deduplication is delivery-intent-based (Phase 0 Slice 3)

**Decision:** Delivery intent is identified independently from the correlation ID: `(environment, commerce_package_id, destination, intended_external_destination, custom_destination_label)`. At most one ACTIVE (pending/processing/needs_confirmation) operational destination request exists per delivery intent, enforced by a partial unique index. Different-correlation duplicate submissions (sequential or concurrent) converge to the existing request; active requests outrank terminal history during convergence; terminal-only history follows the established no-redelivery rule; exact-correlation replay stays idempotent and mismatched correlation reuse fails closed. Correlation IDs remain the replay identity but are not the business deduplication key. Upgrade reconciliation retains the oldest ACTIVE request and records discarded duplicates as terminal `failed`/`duplicate_intent` (distinct from `package_superseded`, which remains reserved for actual package supersession), naming the surviving request in audit metadata. Existing same-row retry and Etsy recreate behavior are preserved; historical requests and events are preserved; superseded packages remain ineligible via Slice 2. **Deliberate redelivery of the same package version to the same destination is not implemented and requires a future explicit owner-authorized mechanism** — it is not invented here.

**Reason:** Phase 0 Slice 3 (PR #71, merge `851be71`, migration `20260817000000_phase0_slice3_destination_dedup.sql`) resolves finding D2/High at the database boundary. Verified: 2211 application tests, migration from empty + upgrade, consolidated schema, dedup race (concurrent and terminal-history), upgrade reconciliation, PR CI, and post-merge CI run `31615866708` (24/24 jobs).

## 2026-08-12 — Destination outbox/lease processing design

**Decision:** All three destination adapters write through the canonical destination routing outbox (PR #65): a durable request with a lifecycle, lease-protected claim (one worker wins the lease), immutable attempts/ledger, error allowlist, and atomic completion. The database write and the remote destination write are never one atomic transaction.

**Reason:** This gives idempotency, retry, and failure visibility for remote destination calls, consistent with the earlier Listings Spreadsheet decision (never claim DB + remote as one transaction). **Open finding C3/High remains open:** there is **no automatic background consumer** — processors are manual server actions today. Automatic workers/retries/queue maintenance are required for launch per owner decision 3 and are not yet implemented.

## 2026-08-12 — Etsy fail-closed configuration

**Decision:** Etsy launches only after its token store and policy source are configured and verified. Until then, the Etsy path remains fail-closed (a create-intent record is made before any remote call; unresolved/needs-confirmation intents park rather than publish). Shopify launches first.

**Reason:** Owner decision 6. The Etsy draft adapter is implemented (PR #68) but is not launch-ready until its external configuration is verified; Etsy stays fail-closed until that verification completes.

## 2026-08-12 — One final canonical GM Commerce system

**Decision:** GM Commerce converges on **one canonical system** as the end state. Legacy functionality is retired only after a dependency-ordered cutover; there is **no premature deletion** of legacy functionality.

**Reason:** Owner decision 1. The legacy pipeline remains the working source pipeline during Phase I and Phase 0; cutover is sequenced (below) so nothing is removed before its canonical replacement is mapped, migrated, and verified.

## 2026-08-12 — Safe legacy cutover sequence

**Decision:** Legacy retirement follows this dependency-ordered sequence: (1) inventory dependencies/capabilities are moved, (2) canonical replacements are mapped, (3) migrate and verify, (4) stop new legacy writes and legacy routing, (5) confirm no canonical dependency remains, then (6) retire legacy runtime paths.

**Reason:** Owner decision 1. Legacy cutover comes **later** — after the required capabilities and dependencies are safely moved. Phase 0 Slices 2 and 3 (current-version invariant/regeneration safety and destination-request deduplication) are both completed and merged (PRs #70–#71); legacy cutover is sequenced after the required capabilities exist.

## 2026-08-12 — Photo architecture: Google Drive human master → Supabase Storage app copies → OneDrive archive

**Decision:** Google Drive is the **human master** for photos; Supabase Storage holds the **application copies/derivatives**; OneDrive is an **independent backup/archive** only.

**Reason:** Owner decision 5. This is the required end-state storage architecture; it is recorded as a decision and remains to be implemented (automatic background/derivative storage work is part of the launch requirements).

## 2026-08-12 — Owner work is limited to initiation, image arrangement, owner-only facts, and final approval

**Decision:** Normal owner work in the workflow is limited to: selecting products, adding/arranging images, supplying genuinely owner-only facts, and approving/rejecting listings. Processing, retries, queue maintenance, and destination creation must be **automatic**; manual Process/Retry actions are an exceptional fallback only.

**Reason:** Owner decision 3. This defines the automatic-processing launch requirement (review finding C3/High) and the fallback-only role of manual processing.

## 2026-08-12 — Recovery order: deterministic automation → constrained AI fresh-eyes → owner escalation

**Decision:** The recovery ladder is: (1) deterministic automation, (2) a constrained AI "fresh-eyes" agent, then (3) owner escalation. Manual recovery is not the first resort.

**Reason:** Owner decision 4. This is the required operational recovery design; the deterministic step (and constrained-AI step) are part of the automatic-processing launch requirements, not yet implemented.

## 2026-08-12 — Shopify launches first; Etsy stays fail-closed until verified

**Decision:** Shopify is the first launch destination. Etsy remains fail-closed until its token store and policy source are configured and verified.

**Reason:** Owner decision 6. Confirmed with the Etsy fail-closed configuration decision above; the Shopify draft-only end-to-end launch verification has not yet been executed.

## 2026-08-12 — Batch processing (Phase 2) is required before launch and requires Phil's design approval

**Decision:** Batch processing is **REQUIRED before launch**, and its behavior is a **mandatory owner-design checkpoint**: it must not be designed or implemented until Phil approves how it should work.

**Reason:** Owner decision 2. This gating is recorded so no contributor begins Phase 2 batch design/implementation before Phil's decision.

## 2026-08-12 — Phase 0 Slice 2 / Slice 3 sequence (current-version invariant, then destination dedup)

**Decision:** The approved sequence was **Phase 0 Slice 2: current-version invariant and regeneration safety** (finding D1/High), then **Phase 0 Slice 3: destination-request deduplication** (finding D2/High). **Both are now completed and merged** (PRs #70–#71). Legacy cutover comes later, after the required capabilities and dependencies are safely moved (see the safe cutover sequence above).

**Reason:** Owner-confirmed. This replaces any earlier statement that Phase 0 Slice 2 is "legacy cutover" — it is not; legacy cutover is a later, separately sequenced step.
