# GM Commerce — Product & Architecture Reset

**Status: Revision 3, `APPROVED FOR NARROW PHASE A IMPLEMENTATION`.** Codex confirmed the two targeted corrections and issued final judgment: narrow Phase A only (schema reconciliation, migration ledger, schema-from-empty CI, drift detection, the direct-SQL incident record — §1.1, §23). Phil separately corrected the record on `HY-LOB01-C04` (never real; test data; deletion authorized and executed — §1.2, §22.1) **and on the migration ledger itself**: a prior pass here wrongly assumed a Supabase CLI migration ledger already existed and just needed two rows backfilled. It attempted that backfill and got `ERROR: 42P01: relation "supabase_migrations.schema_migrations" does not exist` — **no ledger has ever existed for this project.** Re-verified against the live database: 6 of the 7 committed migrations are genuinely applied; the 7th (`20260803000000_commerce_field_ownership.sql`) is not (`commerce_details.price`/`listing_packages.content_provenance` both return Postgres `42703`, column does not exist). `gm-commerce` (`f316ec2`) now has `supabase/ledger-bootstrap.sql` — a single, idempotent, reviewed SQL file that creates the ledger from scratch and records exactly the 6 verified migrations (never the 7th) — plus a reproducible generator script and structural tests. **One action remains, for Phil only:** open `supabase/ledger-bootstrap.sql` on GitHub, copy it into the already-open Supabase SQL Editor, click Run, and report the result. See `gm-commerce`'s `docs/incidents/2026-08-03-direct-sql-commerce-details.md` for the full corrected account. **Phase B0, Phase B, Etsy, new AI features, and any implementation beyond this narrow Phase A scope remain unauthorized**, and the broader uncommitted GMCOM-015/016 working tree on `gm-commerce` was deliberately left uncommitted, out of this narrow scope.

## Revision 3 changelog (2026-08-03)

Copilot's independent review of Revision 2 (source: `gm-commerce` audit commit `bd6503841add2b00c7bb48c4a94e4076abd373d9`, `docs/autonomous-commerce-intelligence-audit.md`; headquarters decision commit `da44c96`) returned **REQUIRES REVISION 3 AND RE-REVIEW**. Two categories of correction, both addressed below:

1. **A documentation-state discrepancy** — Revision 2 referred to `listing_packages.content_provenance`, `commerce_details`, and a Commerce Readiness Gate as "substantially coded" Phase A work. Copilot could not locate these on `gm-commerce`'s reviewed `origin/main` because they aren't there — they exist only in this session's local, uncommitted working tree. §1 below resolves this precisely, component by component, with no rounding in the system's favor.
2. **Fifteen finite architectural corrections** and **seven canonical architecture decisions**, each incorporated as an actual specification (interfaces, schemas, state machines) in its own section below, not a restated intention. A traceability table (§22) maps every correction to where it's addressed, what the concrete artifact is, how it's tested, which phase builds it, and what risk remains open.

**Post-push correction (same day, before Copilot's re-review began):** the first push of this revision (`b2feb517`) itself mischaracterized `HY-LOB01-C04`'s test-session `commerce_details` values as "real data ... entered from facts Phil gave directly in chat." Phil caught and corrected this directly: the plant, its Skrybix record, and its photographs are real; the generated listing, price, commerce details, review approval, and Shopify draft were test artifacts, not a genuine owner-approved offering. §1's second paragraph is corrected in place, §1.1 adds database-change-control policy, §1.2 adds a general test-data quarantine/remediation process (applied first to `HY-LOB01-C04`, with no deletion or alteration of its records without Phil's explicit approval), and §25 adds an enforced `RecordContext` model (environment, record purpose, owner-approval genuineness, and eligibility for promotion/pricing/publication) so this class of mistake — a real inventory item's test data being treated as production truth — is structurally prevented going forward, not just corrected once in prose. §22.1 tracks these additions in the traceability table separately from Copilot's original fifteen.

### Codex's independent re-review (2026-08-03): `APPROVE AFTER DOCUMENTED CORRECTIONS`

Codex reviewed the corrected commit and returned **`APPROVE AFTER DOCUMENTED CORRECTIONS`** — twelve finite corrections, none requiring a broad Revision 4 redesign. Each is resolved in place below, with its own section reference:

1. **Marketplace publication and public editorial content were one overloaded entity.** The old `Publication` entity conflated an immutable marketplace publish-attempt/receipt record with a public educational asset. Split into `MarketplacePublicationAttempt` (§5, §13, §22.2) and `EditorialAsset` (§5, §18) — `Publication` no longer exists as a type name in this document.
2. **`CommercePackage` was SKU-level only.** Expanded with variant-level offer records, inventory-item references, per-variant price/compare-at price, quantity/inventory policy, SKU/barcode, weight/dimensions, shipping profile, tax/compliance attributes, exact-item vs. representative-item offer mode, variant media associations, option names/values, availability state, and market/currency context (§7).
3. **A package-wide `evidenceRefs` array was insufficient lineage.** Every `CommercePackage` field/content block now carries its own field-level lineage — selected claim IDs, recommendation ID, `SelectionTrace` ID, confidence, applied policy versions, and owner-decision/override reference where applicable (§7).
4. **The Repository contract (§9) was missing operations required elsewhere in this document.** Evidence (`ingestEvidence`, `getEvidenceRevision`, `searchEvidence`), claims (`verifyClaim`, `supersedeClaim`, `recordContradiction`, `resolveContradiction`), policies (`createPolicyVersion`, `queryApplicablePolicies`, `evaluatePolicy`), recommendations/packages (`createRecommendationRun`, `recordValidationResult`, `getCompletedPackage`), decisions/learning (`proposeCorrectionScope`, `activateRule`, `revokeRule`), and lineage (`getAuditTrail`, `getMarketplacePublicationLineage`, `getEditorialPublicationLineage`) are all now part of `IntelligenceRepositoryV1` (§9).
5. **`SourceConnector` (§8) was missing operations real connectors need.** `health`, cursor-based pagination with a returned next cursor, connector/source schema versions, an observed-at timestamp, source provenance, access classification, an identifier-resolution method returning candidate matches with confidence, a structured change-feed result, idempotency keys for writeback, an approved command type in place of a generic payload, a structured writeback receipt, and explicit retry/error semantics are all added (§8). Writeback remains disabled by default and independently authorized.
6. **`RecordContext.ownerApproval.genuine: boolean` (§25) collapsed two different meanings into one `false` value** — a record nobody has reviewed yet, and a documented test exception, were indistinguishable. Replaced with an explicit `approvalState: "not_required" | "pending" | "genuine" | "test_exception"`, with `testExceptionRef` required only for `test_exception` and reviewer identity/decision timestamp/`OwnerDecision` reference required for `genuine` (§25).
7. **Eligibility flags (§25) had no stated default.** `knowledgePromotion`, `pricingAndPerformanceAnalysis`, and `publication` now default to `false` at both the database and API layer; a missing or null eligibility value fails closed, never open (§25).
8. **`queryApplicableClaims`'s `environmentFilter` (§9) allowed ordinary cross-environment widening as a query parameter.** Removed. Production queries can never consume test/staging/development claims through the normal query path; cross-environment access now requires a separate, privileged operation carrying actor, reason, source/destination environment, correlation ID, and its own immutable audit event, scoped to audited promotion, authorized diagnostics, or fixture evaluation — diagnostic reads can never feed production recommendations or knowledge promotion (§9, §25).
9. **Claim verification and promotion had no stated threshold beyond "no unresolved contradiction."** §10.1 now defines minimum corroboration by claim class, independent-source-family requirements, authority thresholds, freshness requirements, when a single authoritative primary source suffices, and which state transitions may be automated, which require an authorized verifier, and which always require Phil.
10. **§19's Phase B0 correction capture referenced canonical `Correction` records that don't exist until Phase B.** A temporary `LegacyCorrectionEvent` (§14.1) closes the gap for Phase B0, with a defined migration into canonical `Correction` records once Phase B's entities and claims exist; test-derived corrections remain ineligible for migration.
11. **Research-ingestion isolation relied on delimiters alone.** Expanded into a full untrusted-content handling pipeline: raw retrieved content stored separately and treated as untrusted, deterministic normalization/sanitization before extraction, no retrieved text ever placed in the system/developer instruction channel, extraction into bounded structured claim candidates, validators receiving only the candidate claim and minimal cited evidence spans, retrieved instructions ignored and logged as suspicious content, automatic promotion limited by source class and verification policy, and required adversarial prompt-injection fixtures across web pages, PDFs, OCR, supplier files, and OneDrive documents (§10).
12. **Owner-minutes-per-SKU instrumentation (§20) didn't separate review/approval/correction/escalation/navigation time from unavoidable physical work**, and inferred duration only from widely spaced timestamps. Corrected with explicit time categories and defined start/stop event semantics (§20).

**Additional documentation corrections from the same review:** the six source categories and eight evidence types are now stated in full in this document (§10) rather than by reference to Revision 2's history; §1.1 clarifies that migration reconciliation must *baseline* the already-applied live schema rather than re-execute its non-idempotent creation SQL, and requires a migration-ledger backfill/reconciliation record specifically for the direct-SQL incident; §1 restates that all four currently-failing local tests (`app/review/shopify-publish.test.ts`) remain documented as unresolved and that nothing merges until the retained code they cover has a fully passing suite; and `HY-LOB01-C04`'s records remain untouched — no deletion, quarantine action, or alteration — without Phil's explicit approval, unchanged from §1.2. §22.2 tracks all twelve corrections in the traceability table, alongside §22 (Copilot's original fifteen) and §22.1 (the same-day environment/test-data corrections).

**No broad Revision 4 redesign was required or performed.** This remains Revision 3, corrected a second time in place. Per Codex's instruction, only this reset/status documentation moves in this pass — production implementation and Etsy work remain paused pending Codex's targeted verification of these twelve corrections.

### Codex's targeted verification (2026-08-03): two corrections applied

Codex verified the twelve corrections above and returned **`APPROVE AFTER TWO TARGETED CORRECTIONS`** — all twelve are present and substantially satisfy the review; no Revision 4 or broad redesign is required. Two internal inconsistencies, both now fixed:

1. **§13 artifact 7 still pointed to the retired package-wide `evidenceRefs`** even though §7 had already replaced it with field-level `fieldLineage[]` (Codex Correction #3). Corrected to require the persisted `CommercePackage`, its complete `fieldLineage[]`, field-level confidence, and resolvable `SelectionTrace` records — the document-wide search Codex requested found no other current-usage reference; the remaining mentions of `evidenceRefs` (§1's changelog, §7's own "replaces the old..." explanation, and a code comment) are historical and correctly describe what was replaced.
2. **§10.1's "always requires Phil" rule over-escalated.** The prior wording required Phil's confirmation for every price/weight/shipping/compliance/safety/marketplace-policy claim regardless of evidence — directly contradicting this document's own governing completion test by recreating exactly the bottleneck it exists to eliminate. Corrected to the operating distinction Codex specified: authoritative-system-of-record facts (Skrybix physical state, a Gathering Moss controlled measurement, a marketplace's own current policy) and applications of an already-versioned Gathering Moss policy may verify automatically when fresh, uncontradicted, correctly classified, and fully traced; an AI-generated recommendation (most consequentially, price) is never auto-verified as a fact but is evidence-backed, independently validated, and presented with reasoning/confidence for Phil's approval as part of one completed-package review, not a separate per-field fact-check; Phil remains required for genuinely new/changed policy, material unresolved conflict, exceptional safety/legal judgment, identity merge/split, and durable-rule activation broader than SKU/InventoryItem scope. §22.2 item 9's acceptance test is updated to assert both halves of this distinction.

Per Codex's instruction, only `PRODUCT_RESET_2026-08-03.md` and `STATUS.md` moved in that pass; `HY-LOB01-C04`'s records remained untouched without Phil's explicit approval at that time.

### Codex's final judgment and Phil's `HY-LOB01-C04` correction (2026-08-03): `APPROVED FOR NARROW PHASE A IMPLEMENTATION`

Codex confirmed both targeted corrections above and issued its final judgment: **`APPROVED FOR NARROW PHASE A IMPLEMENTATION`** — strictly §1.1's schema reconciliation, migration ledger, schema-from-empty CI, drift detection, and the direct-SQL incident record (§23). No broader implementation, Phase B0, Phase B, or Etsy work is authorized by this judgment.

Separately, **Phil corrected a standing factual error this document repeated across two prior revisions**: `HY-LOB01-C04` was never a real plant cutting or a real Gathering Moss inventory item — it was synthetic test data created and used by Claude for pipeline testing, not (as previously written) a genuine Skrybix cutting with a test-generated commerce layer on top. Phil explicitly authorized eliminating anything specifically associated with the identifier. §1 and §25 are corrected in place to stop describing it as real at any layer, and §1.2's remediation process — previously blocked on approval — ran to completion: the `products` row and its cascaded child rows (`listing_packages`, `listing_package_versions`, `photo_sets`, `photo_assets`, `photo_derivatives`, `shopify_publications`, `commerce_details`) were deleted and verified empty; the live Shopify draft `gid://shopify/Product/10220386648384` was deleted via `productDelete`; the local photo folders were removed. See §1.2 and §22.1's updated remediation row for the full disposition, including the one item out of this project's control (a possible corresponding Skrybix-side record).

This revision does not soften Revision 2's substance — the vision, the completion test, the six-source and eight-evidence-type taxonomies, the manual-handoff inventory, and the Skrybix gap analysis all carry forward. What changes is that everything downstream of "Product Truth" is now specified as an actual system: canonical entities, a marketplace-neutral Commerce Package, a formal source-connector and Repository service contract, a compliance gate distinct from recommendations, catalog-scale non-functional requirements, and a corrected phase sequence that puts the completed-package review shell and correction-capture in Phase B0 — before the claim model itself — instead of leaving the legacy edit form as the default experience for months while the backend gets smarter underneath it.

---

## Why this document exists

Inspecting HY-LOB01-C04's real Shopify draft (GMCOM-012) surfaced defects — placeholder text, $0 price, no weight, duplicate alt text, no care instructions. The Commerce Readiness Gate built in response correctly refuses to publish incomplete content, but every blocker it raised turned into a question back to Phil rather than a system-derived answer. Phil's correction, reconfirmed twice since: GM Commerce should function as an autonomous commerce director. Phil directs it and approves its output; he should never be the fallback worker inside his own software, and the software's own documentation of itself should never claim more than what's actually committed and addressable.

**Governing completion test**, referenced throughout:

> Does this let Phil accomplish work that would otherwise require additional employees, without making Phil the missing employee inside the software? If routine work still falls back to Phil or Crystal when it could reasonably be retrieved, researched, inferred, derived, remembered, recommended, generated, or verified by the system, the workflow is not complete — regardless of whether the technical pipeline functions.

## 1. Documentation-state reconciliation

Copilot is correct: `origin/main` on `gm-commerce` is at `f2a073d` ("Verify GMCOM-012 against a real Shopify store"), the same commit Revision 1 described as the last real milestone. Every "Phase A" component Revision 2 described as built or substantially coded exists **only** in this session's local, uncommitted working tree on the `main` branch checkout at `C:\Users\pwach\OneDrive\Documents\GitHub\gm-commerce`. There is no commit SHA to give for any of it, because none of it has been committed.

One further discrepancy, more consequential than the code: the **live Supabase database** (`wcrcllhvgbhykbonopzx`) already has the `commerce_details` table and the `listing_packages.content_provenance` column applied — created by running the migration SQL directly against the live database earlier in this session. **The migration file that created these does not exist in git at all.** The live schema is currently unreproducible from version control. This is exactly the "architecture cannot depend on an unaddressable worktree" risk Copilot named, now confirmed to extend past the worktree into the live database itself. §1.1 below adds the database-change-control policy this requires.

**Second correction to this section, per Phil's explicit instruction (2026-08-03):** the prior wording above described `HY-LOB01-C04` as "a real Skrybix cutting — the plant, the Skrybix record, and its photographs are genuine." That is also wrong, and is corrected here rather than left standing. **`HY-LOB01-C04` was never a real plant cutting or a real Gathering Moss inventory item.** It was synthetic test data created and used by Claude for pipeline testing — the identifier, the "plant," its listing, its `commerce_details` row (price, weight, shipping/condition/exact-item text, collection assignment), the review approval, its photographs, and the resulting Shopify draft were all **artifacts of this session's end-to-end pipeline test**, not a genuine Gathering Moss product at any layer. Phil has explicitly authorized elimination of anything specifically associated with this identifier, and §1.2's remediation ran on that authorization — see its updated disposition below. This document no longer describes `HY-LOB01-C04` as a real inventory item, a real cutting, or an owner-approved product at any point. Specifically:

- `HY-LOB01-C04`'s `commerce_details` values are **not** genuine production commerce data.
- Its **$24.95 price must not be used as pricing history or a comparable sale** anywhere in this document, in any future price recommendation, or in any pricing-history record §7/§14 describe.
- Its generated listing copy and care guidance are **not** approved Gathering Moss knowledge and must not be promoted into the knowledge domains §17/§18 describe.
- Its collection, shipping, weight, inventory-policy, and condition values are **not** durable owner decisions.
- It must **not** be included in any commerce-performance metric (§20).
- It must **not** seed correction learning (§14), marketplace strategy, public plant knowledge (§17), or future recommendations (§7).

| Component | Repository | Branch | Commit | State | Tests currently covering it | Revision 3 disposition |
|---|---|---|---|---|---|---|
| `commerce_details` table + `listing_packages.content_provenance`/`seo_title`/`seo_description` columns | `gm-commerce` (schema) + live Supabase project `wcrcllhvgbhykbonopzx` | `main` (local working tree only) | **none** — two migration files exist on disk, untracked, never committed | **Applied directly to the live database**, not reproducible from git | None (schema has no CI) | **Reconcile immediately, before any other Phase A work resumes**: commit the actual applied migration SQL so the live database becomes reproducible, or the live table must be treated as an undocumented, at-risk production change. This is not a Revision 3 design question — it is an operational gap that predates and is independent of this reset. |
| Commerce Readiness Gate (`lib/commerce-readiness/gate.ts`, `denylist.ts`, `taxonomy.ts`, `defaults.ts`, `types.ts`, `load.ts`) | `gm-commerce` | `main` (local working tree) | none | Local-only, untracked files | 12 local `gate.test.ts` assertions, last run passing, never executed in CI | **Migrate.** The deterministic checks (denylist, near-duplicate, structural completeness) become inputs to Phase E's independent-validation layer; the field-by-field blocking logic is superseded by the canonical `CommercePackage` readiness state (§7) and the compliance gate (§12), not retained as its own standalone gate module. |
| `/commerce/[sku]` UI + `app/commerce/actions.ts` (`saveCommerceDetails`, `ensureCommerceDefaults`) | `gm-commerce` | `main` (local working tree) | none | Local-only, untracked | 7 local `actions.test.ts` assertions, passing | **Discard the UI, retain the decision logic.** `ensureCommerceDefaults`'s deterministic default rules (inventory policy, shipping text, exact-item disclosure, weight templates, category/collection mapping) are genuinely dependency-free and migrate into Phase B's claim model as `source: derived_rule` / `store_policy` claims. The page itself is superseded by Phase B0's completed-package review shell. |
| Price ownership moved from `listing_packages.price` (AI-touched, silently wiped every regen) to `commerce_details.price` | `gm-commerce` | `main` (local working tree) | none | Local-only | Covered in `app/actions.test.ts`, `app/review/actions.test.ts` (passing); **`app/review/shopify-publish.test.ts` has 4 failing tests** from an incomplete fixture update, left mid-fix when implementation was paused | **Retain the decision** (price is a stored commerce fact, never AI-generated content) — it was correct and stays correct under Revision 3. The implementation migrates into the `CommercePackage` model (§7) rather than living in `commerce_details` as a bespoke column once Phase B lands. |
| AI-researched care instructions (`lib/ai/prompt-builder.ts` changes allowing the model to write general species care) | `gm-commerce` | `main` (local working tree) | none | Local-only | Covered by `listing-generator.test.ts` (passing) | **Discard as implemented.** This is Revision 2's own named mistake (§12 of Revision 2, carried forward): pretrained-knowledge recall with calibrated language, no live retrieval, no citation, is not the research capability Phil asked for. Redo under Phase C/D once OneDrive-backed evidence and research-ingestion security (§10) exist. |
| Sales-summary / alt-text automatic duplicate repair (one conditional 3rd AI call, fail-closed on persistent duplication) | `gm-commerce` | `main` (local working tree) | none | Local-only | Covered by `listing-generator.test.ts` + `alt-text.test.ts` (passing) | **Retain the mechanism** (deterministic detection → one targeted repair call → fail closed if still duplicated), reframed as an output of Phase E's independent-validation layer rather than folded into the generation call itself. |
| "Regenerate listing" button on `/review` (fixes a real, independent bug: unlocking a reviewed package left no UI path to actually re-trigger generation) | `gm-commerce` | `main` (local working tree) | none | Local-only | Manually verified against the live database and a real OpenAI call in a real browser session (not a vitest test) | **Retain unconditionally.** This is a genuine, narrow bug fix orthogonal to the reset — safe to commit and keep regardless of how Revision 3's review lands. |
| "Regenerate all" alt-text button on `/photos/[sku]` | `gm-commerce` | `main` (local working tree) | none | Local-only | Manually verified against the live database and a real OpenAI call in a real browser session | **Retain unconditionally**, same reasoning. |

**Current full local test-suite state, run just now, not carried forward from an earlier claim:** 101 passing, 4 failing, 105 total, all four failures isolated to `app/review/shopify-publish.test.ts` from the incomplete fixture update noted above. None of this has ever run in CI, because none of it is committed.

**What Revision 3 does about this:** nothing in this document authorizes committing or pushing any of the above gm-commerce code or the undocumented database migration — Phil's instruction remains that only the reset and status documents move this turn. The disposition column above is Revision 3's *design* answer (retain/migrate/discard), not an action taken. Before Phase A of the revised sequence (§19) resumes, the live-database migration-file gap specifically should be resolved on its own, independent of whether Revision 3 is approved, since an unreproducible production schema is a risk regardless of which architecture eventually builds on top of it.

**The four currently-failing tests remain documented as unresolved, not silently carried forward.** `app/review/shopify-publish.test.ts`'s 4 failures (of 105 total local tests) stay exactly as reported above until whichever future session resumes this code fixes the incomplete fixture update that caused them. Nothing built on top of the retained `gm-commerce` code in §1's disposition table may merge until the intended retained portions of that code have a fully passing local suite — a partially-fixed or newly-green-by-deletion suite does not satisfy this.

### 1.1 Database change control

Direct-to-production SQL (exactly what created the undocumented `commerce_details`/`content_provenance` state above) is the specific failure this policy closes, stated as a standing rule rather than a one-time cleanup:

- Every schema change requires a committed migration. No exceptions for "it's just a quick column add."
- Migrations are ordered, immutable once applied, and tracked in a migration ledger (a table in the database itself recording which migrations have run, in what order, when — the standard pattern `supabase/migrations/` already implies but has never been enforced against direct SQL execution).
- **Direct production SQL, when it genuinely must happen (an emergency fix), requires a matching version-controlled migration written and committed immediately after** — not "eventually," not "next session" — plus an incident record stating what was run directly, why the normal path wasn't used, and confirmation the follow-up migration reproduces it exactly.
- CI must be able to build the complete schema from an empty database using only committed migrations — this is the actual test that would have caught today's gap immediately, and its absence is why it wasn't caught until Copilot's review.
- CI must compare the expected migration set against the live database's migration ledger and fail on drift — so "the live database has something git doesn't know about" becomes a blocked deploy, not a discovery six phases later.
- No phase in §23's sequence may depend on an uncommitted or unreproducible schema. Concretely: Phase A cannot be considered started until the current live-database gap is closed under this policy.
- **Reconciling today's specific gap is a baseline operation, not a re-creation.** The live database already has `commerce_details`, `listing_packages.seo_title`/`seo_description` applied (verified 2026-08-03 via direct REST probes) — **but not `listing_packages.content_provenance` or `commerce_details.price`**, which a later verification pass found were never actually applied despite their migration file being committed; see `gm-commerce`'s incident doc. Where a migration's objects genuinely are already live, the reconciliation migration must be written to **record** that this schema state already exists — a migration-ledger entry stamped against the actual applied state — not to re-run the `CREATE TABLE`/`ALTER TABLE` SQL a second time, and never to record a migration as applied that a direct check shows is not.
- **A dedicated incident record accompanies the reconciliation**, per this policy's emergency-SQL requirement applied retroactively to the SQL already run against production: what was executed directly, when, why the normal path wasn't used, and — critically — independent verification of what's actually live rather than an assumption carried over from a prior session's documentation. `gm-commerce`'s `docs/incidents/2026-08-03-direct-sql-commerce-details.md` and `supabase/ledger-bootstrap.sql` are this record and this reconciliation for the current, corrected understanding: **no migration ledger ever existed for this project** — this was not a two-row backfill, and treating it as one was itself an error this document corrects.

### 1.2 `HY-LOB01-C04` test-data correction and remediation

The general process, stated once and applicable to any future test-derived data found in production, not only this instance:

1. **Identify** existing test-derived records in the live database.
2. **Classify and quarantine** them from production retrieval — marked so no query path (§9, §25) treats them as eligible production knowledge, pricing history, or performance data while classification is pending.
3. **Determine disposition** for each: retained as a labeled evaluation fixture (§25's `recordPurpose: fixture`), moved to a non-production environment, or removed.
4. **Preserve an audit record** of the test itself — what was tested, when, by whom/what process, and why the data exists — regardless of what happens to the data.
5. **Do not delete or alter the existing records without Phil's explicit approval.**

**Disposition executed for `HY-LOB01-C04` (2026-08-03), following Phil's explicit factual correction and deletion authorization:** Phil clarified that `HY-LOB01-C04` was never a real plant cutting or a real Gathering Moss inventory item at all — it was synthetic test data created and used by Claude for pipeline testing — and explicitly authorized elimination of anything specifically associated with the identifier, satisfying step 5's approval requirement for this instance. Steps 1–4 ran against the live database and connected Shopify store, exact-identifier only, no pattern-based deletion:

- **Inventoried** (step 1): one row each in `products`, `listing_packages`, `photo_sets`, `shopify_publications`, `commerce_details`; two rows in `listing_package_versions` (versions 1–2); two rows in `photo_assets`; four rows in `photo_derivatives`; the live Shopify draft `gid://shopify/Product/10220386648384`; two original photo files and four derivative files under `All Product Photos/HY-LOB01-C04` and `All Product Photos/_derivatives/HY-LOB01-C04`.
- **Disposition** (step 3): removal, not quarantine-and-retain — Phil's authorization was for elimination, not fixture retention.
- **Executed**: the `products` row was deleted (cascading, by foreign-key design, to every listed child table in one operation — verified empty afterward in each table individually); the Shopify draft was deleted via `productDelete` (no user errors, confirmed by ID); the two local photo folders were removed. No shared migration, reusable code, general test fixture, or unrelated record was touched — the several unit-test files that use the string `"HY-LOB01-C04"` as an example SKU are ordinary test fixtures, not data records, and are correctly out of this process's scope.
- **Audit record** (step 4, this paragraph and the reset repository's commit history): what existed, exactly what was removed, and why, is preserved here even though the underlying rows are gone.
- **Out of this project's control:** `HY-LOB01-C04` also appears as a `source_record_id` under Skrybix's own separate system (a distinct repository/service this project has no database credentials for). If Phil wants any corresponding Skrybix-side record removed, that is a Skrybix-repository action, not a GM Commerce one — flagged here, not executed.

No append-only record required voiding-in-place instead of deletion — the cascade delete resolved every table cleanly.

## 2. Product vision

Unchanged from Revision 2. GM Commerce is meant to operate as a commerce director — a function that would otherwise be a merchandiser, copywriter, product researcher, pricing analyst, and marketplace operations person, all reporting to Phil — not a form, not a Skrybix-to-Shopify pipe, and not a CRUD app with AI-generated copy bolted on. It assembles everything knowable before treating something as unknown, researches with real evidence rather than leaving a blank field, sees actual photo content, verifies independently, recommends as standard output, remembers corrections durably and at the right scope, and defers to Phil only for what only Phil can decide.

The connected, later-phase objective also carries forward: Gathering Moss becoming a trusted public source of plant knowledge, built from the same verified repository, not a separate content operation (§16, §17).

## 3. Current architecture gap analysis (condensed)

Carried forward from Revision 2 without change in substance. What's built and sound on `origin/main` today: guided intake (GMCOM-006), the AI Provider abstraction (GMCOM-007), the multi-stage Listing Quality Engine (GMCOM-008), atomic/versioned/validated generation (GMCOM-009), the photo pipeline (GMCOM-011), and the real Shopify draft publisher (GMCOM-012). Everything from GMCOM-015 onward — the Commerce Readiness Gate, `commerce_details`, price ownership, care research, duplicate repair — exists only locally, per §1.

The gap itself is unchanged: price, weight, inventory policy, category/collections, shipping text, exact-item disclosure, care instructions, duplicate copy, photo quality judgment, self-review, cross-listing consistency, durable knowledge, recommendations, and learning from correction were all either a blank field for Phil to fill or a defect for him to notice. Revision 3 doesn't relitigate this table; it specifies the system that closes it.

## 4. Manual-handoff inventory (condensed)

Unchanged classification from Revision 2: (A) genuinely owner-only — final approval, marketplace choice, policy content itself, price as a final decision, what to carry/discontinue, a physical per-item fact nowhere else yet, establishing a new durable rule, approving public content; (B) currently manual, should be system-derived — everything else. Revision 3 adds precision to what "system-derived" actually means architecturally in the sections below, and Correction #11 (§14) adds precision to exactly when a (B)-category correction is allowed to generalize versus stay local.

## 5. Canonical entity model

*Correction #1, Decision #1.* This is the foundational gap Revision 2's own §11 self-critique identified and Copilot's review confirmed as the largest concrete hole: a claim model that's generic in the abstract still needs a real entity hierarchy underneath it, or "subject" stays an informally-typed string forever.

**Eighteen canonical entity types** (Codex Correction #1 splits the former `Publication` into two — see below), each with a stable, opaque ID (ULID-shaped, independent of any source system's natural key) issued once and never reused, and each carrying a `RecordContext` (§25) that determines whether it's real production data, a test artifact, or a fixture — an entity being real never automatically makes the claims about it real, which matters immediately: §1's `HY-LOB01-C04` correction is the concrete case this distinction exists for.

```
ProductConcept   — an abstract thing Gathering Moss sells (e.g. "Hoya lobbii
                    rooted cutting" as a concept). 1—N SKU.
SKU              — the sellable-unit identity (matches products.sku today).
                    N—1 ProductConcept (required). 1—N Variant (0 or more).
                    1—N InventoryItem. Identity/string is source-created
                    (Skrybix or SKU Generator) per the existing "GM Commerce
                    never creates SKUs" decision — unchanged — but the
                    canonical SKU *entity record* (with all its claim,
                    recommendation, and CommercePackage relationships) is
                    GM-Commerce-owned.
Variant          — a specific sellable configuration under a SKU (size,
                    color). N—1 SKU. Mostly relevant to non-plant products.
InventoryItem    — one specific physical unit: for a Skrybix cutting,
                    literally one physical plant; for an accessory, a
                    stock-keeping quantity record. N—1 SKU or Variant. This
                    is the default correction-scope boundary (§14).
SourceRecord     — an immutable snapshot-in-time record imported from an
                    external source system. Produced by a SourceConnector
                    (§8) as a SourceRecordSnapshot. N—1 SKU or ProductConcept
                    (a SKU may aggregate facts from more than one source
                    record over time; typically one primary).
PhotoAsset       — an original or derivative photo file (already has real
                    tables: photo_assets/photo_derivatives). N—1
                    InventoryItem, SKU, or ProductConcept.
Claim            — the atomic evidence-backed assertion (§6). Subject is a
                    typed reference to any entity above by (entityType,
                    entityId).
EvidenceSource   — a distinct originating source: a document, a database
                    record, a research query result. 1—N EvidenceRevision.
EvidenceRevision — one immutable version of an EvidenceSource's content
                    (sources get re-fetched/updated over time; each fetch is
                    a new revision, never a mutation). N—1 EvidenceSource.
                    1—N EvidenceAnchor.
EvidenceAnchor   — a specific locatable reference within an EvidenceRevision
                    (page, character range, bounding box, timestamp, image
                    region). N—1 EvidenceRevision. This is what makes
                    "evidence span" a typed, durable reference instead of a
                    prose description.
Policy           — a versioned Gathering Moss or marketplace rule. 1—N
                    Recommendation (a recommendation cites the policy
                    version active when it was made).
Recommendation   — a produced suggestion (price, taxonomy, SEO angle, ...)
                    tied to the claims/evidence/policy versions that
                    produced it. N—1 SKU or ProductConcept. 1—1
                    SelectionTrace (§18).
CommercePackage  — the marketplace-neutral, versioned assembled package
                    (§7). N—1 SKU. 1—N MarketplaceListing.
MarketplaceListing — one adapter's realization of a specific CommercePackage
                    version on one marketplace/shop (a Shopify draft, an
                    Etsy listing) — the current/latest realized state.
                    N—1 CommercePackage. 1—N MarketplacePublicationAttempt.
MarketplacePublicationAttempt — *(Codex Correction #1, new this revision.)*
                    an immutable record of one attempt to publish a specific
                    CommercePackage version to a specific MarketplaceListing:
                    MarketplaceListing ID, CommercePackage version, adapter
                    name + adapter version, the policy/compliance snapshot
                    (§12) checked at attempt time, the outbound payload and
                    its content hash, attempt timestamp, the remote
                    response/receipt, remote product/listing IDs returned,
                    verification result (§13 artifact 9's field-by-field
                    confirmation, formalized as this entity), and
                    failure/retry status. N—1 MarketplaceListing. Never
                    mutated after creation — a republish is a new attempt,
                    not an edit to a prior one. This is the entity that
                    carries §13 artifact 9 and closes the gap the old,
                    overloaded `Publication` entity left: a marketplace
                    publish-attempt/receipt record is structurally nothing
                    like a public educational asset, and conflating them
                    lost the immutable-receipt requirement Copilot's review
                    named.
EditorialAsset   — *(Codex Correction #1; renamed from `Publication`, same
                    slot in this list, no relation to commerce publishing.)*
                    a public educational asset (care guide, article,
                    species profile). Carries claim/evidence references
                    (§6), license and attribution state per
                    `EvidenceSource.sourceType` (§17), editorial approval
                    (RBAC-tracked, §15), and its own publication/
                    correction/retraction history as tracked events, never
                    a silent delete (§18). 0—N link to ProductConcept/SKU
                    (loose, editorial, not commerce) — distinct from
                    MarketplaceListing/MarketplacePublicationAttempt, which
                    are commerce-only.
OwnerDecision    — Phil's or Crystal's explicit decision record (approval,
                    rejection, policy-setting, merge/split confirmation).
                    1—N Correction may reference the same OwnerDecision.
Correction       — a specific edit/override event: scope, rationale,
                    before/after value, referencing the Claim,
                    Recommendation, or Policy it modifies (§14).
```

**Aliases.** `ProductConcept` and `SKU` carry an alias list (alternate names, common names, synonyms) distinct from canonical display name — directly closes the Skrybix "synonyms" gap (§15).

**Lifecycle ownership**, stated precisely to avoid re-litigating an already-settled boundary: Skrybix and the SKU Generator remain the *identity-creating* systems (unchanged — GM Commerce never mints a SKU or a Mother/Cutting ID). GM Commerce owns the *canonical entity record* once identity exists — the claims, recommendations, and packages attached to it. `InventoryItem` state (sold, reserved, available) is jointly owned: Skrybix is authoritative for a plant's physical state, GM Commerce is authoritative for its commerce/publication state, and §8's SourceConnector writeback contract is the only sanctioned path for GM Commerce to inform Skrybix of a sale — never a direct write to Skrybix's own tables.

**Cross-system merge/split identity resolution.** When two `SourceRecord`s are later discovered to represent the same real-world `ProductConcept`/`SKU` (a merge), or one entity is discovered to actually represent two distinct things (a split), this is **always an `OwnerDecision`** — never automatic, regardless of how high the system's own confidence is. A merge reassigns `Claim.subject` references from the losing entity ID to the winning one; the losing ID is retained as a superseded alias (never deleted), so every historical claim and citation that referenced it remains resolvable. A split spawns a new entity ID and re-partitions claims by evidence, with the same non-deletion guarantee. Cross-system identity *confidence* (how sure the system is that two records refer to the same thing) is itself a `Claim` — with its own evidence, confidence, and precedence — not a side-channel heuristic; a low-confidence match surfaces as a candidate for Phil's review, a high-confidence match still requires his confirmation before merging, and the system never silently merges on its own.

## 6. Claim & evidence model, and migration from `content_provenance`

*Correction #5.* The claim record, now fully typed against §5's entity model:

```
Claim {
  id: ClaimId
  subjectType: EntityType        // one of the 17 types in §5
  subjectId: EntityId
  predicate: string               // e.g. "care.light_requirement",
                                   // "commerce.price", "taxonomy.genus"
  value: Json
  sources: SourceRef[]            // ≥1; each a (sourceCategory, evidenceSourceId?)
  evidenceAnchors: EvidenceAnchorId[]   // may be empty only for
                                   // deterministic/derived claims (§6 of
                                   // Revision 2's source-category list,
                                   // categories 3/4/6) — never empty for
                                   // an AI-research claim (category 5)
  confidence: Confidence          // calibrated, not merely present/absent
  establishedAt: Timestamp
  lastVerifiedAt: Timestamp
  freshnessPolicy: FreshnessPolicyId
  applicabilityScope: Scope       // { entityType, entityId } | { genus } |
                                   // { category } | { marketplace } | global
  contradicts: ClaimId[]          // populated by reconciliation (§10), never
                                   // silently cleared
  precedenceRank: number          // derived, not stored authoritatively —
                                   // recomputed from sourceCategory +
                                   // ownerOverride at query time
  supersededBy: ClaimId | null
  ownerOverride: OwnerDecisionId | null
  correlationId: CorrelationId    // threads back to the pipeline run that
                                   // produced it (§13)
}
```

**Migration from the legacy `content_provenance` structure**, per Copilot's explicit requirement — exact source, not a generic "there's some legacy data" statement:

- **Source branch/commit:** none exists. Per §1, `content_provenance` (a `listing_packages` jsonb column mapping field name → source-tag string) was written this session, applied directly to the live database, and never committed. There is no commit to migrate *from* in the git-history sense — the migration is from **live, uncommitted schema state** to the new model, which is a materially different (and higher-risk) migration than a normal versioned-schema upgrade.
- **Dual-write period:** while `commerce_details`/`listing_packages` remain the system of record for any in-flight product during Phase B, every write to those tables also writes the equivalent `Claim` record(s), so no product mid-pipeline loses data during cutover.
- **Backfill validation:** for every existing row in `commerce_details`/`content_provenance` (currently: one row, `HY-LOB01-C04` — a test artifact per §1.2, not genuine production data), a backfill job creates the equivalent `Claim` records, each carrying `context.recordPurpose: "test"` (§25) so the backfill itself doesn't launder test data into production-eligible claims, and a validation pass confirms the reconstructed value matches the legacy column value exactly before the legacy column is considered migrated.
- **Rollback:** the legacy columns are not dropped until Phase B's claim-store has run in production (however "production" is defined at that point — even a single-owner real-use period) for a defined burn-in window with zero reconstruction mismatches; until then, the legacy columns remain the fallback read path.
- **Cutover:** reads switch to the claim store first (writes still dual-written), then writes switch, then legacy columns are marked deprecated, then — only after the burn-in window — dropped in a dedicated migration.
- **Retirement criteria:** legacy columns may be dropped only when (a) the burn-in window has passed with zero mismatches, (b) no code path reads them directly, and (c) a schema diff confirms no other consumer (an ad hoc script, a future integration) depends on them.

## 7. Marketplace-neutral Commerce Package

*Correction #2, Decision #2. Expanded by Codex Corrections #2 and #3.* Adapters (Shopify today; Etsy, Whatnot, and future marketplaces later) must consume only a validated `CommercePackage` version and must never reconstruct business truth independently — directly closing the gap named in Revision 2's own §11 (`buildShopifyProductInput` today builds straight from Listing-Package-shaped input to Shopify-specific types, with no marketplace-neutral intermediate).

**Codex Correction #2 — variant-level offers and exact-item disclosure.** The original schema was SKU-level only, which cannot represent a genuinely multi-variant product (Plant Pals and similar) without an adapter silently reconstructing offer-level business truth — exactly what adapters are forbidden from doing. `CommercePackage` now carries one or more `VariantOffer` records:

```
VariantOffer {
  id: VariantOfferId
  commercePackageId: CommercePackageId
  variantId: VariantId | null       // null for a single-variant SKU offered
                                     // directly at the SKU level
  inventoryItemRefs: InventoryItemId[]   // which physical unit(s) this
                                     // offer draws against
  offerMode: "exact_item" | "representative_item"   // exact_item: this
                                     // offer is for the specific physical
                                     // unit(s) referenced above (a single
                                     // rooted cutting); representative_item:
                                     // this offer represents a stock class,
                                     // fulfilled from any matching unit
  sku: string
  barcode: string | null
  price: { amount: number, compareAtAmount: number | null, currency: string }
  quantity: number
  inventoryPolicy: InventoryPolicy
  weight: { value: number, unit: WeightUnit }
  dimensions: { length: number, width: number, height: number, unit: DimensionUnit } | null
  shippingProfile: ShippingProfileRef
  taxCode: string | null
  complianceAttributes: Json         // marketplace/region-specific
                                     // compliance flags (e.g. hazmat,
                                     // restricted-plant/agricultural)
  optionValues: { name: string, value: string }[]   // e.g. {size: "4in pot"}
  mediaRefs: PhotoAssetId[]          // variant-specific images, distinct
                                     // from the package's general image set
  availability: "available" | "backordered" | "discontinued" | "sold_out"
  marketContext: { marketId: string, currency: string }[] | null   // only
                                     // where a marketplace supports
                                     // per-market pricing/currency
}
```

**Codex Correction #3 — field-level lineage.** A package-wide `evidenceRefs` array cannot answer "why is this exact value here" for a specific field, which is precisely the auditability §18's `SelectionTrace` is meant to provide at the recommendation level and which a flat array loses at the package level. Every field or content block below is individually addressable and carries its own `FieldLineage`:

```
FieldLineage {
  fieldPath: string                 // e.g. "content.price",
                                     // "offers[3].price", "content.title"
  claimIds: ClaimId[]                // ≥1, except where the field is a
                                     // direct owner override (see below)
  recommendationId: RecommendationId | null
  selectionTraceId: SelectionTraceId | null   // §18 — resolvable for any
                                     // field that came from a Recommendation
  confidence: Confidence
  appliedPolicyVersions: PolicyId[]
  ownerDecisionRef: OwnerDecisionId | null   // set, and claimIds/
                                     // recommendationId may be empty, only
                                     // when the field is a direct owner
                                     // override with no upstream claim
}
```

```
CommercePackage {
  id: CommercePackageId
  skuId: SkuId
  version: number                  // monotonic per SKU
  status: "draft" | "validated" | "approved" | "superseded"
  content: {
    title, description, salesSummary: string
    tags: string[]
    taxonomy: CanonicalTaxonomyRef      // not a marketplace-specific category
    images: { assetRef: PhotoAssetId, order: number, altText: string }[]
    seoTitle, seoDescription: string
    careContent: string | null
    recommendations: RecommendationId[]   // referenced, not embedded —
                                           // each still queryable with its
                                           // own SelectionTrace (§18)
  }
  offers: VariantOffer[]            // ≥1; price/weight/inventory/shipping
                                     // now live per-offer, not as
                                     // package-level scalars — see
                                     // Correction #2 above. A single-
                                     // variant SKU has exactly one offer.
  fieldLineage: FieldLineage[]      // replaces the old package-wide
                                     // evidenceRefs array; one entry per
                                     // addressable field/content block,
                                     // including one per offer field —
                                     // see Correction #3 above
  complianceResults: ComplianceCheckId[]   // §12, one per target marketplace
  readinessGateResult: GateResult
  createdAt: Timestamp
  supersedes: CommercePackageId | null
}
```

**Adapters are pure functions**: `(CommercePackage, marketplace-specific config) → MarketplaceListingPayload`. A Shopify adapter and a future Etsy adapter are equally thin siblings off the same canonical input — neither may query `commerce_details`, `listing_packages`, or any other business-truth table directly. This is the concrete fix for the Q9 gap Revision 2's own §11 named: today's Shopify publisher does exactly the thing this section forbids. With `offers[]` now representing every variant explicitly, an adapter maps each `VariantOffer` to its marketplace's own variant/inventory-item shape mechanically — it still never invents a price, quantity, or shipping value that isn't already present on the offer it's mapping.

Every `MarketplacePublicationAttempt` (§5) created from a `CommercePackage` version, on success or failure, is itself the artifact §13's ninth required item points to — the package and its field lineage answer "what was intended and why"; the attempt record answers "what actually happened when we tried to publish it."

## 8. `SourceConnector` contract

*Correction #3, Decision #3. Expanded by Codex Correction #5.*

```
interface SourceConnector {
  readonly name: string
  readonly capabilities: {
    read: boolean
    writeback: boolean            // false by default; must be explicitly
                                   // scoped per connector (e.g. Skrybix's
                                   // connector might enable only
                                   // "acknowledge" and "sale reconciliation"
                                   // writeback, mirroring the existing
                                   // acknowledge-after-durable-save pattern
                                   // already proven in lib/sources/skrybix.ts)
    changeFeed: boolean
  }

  // Every call is environment-scoped (§25) — a connector configured for
  // the "test" environment must never be able to fetch into or write back
  // against a production source record, and vice versa. This is enforced
  // by the connector's own configuration, not left to the caller to
  // remember.
  readonly environment: Environment

  // Codex Correction #5 — versioning and liveness, required so a caller
  // (or CI) can detect an incompatible connector/source schema change or a
  // dead connector before it silently returns stale or malformed data.
  readonly connectorSchemaVersion: string
  readonly sourceSchemaVersion: string
  health(): Promise<{ status: "healthy" | "degraded" | "down",
                       detail: string, checkedAt: Timestamp }>

  getRecord(sourceRecordId: string): Promise<SourceRecordSnapshot>

  // Codex Correction #5 — cursor-based pagination replaces an unbounded
  // listChangesSince return; the caller always receives an explicit next
  // cursor, including when there are currently no further changes (a
  // repeatable cursor, not an implicit "empty means done").
  listChangesSince(cursor: string | null):
    Promise<{ changes: SourceRecordSnapshot[], nextCursor: string,
              hasMore: boolean }>

  enumerateEvidence(sourceRecordId: string): Promise<EvidenceSourceRef[]>

  // Codex Correction #5 — identifier resolution is its own method,
  // returning candidates with confidence rather than a single assumed
  // match, since a source system's natural key is not guaranteed to map
  // 1:1 to a canonical SKU/ProductConcept (§5's merge/split model).
  resolveIdentifier(sourceRecordId: string):
    Promise<{ candidates: { entityType: EntityType, entityId: EntityId,
                             confidence: Confidence }[] }>

  // Only callable if capabilities.writeback is true for this connector.
  // Codex Correction #5 — writeback now takes an approved command type
  // (an enumerated, per-connector allowlist, e.g. "acknowledge" |
  // "reconcile_sale") rather than a generic payload, carries a caller-
  // supplied idempotency key so a retried call cannot double-write, and
  // returns a structured receipt plus explicit retry/error semantics
  // instead of a bare success/failure.
  writeback(sourceRecordId: string, command: ApprovedWritebackCommand,
            idempotencyKey: string):
    Promise<WritebackReceipt>
}

SourceRecordSnapshot {           // immutable — every fetch creates a new
  sourceSystem: string           // one, never mutates a prior snapshot,
  sourceRecordId: string         // the same discipline OneDrive's evidence
  snapshotAt: Timestamp          // revisions use (§16)
  observedAt: Timestamp          // Codex Correction #5 — when the source
                                  // system itself claims this state became
                                  // true, distinct from snapshotAt (when
                                  // GM Commerce fetched it); the two can
                                  // legitimately differ under retry/backfill
  contentHash: string
  payload: Json                  // raw, as received
  sourceProvenance: string       // Codex Correction #5 — exactly which
                                  // endpoint/export/feed within
                                  // `sourceSystem` this came from
  accessClassification: "public" | "internal" | "restricted"   // Codex
                                  // Correction #5 — drives who/what may
                                  // read this snapshot, independent of
                                  // RecordContext's environment scoping
  extractedClaims?: ClaimDraft[] // optional post-processing output
}

WritebackReceipt {               // Codex Correction #5
  idempotencyKey: string
  status: "applied" | "already_applied" | "rejected" | "failed_retryable" |
          "failed_permanent"
  remoteReference: string | null
  appliedAt: Timestamp | null
  errorDetail: string | null
  retryAfter: Duration | null    // set only when status is failed_retryable
}
```

Identity resolution: a connector's `(sourceSystem, sourceRecordId)` pair maps to a canonical `SourceRecord` entity (§5); the further mapping to a `SKU`/`ProductConcept` is `resolveIdentifier`'s job above — a separate, explicit, confidence-scored operation, never assumed 1:1, since a merge/split (§5) can change that mapping without touching the immutable snapshot history. Reads and writeback are structurally separate methods with independently gated capability flags, so a connector added purely for research (a supplier catalog feed, say) can never accidentally acquire write access by virtue of implementing the same interface. Writeback remains disabled by default per connector and independently authorized (§15's RBAC) regardless of the approved-command-type allowlist above.

## 9. Intelligence Repository service contract

*Correction #4, Decision #4. Completed by Codex Correction #4; cross-environment access restricted by Codex Correction #8.* A versioned command/query boundary — the "minimal Repository API" multiple parts of Copilot's review ask for is this same contract, not a separate one per caller (Skrybix, GM Commerce, OneDrive ingestion, audits, adapters, recommendations, and future systems all consume this same v1 surface):

```
interface IntelligenceRepositoryV1 {
  // Commands — all idempotent on (contentHash, correlationId); all emit an
  // audit event: {actor, role, command, correlationId, timestamp, result}.
  // Every write requires a RecordContext (§25) — there is no command that
  // creates a claim, correction, or decision without declaring its
  // environment, purpose, and eligibility up front.

  // --- Evidence (Codex Correction #4) ---
  ingestEvidence(evidence: EvidenceRevisionDraft, context: RecordContext,
                 correlationId: CorrelationId): Promise<EvidenceRevisionId>
  getEvidenceRevision(evidenceRevisionId: EvidenceRevisionId):
    Promise<EvidenceRevision>
  searchEvidence(query: EvidenceSearchQuery):   // scoped to the caller's
                  // own environment by construction, same as
                  // queryApplicableClaims below — no widening parameter;
                  // cross-environment search goes through
                  // requestCrossEnvironmentAccess like everything else
    Promise<EvidenceSource[]>

  // --- Claims ---
  proposeClaim(claim: ClaimDraft, context: RecordContext,
               correlationId: CorrelationId): Promise<ClaimId>
  verifyClaim(claimId: ClaimId, verifierActor: ActorRef,   // Codex
              verification: VerificationResult):            // Correction #4;
    Promise<Claim>                                           // gated by
                                                              // §10.1's
                                                              // thresholds
  supersedeClaim(claimId: ClaimId, supersededBy: ClaimId,
                 reason: string): Promise<Claim>
  recordContradiction(claimIds: ClaimId[], detail: string):
    Promise<ContradictionId>
  resolveContradiction(contradictionId: ContradictionId,
                        resolution: ContradictionResolution):
    Promise<Contradiction>

  // --- Policies ---
  createPolicyVersion(policy: PolicyDraft, context: RecordContext):
    Promise<PolicyId>
  queryApplicablePolicies(scope: Scope, asOf: Timestamp): Promise<Policy[]>
  evaluatePolicy(policyId: PolicyId, subject: EntityRef):
    Promise<PolicyEvaluationResult>

  // --- Recommendations / packages ---
  createRecommendationRun(input: RecommendationRunInput,
                           correlationId: CorrelationId):
    Promise<RecommendationRunId>
  recordValidationResult(recommendationId: RecommendationId,
                          result: ValidationResult): Promise<void>
  getCompletedPackage(commercePackageId: CommercePackageId):
    Promise<CommercePackage>      // §7 — includes offers[] and
                                   // fieldLineage[] in full

  // --- Decisions and learning ---
  recordOwnerDecision(decision: OwnerDecisionDraft, context: RecordContext):
    Promise<OwnerDecisionId>
  recordCorrection(correction: CorrectionDraft, context: RecordContext):
    Promise<CorrectionId>
  proposeCorrectionScope(correction: CorrectionId):   // §14's eligible-
    Promise<{ eligibleScopes: Scope[],                // scope derivation,
               requiresConfirmation: boolean }>        // exposed as a query
  activateRule(ruleProposal: RuleProposalId,
               confirmedBy: OwnerDecisionId): Promise<Rule>   // §14 — never
                                                        // callable without
                                                        // an OwnerDecision
                                                        // for any rule
                                                        // broader than
                                                        // SKU/InventoryItem
  revokeRule(ruleId: RuleId, reason: string,
             actor: ActorRef): Promise<void>

  promoteEvidence(evidenceRevisionId: EvidenceRevisionId):
    Promise<PromotionResult>      // gated on context.eligibility
                                   // .knowledgePromotion, §25

  // --- Lineage ---
  getAuditTrail(correlationId: CorrelationId): Promise<AuditEvent[]>
  getMarketplacePublicationLineage(marketplacePublicationAttemptId: MarketplacePublicationAttemptId):
    Promise<{ commercePackage: CommercePackage,
               fieldLineage: FieldLineage[],
               attempt: MarketplacePublicationAttempt }>   // §5, §7, §13
  getEditorialPublicationLineage(editorialAssetId: EditorialAssetId):
    Promise<{ asset: EditorialAsset, claimIds: ClaimId[],
               approvalHistory: OwnerDecisionId[] }>        // §5, §18

  // Queries — always return contradictions explicitly; never resolve them
  // server-side before returning. Codex Correction #8: `environmentFilter`
  // is REMOVED as an ordinary widening parameter here — a production
  // caller receives production-scoped results only, by construction.
  // Cross-environment access (audited promotion, authorized diagnostics,
  // or fixture evaluation) is a separate, privileged operation — see §25 —
  // never an optional parameter on this query.
  queryApplicableClaims(subject: EntityRef, scope: Scope):
    Promise<{ claims: Claim[], contradictions: Contradiction[] }>
  queryActiveRules(scope: Scope): Promise<Rule[]>   // production-only by
                                                     // construction — a
                                                     // test-context rule
                                                     // can never be active
  getPolicyVersion(policyId: PolicyId, asOf: Timestamp): Promise<Policy>
  getSelectionTrace(recommendationId: RecommendationId):
    Promise<SelectionTrace>       // §18

  // Codex Correction #8 — the only sanctioned cross-environment path.
  // Distinct from every command/query above, which are single-environment
  // by construction. Requires its own elevated authorization (§15) beyond
  // whatever role gates the equivalent same-environment operation.
  requestCrossEnvironmentAccess(request: {
    actor: ActorRef
    reason: string
    purpose: "audited_promotion" | "authorized_diagnostics" | "fixture_evaluation"
    sourceEnvironment: Environment
    destinationEnvironment: Environment
    subject: EntityRef | ClaimId[]
    correlationId: CorrelationId
  }): Promise<{ result: Json, auditEventId: AuditEventId }>
  // Diagnostic reads returned by this operation are structurally
  // ineligible to feed a production recommendation run or a
  // knowledgePromotion transition — enforced by the caller-role check on
  // createRecommendationRun/promoteEvidence, not by caller discipline.

  // Contract metadata
  readonly schemaVersion: string  // this interface's own version, with a
                                   // defined backward-compatibility window
}
```

Authorization: every call carries a caller identity + role (§15's RBAC), checked before the command/query executes. A batch of claim writes belonging to one recommendation run is transactional — either all commit or none do, threaded by one `correlationId` per run, which is also what stitches together §13's nine required artifacts after the fact. Error semantics are structured codes (`NOT_FOUND`, `CONFLICT`, `POLICY_VIOLATION`, `STALE_VERSION`, `UNAUTHORIZED`), never a bare exception a caller has to string-match.

## 10. Research layer: retrieval, verification, provenance, and ingestion security

**Six source categories** (the commerce-fact-ownership taxonomy, unchanged in substance since Revision 1, stated here in full per Codex's documentation correction rather than by reference to Revision 2's history):

1. Skrybix inventory/cutting data.
2. Product/SKU commerce data (GM Commerce's own stored facts).
3. Gathering Moss store/category defaults (policy, deterministic).
4. Deterministic derived rules (e.g., exact-item disclosure from source system).
5. AI research (verified, cited, confidence-scored).
6. Genuine per-item human exception.

**Eight evidence types** (a different, complementary axis: the *epistemic status* of a piece of knowledge, independent of which of the six categories produced it):

1. External published research.
2. Established horticultural consensus.
3. Supplier/manufacturer claims.
4. Community reports (lower default authority — unverified by construction until corroborated).
5. Gathering Moss's own observations.
6. Gathering Moss's own controlled experiments (highest-authority *research*-type evidence, since Phil/Crystal directly produced it).
7. Owner decisions (outranks all research-type evidence per §6's precedence rule).
8. AI-derived recommendations (never their own evidence — see §18's protections).

A single claim's source category (the six above) and evidence type (the eight above) are recorded together — e.g., a care claim might be `source: AI research`, `evidenceType: established horticultural consensus`, with a specific evidence span and citation.

Added this revision, per Correction #9 and the "additional requirements" list, and substantially expanded by **Codex Correction #11**:

**Untrusted-content handling for research ingestion.** Delimiters alone are not sufficient prompt-injection protection — Codex's review is explicit that a determined adversarial document can still influence a call that merely wraps retrieved text in markers. The full pipeline every research/ingestion call must follow:

1. **Raw retrieved content is stored separately from everything else** — its own record (an `EvidenceRevision`, §17), never inlined into a working prompt buffer shared with instructions or other content.
2. **Deterministic normalization and sanitization run before any extraction** — encoding normalization, stripping of non-content control sequences, and structural parsing appropriate to the content type (HTML, PDF, OCR output) happen before an LLM ever sees the content, not as an afterthought.
3. **Retrieved text is never placed in the system or developer instruction channel** — only in a clearly-scoped user/content channel, structurally separated by the provider API's own role mechanism, not by an in-band delimiter alone.
4. **Extraction produces bounded structured claim candidates**, not free-form model output — the model's job on a retrieval call is to fill a constrained schema (claim predicate/value/evidence-span candidates), not to converse or follow embedded instructions.
5. **Validators receive only the candidate claim and its minimal cited evidence span** — never the full retrieved document and never the original generation/ingestion context, so a validator has nothing to be manipulated by beyond the narrow thing it's checking.
6. **Retrieved instructions are ignored and logged as suspicious content.** Any retrieved text that reads as an instruction directed at the pipeline (a "system prompt," a request to ignore prior instructions, an embedded command) is never followed, and its presence is itself recorded as a suspicious-content event for review — the same discipline this document's own instruction-source-boundary rule applies to Claude's own tool use, now formalized for the pipeline's research layer.
7. **Automatic promotion is limited by source class and verification policy** — a claim extracted from an unvetted or low-authority source can never auto-promote regardless of how confidently it's phrased; §10.1 defines the actual thresholds.
8. **Adversarial prompt-injection fixtures are required test coverage**, spanning web pages, PDFs, OCR output, supplier files, and OneDrive documents — a passing test suite for this layer must include documents that actively attempt to redirect pipeline behavior, not only well-formed benign content.

**Retrieved content is data, never instructions**, restated as the governing principle the eight-step pipeline above implements: every research call structurally separates the fixed system prompt from retrieved external content, and no retrieved text is ever treated as a command, regardless of what it appears to say.

**Source allowlists.** Research begins against a curated, versioned set of source types/domains, not an open web crawl — expanding the allowlist is itself a policy change (§14), reviewable the same as any other.

**Independent-source-family detection, based on lineage, not URL count.** Two URLs on different domains that both mirror the same original wire article or the same supplier press release are one source family, not two independent confirmations — detected by content-similarity/lineage comparison across `EvidenceRevision`s, not by counting distinct hostnames. This directly answers the review's explicit "independent-source detection based on source families and lineage" requirement and closes a real credibility gap: three sites repeating one original claim must never be scored as three-source consensus.

**Operational controls**, all real requirements for a service that makes paid, rate-limited calls: caching and deduplication (don't re-fetch identical content), rate limits and budgets per research run, a queue with bounded concurrency, retries with backoff, dead-letter handling for a request that can't complete, and per-run cost/latency telemetry — the same discipline §19's catalog-scale requirements demand of the pipeline generally, applied specifically to research.

**Verification and conflict detection** carry forward from Revision 2 §6 unchanged: source ranking, citations with evidence spans, freshness, explicit rejected-source reasoning, and a precedence rule (owner decision > verified research > single-source claim > unverified assertion) that resolves a contradiction only when a resolution is actually needed — both sides of a genuine disagreement are retained as claims, never silently dropped.

### 10.1 Verification and promotion thresholds

*Codex Correction #9.* "No unresolved contradiction" is necessary but not sufficient to mark a claim `verified` (§18's lifecycle) — the review is explicit that a claim can be free of contradiction and still be under-evidenced. This section defines the actual bar, by claim class:

- **Minimum corroboration by claim class:**
  - *Deterministic/derived claims* (source categories 3–4, §10): zero additional corroboration required — the deterministic rule that produced them is itself the evidence, per §6's `evidenceAnchors` exception for this category.
  - *General botanical/care claims* (evidence types 1–2, 5): at least two independent source families (see below), or one source meeting the "authoritative primary source" bar defined next.
  - *Commerce-consequential claims* (price, weight, shipping, compliance, safety, marketplace policy) — **corroboration depends on where the claim actually comes from, not on a blanket rule that treats the whole category alike** (corrected per Codex's targeted verification of the prior wording, which over-escalated):
    - A fact obtained **directly from the authoritative system of record** for that fact type — Skrybix for a plant's physical state, a Gathering Moss controlled measurement for weight/dimensions, the marketplace's own current published policy for a compliance rule — requires no additional corroboration and may reach `verified` automatically once it is fresh, uncontradicted, correctly source-classified, and fully traced back to that source (§6's `evidenceAnchors`, §7's `fieldLineage`).
    - A claim applying an **existing, already-versioned Gathering Moss policy** (inventory policy, standard shipping language, exact-item disclosure rules, approved category defaults) is corroborated by the policy version itself (§6's `policySnapshotId`) — applying a policy that already exists is not a new policy decision and needs no re-corroboration each time it's applied to a new SKU.
    - An **AI-generated recommendation** for one of these fields (most consequentially, price) is never treated as a verified fact on its own regardless of confidence — it must be evidence-backed and independently validated (§10, §11 of Revision 2) and carried into the completed package (§19) with its reasoning, evidence, and confidence intact; it reaches Phil once, as part of completed-package approval — never as a separate per-field fact-entry or knowledge-check step.
  - *Community-report claims* (evidence type 4): never independently verified — always require corroboration from a higher-authority evidence type before `verified`, consistent with their lower default authority.
- **Independent-source-family requirement:** per §10's lineage-based detection, corroboration must come from genuinely distinct source families, not multiple mirrors of one origin — a claim citing three same-family sources counts as single-source for this threshold.
- **Authority thresholds:** an "authoritative primary source" is the system of record for that specific fact type (see the commerce-consequential bullet above) — a claim citing only that source may be verified without additional corroboration; a claim citing a secondary or tertiary source never qualifies for the single-source path regardless of how many secondary sources agree.
- **Freshness requirement:** a claim's `freshnessPolicy` (§6) must not be expired at the moment `verified` is evaluated — an otherwise-qualifying claim past its freshness window is `STALE` for verification purposes and must be revalidated, not grandfathered in.
- **Automated vs. verifier vs. Phil-required transitions:**
  - *May be automated* (system executes `verifyClaim` without a human call): deterministic/derived claims meeting the corroboration bar above; general botanical/care claims meeting the two-independent-source-family bar with no open contradiction and no expired freshness; and, per the commerce-consequential bullet above, authoritative-system-of-record facts and existing-policy applications meeting the same freshness/traceability bar. **Routine authoritative facts are never escalated to Phil merely because the field they populate is commercially consequential** — doing so would recreate the exact bottleneck this document's own governing completion test exists to eliminate.
  - *Requires an authorized verifier* (a role with the `verify claim` grant, §15 — may be Crystal or a future task-scoped role, not necessarily Phil): a commerce-consequential claim citing a secondary/tertiary source rather than its authoritative primary source, and any claim that met corroboration only after a contradiction was resolved (§6).
  - *Always requires Phil*: genuinely new or changed business policy (as distinct from applying an existing, already-versioned one); a material unresolved conflict (§6); exceptional safety or legal judgment beyond what an authoritative source itself already states; an identity merge/split decision (§5); activating any durable rule broader than SKU/InventoryItem scope (§14); and — once per pipeline run, not once per field — **final completed-package approval** (§19), which is where an AI-generated price recommendation (or any other recommendation) actually receives Phil's sign-off, as a single assembled-result approval. A material exception surfaced during that same review (a failed compliance check, an unresolved contradiction, a flagged safety concern) is escalated as part of that one review, never spun out into a separate, repetitive per-claim approval task.
- **Behavior when no adequate verification is available:** the claim remains in `under_review` indefinitely — never defaults to `verified` on a timeout, never defaults to rejected either (a genuinely unresourced verification is a known-unknown, not a negative finding), and is surfaced as a standing item on whichever review surface (§19's completed-package shell, or a future dedicated queue) tracks unresolved verification work.

## 11. Image understanding: vision-provider contract and inference boundaries

Expands Revision 2 §4's image-understanding layer with the two things Copilot's review specifically asked for.

**Vision-provider and authorized media-access contract.** A vision request is authorized against a specific, enumerated set of `PhotoAssetId`s belonging to the product under review — never an arbitrary URL, and never a request the pipeline itself didn't originate. Same operational controls as the research layer (§10): rate limits, cost telemetry, caching (don't re-analyze an unchanged photo).

**Allowed inferences:** whether a photo shows a plant versus a pot versus is simply unusable (blur, exposure, wrong subject); rough composition/framing suitability for a hero slot; whether two photos appear to be the same angle (duplicate-view detection); whether an expected shot type (a root-system view, for a rooted-cutting claim) appears to be present at all.

**Prohibited inferences**, stated as explicitly as the allowed list, because this is where an under-specified vision layer becomes a *new* fabrication surface rather than closing the old one: diagnosing plant health or disease from pixels alone without a human confirmation step; asserting an exact species/cultivar from a photo when it disagrees with Skrybix's own recorded identity (a disagreement here is a contradiction claim, §6, routed to Phil — never silently resolved in either direction); inferring an exact count, measurement, or dimension from a photo with no reference scale present. Every vision-derived claim carries the same confidence and evidence-anchor requirements as any other claim (§6) — a vision call never gets to assert something with unearned certainty just because it "looked at the picture."

## 12. Marketplace-policy compliance gate

*Correction #6.* Compliance is pass/fail, not a recommendation a human weighs — the review correctly separates this from §7's recommendation content, and Revision 2 had folded it into the recommendation list, which this section corrects.

```
ComplianceCheck {
  id: ComplianceCheckId
  commercePackageId: CommercePackageId
  policySnapshotId: PolicyId        // the exact policy version checked against
  marketplace: string
  shop: string
  region: string | null
  checkedAt: Timestamp
  freshnessTtl: Duration            // a stale check is not a passing check
  result: "PASS" | "FAIL" | "STALE"
  violations: { rule: string, detail: string, adapterMapping: string }[]
}
```

Fail-closed behavior: a `CommercePackage` cannot reach `approved` status while its `ComplianceCheck` for the target marketplace is `FAIL` or `STALE`. A marketplace policy change (a new Shopify content requirement, an Etsy category restriction) invalidates every `ComplianceCheck` scoped to that marketplace whose `policySnapshotId` predates the change — they become `STALE` immediately, not silently still-passing, which is the "marketplace-policy freshness and blocking behavior" the review asked for as an additional requirement.

## 13. Required persisted artifacts

*Correction #7.* Every recommendation run must persist all nine of the following, threaded by one `correlationId` (§9). **A run missing any one of these artifacts is itself a readiness-gate failure** — not a warning, not a "best effort" — mirroring the audit's own release rule that no field is "complete" merely because something was typed into it.

1. Source snapshots and content hashes (§8's `SourceRecordSnapshot`).
2. Claims, citations, and evidence spans (§6).
3. Policy and taxonomy versions used (§7's `policySnapshotId`, §12).
4. Research queries, sources considered, rejected sources, and rejection reasons (§10).
5. Model, provider, and prompt-template versions.
6. Deterministic validation results (§11's readiness/repair checks, formalized as their own artifact rather than folded silently into a pass/fail).
7. The persisted `CommercePackage`, its complete `fieldLineage[]`, field-level confidence, and resolvable `SelectionTrace` records for every field that came from a `Recommendation` (§7's `fieldLineage[]` — which replaced the package-wide `evidenceRefs` array under Codex Correction #3 — and §18's `SelectionTrace`).
8. Owner overrides with rationale and scope (§14's `Correction` record).
9. The final marketplace payload, its content hash, and the remote verification receipt — now the canonical `MarketplacePublicationAttempt` entity (§5, §7), not an ad hoc log line: the pattern already proven in this session's real Shopify verification (field-by-field confirmation against Shopify's own response) generalized into a required, immutable, queryable artifact rather than a one-off manual check.

## 14. Owner-authority, durable learning, and correction-scope inference

*Correction #11, Decision #6.* Refines Revision 2 §7 with the exact mechanics the review asked for:

- **Eligible scopes are derived from the canonical entity hierarchy (§5) and field semantics**, not offered as an undifferentiated list. A care-instruction correction's eligible scopes are `{this SKU, this ProductConcept, this genus/species if known, all plants}` — a price correction's eligible scopes never include a botanical grouping, because price isn't a botanical property. The eligible-scope set is computed from what kind of predicate is being corrected.
- **Default to `SKU`/`InventoryItem` scope when uncertain.** A correction never auto-generalizes; broader scope is always a proposal, never a default.
- **Any new rule broader than `SKU`/`InventoryItem` requires explicit owner confirmation** before it activates.
- **Price, safety, compliance, marketplace, editorial, brand, and policy corrections always require confirmation to become a standing rule**, regardless of how narrow their proposed scope is — these categories are consequential enough that even a SKU-scoped new *rule* (as opposed to a one-off correction to that single item, which needs no confirmation at all) needs Phil's sign-off.
- **Item-level owner overrides are always preserved.** A later broader rule can never silently supersede an explicit override Phil made for one specific item — the override's `OwnerDecision` outranks any subsequently-created rule for that item by construction (§6's precedence model).
- **Positive and negative replay tests are required before a new rule activates**: a fixture case the rule *should* apply to, and a fixture case it should *not* — both asserted, not just the positive case, mirroring the audit's own "Owner-correction learning" evaluation criterion exactly.
- **Sales-performance correlation is never promoted directly into an autonomous rule.** A performance observation can produce a *recommendation* or a *proposal* for Phil to confirm; it can never become an active rule on its own, closing the correlation-versus-causation gap the review named explicitly.

### 14.1 Phase B0 correction sequencing: `LegacyCorrectionEvent`

*Codex Correction #10.* §19 places completed-package review and correction-*capture* in Phase B0, deliberately before Phase B builds the canonical entity/claim model — but a canonical `Correction` record (§5) references a `Claim` or `Recommendation` ID, neither of which exists yet in Phase B0. Without a fix, §19's own "every edit made through the legacy form is also captured as a correction event" instruction has no valid record type to write into. A temporary, explicitly-scoped type closes the gap:

```
LegacyCorrectionEvent {
  id: LegacyCorrectionEventId
  legacyPackageIdentifier: string   // whatever identifies the edited
                                     // record under the pre-Phase-B model
                                     // (today: a listing_packages row id)
  legacyFieldIdentifier: string     // the specific field/column edited
  beforeValue: Json
  afterValue: Json
  actor: ActorRef
  reason: string
  timestamp: Timestamp
  context: RecordContext            // §25 — environment and purpose apply
                                     // to a LegacyCorrectionEvent exactly
                                     // as to any other record; a
                                     // test-environment edit produces a
                                     // test-context event, never eligible
                                     // for migration below
  correlationId: CorrelationId
  migrationStatus: "pending" | "migrated" | "ineligible"
}
```

**Migration into canonical `Correction` records.** Once Phase B's entities and claims exist for the record a `LegacyCorrectionEvent` describes, a migration job attempts to resolve `legacyPackageIdentifier`/`legacyFieldIdentifier` to a canonical `(EntityType, EntityId, predicate)` and, where a corresponding `Claim` or `Recommendation` can be identified, creates the equivalent `Correction` record referencing it, then marks the event `migrated`. Where no canonical claim/recommendation can be resolved (the field never got a canonical equivalent, or the record was retired before migration ran), the event is marked `ineligible` and retained as history — never silently dropped. **Test-derived `LegacyCorrectionEvent`s are always `ineligible`** — a `context.recordPurpose` other than `"operational"` (§25) excludes an event from migration into a canonical `Correction` unconditionally, the same rule §25's hard invariants already apply to test-context claims generally. This is Phase B0's answer to Codex's sequencing objection: correction signal is captured from day one of B0 in a type that's honest about not yet being canonical, and migrates forward once the model it needs actually exists, rather than either being lost during the transition or forcing a premature `Correction` schema into Phase B0.

## 15. Role-based access control

*Correction #10.* Roles, and what each may do — Phil is the only role no other role's action can ever override:

| Role | Ingest evidence | Propose claim | Verify/promote evidence | Change policy | Override claim/recommendation | Approve commerce draft | Approve public content | Issue correction | Retract/supersede knowledge |
|---|---|---|---|---|---|---|---|---|---|
| **Phil (Owner)** | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes — final authority |
| **Crystal (Co-owner/Staff)** | Yes | Yes | Yes | No (propose only) | Item-scoped only | Yes, within defined scope — mirrors the existing "Phil or Crystal approves" pattern | Per editorial role grant | Yes | Item-scoped only |
| **Future task-scoped staff** (e.g. photography-only) | Scoped to grant | No | No | No | No | No | No | No | No |
| **Service/system identity** (the pipeline itself) | Yes (as itself, attributed) | Yes (as a proposal, never auto-approved) | No | No | No | No | No | No | No |
| **Public/editorial reviewer** (may differ from commerce approvers over time) | No | No | No | No | No | No | Yes, within editorial workflow (§17) | No | No |

Every role's actions are logged with actor + role on the same audit-event stream §9's Repository contract already requires; no permission here is enforced only by UI — the Repository service contract (§9) checks authorization at the command layer regardless of which client calls it.

## 16. Skrybix data and workflow gap analysis

Unchanged from Revision 2: taxonomy, synonyms, physical observations, dimensions, weight, cost, live inventory state, photo associations, sales/outgoing reconciliation, lifecycle data, provenance, and marketplace feedback are all currently absent from Skrybix's commerce-export contract (`lib/sources/skrybix.ts`, confirmed by direct code read). §8's `SourceConnector` contract is the formal shape any of these additions would arrive through once proposed to Skrybix's own backlog — this revision doesn't change the gap analysis itself, only gives it a concrete interface to land against.

## 17. OneDrive evidence library

*Correction #8.* The two-layer split from Revision 2 (OneDrive as durable original-evidence storage; the Intelligence Repository as the structured reasoning layer that cites into it) is unchanged. This revision specifies the evidence model precisely:

```
EvidenceSource {                    // stable identity, independent of
  id: EvidenceSourceId              // filename or folder location
  sourceType: EvidenceType          // the 8-type taxonomy, Revision 2 §6
  currentRevision: EvidenceRevisionId
  acl: PermissionSet
  applicability: Scope              // plant/product/genus/species/
                                     // marketplace/policy, §5's Scope type
}

EvidenceRevision {                  // immutable once created
  id: EvidenceRevisionId
  evidenceSourceId: EvidenceSourceId
  oneDrivePath: string              // current location — see relocation
  oneDriveItemId: string            // OneDrive's own stable item ID,
                                     // authoritative over path
  contentHash: string
  extractionVersion: string         // which OCR/text-extraction pipeline
                                     // version produced derived text
  extractedText: string | null
  createdAt: Timestamp
  supersedes: EvidenceRevisionId | null
  promotionState: "discovered" | "under_review" | "verified" |
                  "promoted" | "superseded" | "retired"
}

EvidenceAnchor {
  id: EvidenceAnchorId
  evidenceRevisionId: EvidenceRevisionId
  locator: PageRef | CharRange | BoundingBox | Timestamp | ImageRegion
}
```

**Relocation behavior:** anchors and revisions resolve by `oneDriveItemId`, not by path — a file moved within OneDrive doesn't break a citation. **Duplicate detection** is content-hash based at ingestion. **Version handling**: a superseded revision is retained, never deleted, with `supersedes` pointing forward. **Revalidation jobs** run against each `EvidenceSource`'s `applicability`-appropriate freshness policy (a marketplace policy capture needs far more frequent revalidation than a general botanical reference).

**Ingestion pipeline**, unchanged in shape from Revision 2, now producing the typed artifacts above at each stage: discover/add source → preserve original in OneDrive (assign `oneDriveItemId`) → hash → extract text/media metadata (`extractionVersion`) → classify (`sourceType`, `applicability`) → create atomic claims (draft, `promotionState: discovered`) → verify against other sources (§10's reconciliation) → record citations/evidence spans (`EvidenceAnchor`) → promote (`promotionState: promoted`) → retrieve for future work → monitor freshness and revalidate.

## 18. Gathering Moss Intelligence Repository: lifecycle, protections, and explainability

**Knowledge lifecycle as an enforceable state machine**, not prose — the `promotionState` field above is the concrete mechanism:

```
discovered → under_review → verified → promoted → applied
                                            │
                                            ▼
                                  (outcome measured)
                                            │
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                         revalidated     revised       retired
```

Transitions are triggered by named actors: `discovered → under_review` happens automatically on ingestion; `under_review → verified` requires §10's reconciliation to find no unresolved contradiction; `verified → promoted` may be automatic for high-confidence, corroborated claims or may require Phil's confirmation per §14's rule (policy/price/safety/compliance/editorial/brand claims always do); `applied → (revalidated | revised | retired)` is driven by the freshness policy (§17) or by a new contradicting claim arriving.

**Protections against bad learning**, carried forward from Revision 2 unchanged and now enforceable by construction rather than by intention, because each one maps to a specific mechanism already specified above: unverified research cannot become accepted truth (the `promotionState` gate, §17); one-off corrections cannot be generalized without scope (§14); marketplace-specific rules cannot corrupt canonical product truth (`applicabilityScope`, §6, structurally separates the two); stale knowledge cannot remain active (`freshnessPolicy`/revalidation, §17); an AI-generated claim cannot be its own evidence (every claim requires ≥1 `evidenceAnchor` unless its source category is deterministic/derived, §6); repeated sources cannot be mistaken for independent verification (§10's lineage-based detection); performance correlation cannot become an autonomous rule (§14's explicit prohibition); no learned behavior can override Phil (§15's RBAC, enforced at the Repository command layer, §9).

**Selection traces and explainability.** Every `Recommendation` (§5) has exactly one `SelectionTrace`, queryable via `getSelectionTrace` (§9):

```
SelectionTrace {
  recommendationId: RecommendationId
  claimsConsidered: ClaimId[]
  claimsRejected: { claimId: ClaimId, reason: string }[]
  precedenceDecisions: { conflictBetween: ClaimId[], winner: ClaimId,
                          rule: string }[]
  confidence: Confidence
  freshnessAtDecision: Timestamp
  policiesApplied: PolicyId[]
  ownerDecisionAnchors: OwnerDecisionId[]
}
```

This is the concrete mechanism behind two things Revision 2's own §11 named as under-specified: source-precedence disclosure (Q8) and "why did the system pick this value" auditability generally. It's also part of the answer to "the minimal Repository API needed by Skrybix, GM Commerce, OneDrive ingestion, audits, adapters, recommendations, and future systems" — every one of those callers can ask the same question of the same contract.

**Public editorial safeguards**, per Correction #15: `EditorialAsset` (§5 — renamed from `Publication` per Codex Correction #1, which also separated marketplace publication into its own `MarketplacePublicationAttempt` entity so this section's guarantees apply only to genuine public editorial content, never to a commerce listing) is a versioned entity, same discipline as `CommercePackage` — every public factual claim requires claim-to-evidence validation before publish (same independent-validation pass as commerce copy, §11 of Revision 2, not a separate weaker check); citation, copyright, license, and attribution rules apply per `EvidenceSource.sourceType` (quoting a copyrighted reference at length even with a citation is still a license question, not just an attribution one); the editorial approver's identity is RBAC-tracked (§15) and may differ from commerce approvers; publication history is versioned and an `EditorialAsset` can be corrected, retracted, or taken down as a tracked event, never a silent delete, queryable via `getEditorialPublicationLineage` (§9); customer feedback is evidence-type `community_report` (§10's taxonomy) — research input to investigate, never accepted as truth until independently verified.

## 19. Completed-package review shell (Phase B0) and catalog-scale requirements

*Correction #13, Decision #7; Correction #12, Decision #5.*

**Phase B0 exists specifically to fix a sequencing risk Revision 2's own §11 named**: if the recommendation-rich backend (Phases B–F) ships before the review UI is redesigned (previously Phase H, sequenced last), the running system keeps concealing routine work in a raw edit form for months after the data model stops requiring it. Phase B0 moves the review **shell** — present-as-a-customer-would-see-it, an evidence/confidence panel, Approve/Reject/targeted-regenerate as the primary actions — immediately after Phase A, before the claim model itself exists, so every subsequent phase plugs real capability into an already-correct UI instead of retrofitting one at the end. Correction-event capture (§14's auto-recording) also starts in Phase B0, before claims exist to correct, so no learning signal is lost during the migration window: every edit made through the legacy form while it's still in use is *also* captured as a correction event. For products using the new model, raw free-text editing becomes an explicit "correction exception" action, never the default; the legacy edit form remains available only for legacy packages, only during a documented migration window with an explicit end date.

**Catalog-scale requirements**, evaluated against Phase B's schema design before implementation, not discovered as a performance problem afterward — the exact figures from Copilot's review, treated as non-functional requirements:

- At least 10,000 active SKUs, 50,000 inventory items, 1,000,000 claim/evidence relationships, 100,000 evidence revisions.
- Indexed applicable-claim and policy queries at p95 below 500ms.
- OCR, research, vision, reconciliation, and revalidation all run asynchronously — never inline in a request path.
- Cursors, queues, batching, bounded concurrency, dead-letter handling, resumable checkpoints, caching with explicit invalidation, and per-run cost/latency telemetry are required infrastructure, not later hardening.

This directly closes the catalog-scale gap Revision 2's own §11 flagged as entirely unaddressed.

## 20. Operational instrumentation

*Correction #14. Owner-minutes precision added by Codex Correction #12.* Every metric from Revision 2 §13 plus Copilot's precision requirements: owner minutes per SKU, unnecessary escalation rate (with necessity **classified at the moment the escalation occurs**, not inferred retroactively — closing the exact gap Revision 2's own §11 flagged), factual-error events, citation freshness, correction reuse, recommendation outcomes, performance observations, and staff-equivalent throughput per owner hour.

**Owner minutes per SKU, decomposed** (Codex Correction #12 — the prior single number conflated categories that need to be distinguishable): review time, approval/rejection time, correction time, escalation-resolution time, and required navigation-or-data-entry time are each their own tracked category, summed for the headline "owner minutes per SKU" figure but always queryable individually. **Physical product work — taking photographs, handling a cutting, packing a shipment — is classified separately from all five software-administrative categories above**, specifically so the system can distinguish unavoidable physical labor (which no software change reduces) from software-created administrative labor (which is exactly what this reset is meant to shrink); a metric that blends the two would make software-caused overhead invisible inside genuinely unavoidable physical time.

**Start/stop event semantics**, replacing duration-inferred-from-timestamps: each of the five administrative categories above is bounded by an explicit start event and stop event (e.g. `review_started` / `review_submitted`, `correction_started` / `correction_saved`, `escalation_raised` / `escalation_resolved`), both carrying `correlationId` and `actor`. Duration is the stop-minus-start delta for a matched pair, never inferred from the gap between two otherwise-unrelated timestamps (a prior approach that would misattribute idle time — a lunch break between opening a review and submitting it — as owner effort). A start event with no matching stop event within a defined timeout is reported as "session abandoned or unresolved," a distinct outcome from a fast completion, never silently excluded from the metric or counted as zero duration.

- **Baseline period:** the pre-reset historical average where derivable (git/session history for time-per-SKU under the fully manual flow); where not derivable, a defined first-N-SKU baseline measured going forward under Revision 3's own pipeline, stated explicitly as a baseline rather than compared against nothing.
- **Formulas** are defined per metric against the artifacts in §13 — e.g. owner minutes per SKU sums correction-event duration plus escalation resolution time, grouped by `correlationId`'s originating SKU.
- **Sampling:** 100% at current expected volume; revisited once §19's catalog-scale figures are approached.
- **Attribution:** every measurement attributes to a `correlationId` (§9), which threads back to the SKU/pipeline run that produced it.
- **Missing-data policy:** a period with no data is reported as "insufficient data," never defaulted to zero — a silent zero would read as a fabricated improvement.

## 21. Response to Copilot's ten-question critique (updated from Revision 2)

The original response table (Revision 2 §11) is preserved in git history at commit `a59f9d0`. This revision updates only the items that table named as genuine, unclosed gaps — each now has a concrete home:

| Revision 2's named gap | Resolved in Revision 3 |
|---|---|
| No formal canonical-entity schema (Q9) | §5 — eighteen typed entities (originally seventeen; Codex Correction #1 split `Publication` into `MarketplacePublicationAttempt` and `EditorialAsset`), cardinalities, stable IDs, lifecycle ownership, merge/split resolution |
| No marketplace-neutral listing representation distinct from Shopify's own types (Q9) | §7 — `CommercePackage`, adapters as pure functions |
| No unified `SourceConnector` interface (Q9) | §8 |
| No catalog-scale non-functional requirement (Q9) | §19 |
| Marketplace compliance folded into recommendations rather than its own gate (Q4) | §12 — distinct pass/fail `ComplianceCheck` |
| §13 (was §13 in Revision 2) referenced Copilot's artifacts as a category without itemizing all nine (Q5) | §13 — all nine enumerated, with "missing artifact = readiness failure" stated explicitly |
| Correction scope-inference mechanism unspecified (Q6) | §14 |
| Source-precedence reasoning not required in review presentation (Q8) | §18's `SelectionTrace`, queryable per recommendation |
| Sequencing risk: recommendation backend could ship well before review-UI redesign (Q7) | §19 — Phase B0 moved immediately after Phase A |
| Escalation-necessity tagging not required at time of escalation (Q10) | §20 |

All ten questions' full original answers still stand; none were substantively wrong, only some named a gap this revision now closes rather than leaves open.

## 22. Traceability: Copilot's fifteen corrections

| # | Correction | Revision 3 section(s) | Concrete artifact/interface | Acceptance test | Phase | Residual risk |
|---|---|---|---|---|---|---|
| 1 | Canonical entity hierarchy | §5 | 17 typed entities, `EntityRef`, merge/split as `OwnerDecision` | Fixture asserting a merge reassigns claim subjects and retains the superseded ID as a resolvable alias | B | Entity boundaries (e.g. where `Variant` applies) may need refinement against real non-plant catalog data not yet modeled |
| 2 | Marketplace-neutral `CommercePackage` | §7 | `CommercePackage` schema; adapters as pure functions | A fixture Etsy-shaped adapter (stubbed) built from the same `CommercePackage` instance as the Shopify adapter, asserting neither reads `commerce_details` directly | B/F | No second real marketplace exists yet to prove the adapter boundary holds under a genuinely different platform's constraints |
| 3 | `SourceConnector` contract | §8 | `SourceConnector` interface, `SourceRecordSnapshot` | A fixture connector (mock Skrybix) proving read/writeback capability gating and snapshot immutability | B | Migrating the real `lib/sources/skrybix.ts`/`sku-generator.ts` onto this interface is real, unstarted work |
| 4 | Repository service contract | §9 | `IntelligenceRepositoryV1` interface | A contract test asserting every command emits an audit event and every query surfaces contradictions | B | Schema-version backward-compatibility policy is named but not yet exercised across a real version bump |
| 5 | Claim/evidence schema + legacy migration | §6, §1 | `Claim` schema; dual-write/backfill/rollback/cutover/retirement plan | Backfill validation test: reconstructed claim-derived value matches the one existing legacy `commerce_details` row exactly | A (migration-file reconciliation) → B (full model) | The undocumented live-database migration (§1) must be resolved before this migration plan has anything real to migrate from |
| 6 | Marketplace-policy compliance gate | §12 | `ComplianceCheck`, fail-closed on `FAIL`/`STALE` | Fixture: a policy version bump immediately marks prior checks `STALE`, blocking approval until re-checked | E | No real second marketplace's policy set exists yet to validate the model against |
| 7 | Nine required persisted artifacts | §13 | Enumerated list, tied to `correlationId` | A run missing any one of the nine fails the readiness gate — asserted as a negative fixture | B–F (artifacts land as each producing phase lands) | Artifact 9 (remote verification receipt) depends on Phase F's real marketplace publish existing to verify against |
| 8 | OneDrive evidence model | §17 | `EvidenceSource`/`EvidenceRevision`/`EvidenceAnchor`, relocation-safe via `oneDriveItemId` | Fixture: moving a file within OneDrive doesn't break an existing `EvidenceAnchor` resolution | C | OneDrive API rate limits/quotas at real evidence-library scale are unverified |
| 9 | Research-ingestion security | §10 | Delimited untrusted-content wrapping, source allowlist, lineage-based independent-source detection | Adversarial fixture: a retrieved document containing an embedded instruction must not alter pipeline behavior | C/D | Lineage/similarity detection quality is unproven without a real corpus of mirrored sources |
| 10 | RBAC | §15 | Role/permission matrix, enforced at the Repository command layer | A fixture asserting a Staff-role call attempting a policy change is rejected with `UNAUTHORIZED` | B (enforcement wired into §9's contract) | Role set may need to grow before public/editorial workflows (§17) are real |
| 11 | Correction-scope inference | §14 | Eligible-scope derivation from entity+predicate, confirmation rules, replay-test requirement | Positive/negative replay fixture pair required before any new rule activates, per the correction itself | B0 (capture) / G (rule activation) | Scope-derivation logic itself needs a defined ruleset per predicate type, not yet enumerated exhaustively |
| 12 | Catalog-scale requirements | §19 | Explicit figures + async/queue/index requirements | Load-fixture test at a fraction of target scale (e.g. 1,000 SKUs) asserting p95 query latency budget | B (schema design gate) | Real load testing at the full stated scale (10k+ SKUs) can't happen until real catalog growth or synthetic data generation exists |
| 13 | Phase B0 review shell | §19 | Shell UI spec: presentation, evidence panel, Approve/Reject/targeted-regenerate | A UI fixture asserting free-text editing is reachable only via an explicit "correction exception" action for new-model packages | B0 | Legacy-package migration-window UI needs its own explicit end-date policy, not yet set |
| 14 | Operational instrumentation | §20 | Metric formulas, baseline policy, `correlationId` attribution, missing-data handling | A fixture period with zero events reports "insufficient data," asserted as a negative case (must not report 0% error rate) | B0 onward (capture starts immediately) | Baseline-period historical data may not exist for every metric; some will start from a forward-only baseline |
| 15 | Public editorial safeguards | §17, §18 | `EditorialAsset` entity (renamed from `Publication`, §22.2 item 1), claim-to-evidence validation, RBAC-tracked editorial approval | A fixture asserting an `EditorialAsset` cannot reach approved status with an unvalidated factual claim | Later phase (public editorial, after C/D/E prerequisites) | License/copyright rules per source type are named as a requirement but not yet enumerated per real source category |

### 22.1 Additional traceability — environment/test-data corrections (this revision)

Not part of Copilot's original fifteen; added directly by Phil after Revision 3's first push, in response to this document's own mischaracterization of `HY-LOB01-C04`'s test data. Tracked separately so the two sources of correction stay distinguishable.

| Item | Revision 3 section(s) | Concrete artifact/interface | Acceptance test | Phase | Residual risk |
|---|---|---|---|---|---|
| Test-data/environment isolation | §25 | `RecordContext` envelope on every entity/claim/recommendation/decision/package/publication/correction/metric event/promotion candidate | Fixture: a claim created with `context.environment = "test"` is absent from a production-scoped `queryApplicableClaims` call by default | B (defined alongside the claim model) | Retrofitting `RecordContext` onto every entity type is real schema surface area, not a one-line addition |
| Knowledge-promotion eligibility | §25, §18 | `context.eligibility.knowledgePromotion` gate added to the `verified → promoted` transition | Fixture: a high-confidence, fully-cited test-context claim still fails promotion on eligibility alone | B/C | Eligibility defaults (what a newly-created record's flags default to) need to be specified precisely enough that "forgot to set it" fails closed, not open |
| Database migration reproducibility | §1.1 | CI job building the full schema from an empty database using only committed migrations | CI fails if any table/column exists in a target environment without a corresponding migration | A (blocking — nothing else in Phase A starts until this passes) | This is the one item in this table that isn't just design — it's a real, currently-failing check against the live database today |
| Schema-drift detection | §1.1 | CI comparison of expected migration ledger vs. live database's applied-migration record | A deliberately introduced direct-SQL change to a test database is caught by the next CI run | A | Requires the live database to actually expose its applied-migration history queryably, which depends on adopting a real migration-ledger table first |
| `HY-LOB01-C04` test-data remediation — **executed** | §1.2 | The 5-step process (identify → classify/quarantine → determine disposition → preserve audit record → no deletion without Phil's approval) ran to completion once Phil clarified `HY-LOB01-C04` was never real and explicitly authorized deletion | Verified: `products`, `listing_packages`, `listing_package_versions`, `photo_sets`, `photo_assets`, `photo_derivatives`, `shopify_publications`, `commerce_details` all confirmed empty for this SKU by direct re-query after deletion; Shopify `productDelete` returned no user errors for `gid://shopify/Product/10220386648384`; local photo folders confirmed removed | Executed as a pre-Phase-A operational step, per this authorization — not itself a Phase A architecture deliverable | None remaining for GM Commerce's own systems. Any corresponding Skrybix-side record is out of this project's control (separate repository/credentials) and was not acted on. |

### 22.2 Additional traceability — Codex's twelve corrections

Codex's independent re-review of the corrected Revision 3 commit, returning `APPROVE AFTER DOCUMENTED CORRECTIONS`. Tracked separately from §22 (Copilot's original fifteen) and §22.1 (the same-day environment/test-data corrections), same discipline.

| # | Correction | Revision 3 section(s) | Concrete artifact/interface | Acceptance test | Phase | Residual risk |
|---|---|---|---|---|---|---|
| 1 | Split marketplace publication from public editorial content | §5, §7, §13, §18 | `MarketplacePublicationAttempt` (immutable receipt record), `EditorialAsset` (renamed from `Publication`) | Fixture: creating a `MarketplacePublicationAttempt` never requires or accepts editorial-approval fields, and vice versa | B (entities) / F (real attempts) | No second real marketplace exists yet to prove the attempt record's shape holds under a genuinely different platform |
| 2 | Variant-level `CommercePackage` offers | §7 | `VariantOffer` schema, `offers[]`, exact-item vs. representative-item mode | Fixture: a multi-variant product (Plant Pals-shaped) produces one `VariantOffer` per variant with no adapter-side reconstruction | B/F | Real multi-variant catalog data to validate the schema against doesn't exist yet in this pipeline |
| 3 | Field-level lineage | §7 | `FieldLineage`, `fieldPath`-addressable | Fixture: every field path in a completed `CommercePackage` resolves to a `FieldLineage` entry with ≥1 claim or an owner-decision reference | B/F | Backfilling lineage for any legacy-migrated field (§6, §14.1) needs its own resolution rule, not yet specified per field type |
| 4 | Repository contract completion | §9 | Evidence/claim/policy/recommendation/decision/lineage operations enumerated | Contract test: every method listed in §9 has a corresponding authorization check and audit event | B | Some operations (e.g. `evaluatePolicy`) depend on the policy engine's own design, not yet detailed beyond this interface |
| 5 | `SourceConnector` completion | §8 | `health`, cursor pagination, versioning, `resolveIdentifier`, `WritebackReceipt` | Fixture connector test: a repeated `writeback` call with the same idempotency key returns `already_applied`, never double-writes | B | Real Skrybix/SKU Generator connectors migrating onto this interface is unstarted work, same residual risk as §22's item 3 |
| 6 | `RecordContext` approval-state enum | §25 | `approvalState: not_required \| pending \| genuine \| test_exception` | Fixture: a freshly-generated, unreviewed record defaults to `pending`, never `test_exception` or `genuine` | B | Existing prose/examples elsewhere in this document referring to `ownerApproval.genuine` needed updating to the new enum — verified corrected in this pass, but a future edit could reintroduce the old shape without a schema-level lint |
| 7 | Eligibility defaults closed | §25 | DB/API default `false` for all three eligibility flags; missing/null fails closed | Fixture: a record inserted with eligibility omitted is treated identically to one with all three flags explicitly `false` | B | Requires the actual database schema to enforce `NOT NULL DEFAULT false` rather than relying on application-layer discipline alone — a schema-level requirement to carry into Phase B's migration, not yet written |
| 8 | Cross-environment access restricted | §9, §25 | `environmentFilter` removed from `queryApplicableClaims`; `requestCrossEnvironmentAccess` as the sole cross-environment path | Fixture: a production-scoped call to `queryApplicableClaims` never returns a test-context claim, with no parameter able to change that | B | The privileged authorization level required for `requestCrossEnvironmentAccess` is named but not yet mapped onto §15's specific role grants |
| 9 | Verification and promotion thresholds | §10.1 | Corroboration/authority/freshness thresholds by claim class; automated/verifier/Phil-required tiers, corrected by Codex's targeted verification to distinguish authoritative-source facts and existing-policy applications (auto-verifiable) from AI-generated recommendations (evidence-backed, validated, approved once at completed-package review) | Two fixtures: (1) a commerce-consequential claim sourced directly from its authoritative system of record, fresh and fully traced, reaches `verified` automatically with no Phil call; (2) an AI-generated price recommendation never reaches `verified` as a bare fact and instead surfaces in completed-package review with reasoning/confidence, requiring exactly one approval action, not a per-field confirmation | B/C | Thresholds are specified per claim *class*; real production claims will need to be classified against this list, which hasn't been exercised against live data yet |
| 10 | Phase B0 correction sequencing | §14.1, §19 | `LegacyCorrectionEvent`, migration into canonical `Correction` in Phase B | Fixture: a test-context `LegacyCorrectionEvent` is marked `ineligible`, never `migrated`, regardless of how cleanly it resolves | B0 (capture) / B (migration) | The resolution logic mapping a legacy field identifier to a canonical `(entityType, entityId, predicate)` isn't yet specified beyond "attempted" |
| 11 | Research-ingestion isolation strengthened | §10 | Eight-step untrusted-content pipeline; adversarial fixture requirement | Adversarial fixture suite across web/PDF/OCR/supplier/OneDrive content, each asserting pipeline behavior is unaffected by embedded instructions | C/D | No adversarial fixture corpus exists yet; this is a real, non-trivial test-authoring effort, not just a policy statement |
| 12 | Owner-effort instrumentation precision | §20 | Five decomposed time categories, start/stop event pairs, physical-vs-administrative split | Fixture: a start event with no matching stop event within timeout reports "unresolved," asserted as distinct from both a fast completion and a missing-data period | B0 onward (capture starts with B0's review shell) | Timeout thresholds per category aren't yet set; requires real usage data to calibrate rather than an arbitrary default |

## 23. Revised phase sequencing

Replaces Revision 2's Phase A–K plan. Etsy publishing and production implementation remain paused throughout — nothing below is authorized to start until Phil approves following Codex's targeted verification of the twelve corrections in this revision (Codex's independent re-review having already returned `APPROVE AFTER DOCUMENTED CORRECTIONS`).

- **Phase A** — only isolated mechanical corrections that create **no** dependency on legacy provenance or the legacy review form, narrowly scoped to schema reconciliation per this revision's correction. Concretely, per §1's disposition column: reconciling the undocumented live-database migration so the schema is reproducible from git as a baseline (not a re-execution, §1.1), standing up §1.1's migration-ledger/CI-drift-detection gate, and writing the incident record for the direct-SQL event — nothing else from the current uncommitted work qualifies as "Phase A" under this stricter definition, since most of it either touches `content_provenance` directly or modifies the legacy review form. §1.2's `HY-LOB01-C04` quarantine is a prerequisite operational step, not itself a Phase A deliverable.
- **Phase B0** — completed-package review shell (§19) and correction-event capture via `LegacyCorrectionEvent` (§14.1), built before the claim model itself exists.
- **Phase B** — canonical entities (§5, now eighteen types), claim/evidence model (§6), Repository foundation and service contract (§9, complete per Codex Correction #4), legacy migration (§1, §6), and migration of eligible `LegacyCorrectionEvent`s into canonical `Correction` records (§14.1).
- **Phase C** — OneDrive evidence library and secure ingestion (§17, §10's security controls).
- **Phase D** — image understanding (§11).
- **Phase E** — independent validation and marketplace-compliance gates (§12, plus the deterministic/evidence-grounded validator split from Revision 2 §4/§6).
- **Phase F** — core recommendation services (price, taxonomy, collections, marketplace suitability, photography, SEO, merchandising — Revision 2 §9), each producing a `SelectionTrace` (§18).
- **Phase G** — owner-editable policies and learned-rule activation (§14's confirmation flow, §15's RBAC-gated policy changes).
- **Phase H** — completed-review refinement (deepening B0's shell with everything Phases B–G actually produced).
- **Later phases** — broader proactive portfolio recommendations, Skrybix contract expansion (§16), Etsy (built on the now-proven `CommercePackage`/adapter boundary, §7), and public editorial publishing (§17, §18) — each explicitly gated on its stated prerequisites, not simply "next" by document order.

## 24. Roadmap status

`ROADMAP.md`'s milestones 2 and 3 remain built-but-not-yet-meeting-the-corrected-standard, as in Revision 2 — unchanged assessment. Two stale items this revision removes: **§11 (response to Copilot's critique) is no longer an open step** — it's complete as of this revision, per §21 above; and **Copilot's focused re-review is no longer the open step** — it returned `REQUIRES REVISION 3 AND RE-REVIEW`, was addressed, and Codex's subsequent independent re-review returned `APPROVE AFTER DOCUMENTED CORRECTIONS` on the corrected commit. The open step now is Codex's own targeted verification of the twelve corrections this revision adds in response to that review (§22.2) — confirming they are present and internally consistent. Phil authorizes the implementation sequence only after that verification lands, and even then only for the narrowly-defined schema-reconciliation Phase A (§23), not the full sequence at once.

## 25. Environment and test-data isolation

Added directly by Phil after this document's first push, in direct response to §1's own initial mischaracterization of `HY-LOB01-C04`'s test data as genuine — the clearest possible demonstration, inside this document's own history, of why this section has to be an enforced architectural layer and not a convention people are trusted to remember.

### The `RecordContext` envelope

Every entity defined in §5 (all eighteen types, including `MarketplacePublicationAttempt` and `EditorialAsset`), plus `Recommendation`, `CommercePackage`, and `Correction` specifically, plus three record types implied but not previously named — `MetricEvent` (§20's instrumentation), `KnowledgePromotionCandidate` (§18's lifecycle), and `LegacyCorrectionEvent` (§14.1) — carries this envelope:

```
RecordContext {
  environment: "development" | "test" | "staging" | "production"
  recordPurpose: "operational" | "test" | "demonstration" | "migration" |
                 "fixture"
  sourceRun: RunId
  correlationId: CorrelationId          // §9, §13
  ownerApproval: {
    // Codex Correction #6 — replaces the prior `genuine: boolean`, which
    // collapsed "nobody has reviewed this yet" and "this is a documented
    // test exception" into the same false value. A record generated by
    // the pipeline and not yet looked at by anyone must default to
    // `pending`, never be misclassified as `test_exception`.
    approvalState: "not_required" | "pending" | "genuine" | "test_exception"
    testExceptionRef: OwnerDecisionId | null   // required, populated, only
                                                 // when approvalState is
                                                 // test_exception — the
                                                 // documented, one-time
                                                 // test exception that
                                                 // authorized this record
                                                 // to exist at all (the
                                                 // GMCOM-012/015 pattern
                                                 // already used once this
                                                 // session, now formalized)
    reviewerActor: ActorRef | null      // required, populated, only when
                                                 // approvalState is genuine
    decisionTimestamp: Timestamp | null // required, populated, only when
                                                 // approvalState is genuine
    ownerDecisionRef: OwnerDecisionId | null   // required, populated, only
                                                 // when approvalState is
                                                 // genuine
  }
  eligibility: {
    // Codex Correction #7 — all three default to false at both the
    // database and API layer. A missing or null value on any of the
    // three is treated identically to false (fails closed); no code path
    // may treat an absent eligibility flag as permissive. Eligibility
    // becomes true only through an explicit, authorized transition that
    // itself emits an audit event (§9) — never a bulk default, a
    // migration side-effect, or an inferred value.
    knowledgePromotion: boolean        // default: false
    pricingAndPerformanceAnalysis: boolean   // default: false
    publication: boolean               // default: false
  }
  retention: {
    status: "active" | "quarantined" | "scheduled_for_cleanup" |
            "retained_as_fixture"
    reviewedAt: Timestamp | null
    reviewedBy: ActorRef | null
  }
}
```

**Entity-level and claim-level context are independent, not inherited automatically** — this is the precise mechanism behind "a real inventory item used in a test does not make the test's generated commerce facts real." The general principle does not depend on `HY-LOB01-C04` specifically being a real entity — Phil has since clarified `HY-LOB01-C04` was never a real plant, cutting, or inventory item at all (it and everything derived from it were synthetic test data, now removed per §1.2) — but the mechanism it illustrates still holds for a genuine future case: an entity's `RecordContext.environment` (real or test) is set independently of, and never inherited by, the `context.environment` on any `Claim`, `Recommendation`, or `CommercePackage` generated about it. A record's context is set explicitly when it's created, never derived by walking up to a parent entity — so even a wholly real `InventoryItem` never makes a test-session's generated commerce facts about it real.

### Hard invariants

Each restated with the specific mechanism that enforces it, not left as a principle to remember:

- **Test data cannot enter production knowledge, pricing, performance, policy, learning, or public editorial outputs.** Enforced at every query/promotion boundary: §9's `queryApplicableClaims` is scoped to the caller's own environment by construction, with no parameter able to widen it (Codex Correction #8); §18's promotion state machine checks `eligibility.knowledgePromotion`; §20's metrics only aggregate `eligibility.pricingAndPerformanceAnalysis: true` records; §18's `EditorialAsset` approval checks `eligibility.publication`.
- **A real inventory item used in a test does not make the test's generated commerce facts real.** See above — independent context per record, not inherited.
- **Test approval exceptions cannot become owner decisions or durable rules.** §14's rule-activation flow excludes any correction whose originating claim/recommendation has `ownerApproval.approvalState` other than `genuine` from ever being proposed as a durable rule; `test_exception`/`pending` records are structurally ineligible, and `testExceptionRef` makes a documented exception itself visible and auditable rather than indistinguishable from a real approval. A record still `pending` review is never treated as either genuine or a test exception — it is simply not yet eligible for anything that requires `genuine`.
- **Test publications and Shopify drafts must be visibly labeled and excluded from production metrics.** A `MarketplaceListing`, `MarketplacePublicationAttempt`, or `EditorialAsset` with `context.environment: test` carries a mandatory visible label in its adapter payload (§7) — this is a publish-time requirement, not just a database flag — and is excluded from §20's metrics via `eligibility.pricingAndPerformanceAnalysis: false`.
- **Cross-environment references are prohibited unless performed through an explicit, audited promotion process.** *(Scope tightened by Codex Correction #8.)* A production claim can never cite a test-context claim as its evidence, and — per §9's `queryApplicableClaims` — a production caller structurally cannot receive test/staging/development claims through the ordinary query path at all; there is no parameter that widens it. The only sanctioned path is §9's `requestCrossEnvironmentAccess`, scoped to exactly three purposes (audited promotion, authorized diagnostics, fixture evaluation) and required to carry actor, reason, source environment, destination environment, correlation ID, and its own immutable audit event. **Diagnostic reads obtained this way can never feed a production recommendation run or a `knowledgePromotion` transition** — enforced at the command layer (§9), not left as a caller convention.
- **Production data cannot be copied into lower environments without appropriate privacy and access controls.** Gated by §15's RBAC at the Repository command layer — a production-to-test copy is itself a privileged, logged action.
- **AI evaluation fixtures must remain distinguishable from business records.** `recordPurpose: "fixture"` is its own value, distinct from `"test"` — a fixture used to evaluate the pipeline's own quality is not even a test of a real product, and must never be queryable alongside either test or production business records.
- **Knowledge promotion must verify both evidence quality and production eligibility.** §18's `under_review → verified` transition now requires both no unresolved contradiction (unchanged) *and* `context.eligibility.knowledgePromotion === true` (new) — evidence can be perfectly verified and still correctly blocked from promotion if its context says it shouldn't be.

### Environment-aware queries, applied to every contract this document defines

§8's `SourceConnector` and §9's `IntelligenceRepositoryV1` interfaces are edited directly (above) to carry environment scoping. The same requirement applies to every dedicated Recommendation, Policy, Learning, Metric, and Editorial service surface a later phase defines: **no query interface in this architecture may return records without either an explicit environment scope by construction or a use of §9's dedicated `requestCrossEnvironmentAccess` operation (Codex Correction #8) — never an ordinary optional parameter that widens an otherwise-scoped query.** Where dedicated interfaces don't exist yet (Recommendation, `MarketplacePublicationAttempt`, and `EditorialAsset` currently live as entity types plus operations on the shared Repository contract, not standalone services), this requirement is binding on whichever contract does serve those queries today — meaning `queryApplicableClaims` and any future recommendation-specific query built on top of it inherit this scoping by construction, not by separate opt-in.
