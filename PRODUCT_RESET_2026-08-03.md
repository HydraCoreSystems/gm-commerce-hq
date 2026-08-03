# GM Commerce — Product & Architecture Reset

**Status: awaiting Phil's review. Implementation is paused pending sign-off — see "Migration plan" for what is and isn't safe to continue in the meantime.**

## Why this document exists

Inspecting HY-LOB01-C04's real Shopify draft (GMCOM-012) surfaced defects — placeholder text, $0 price, no weight, duplicate alt text, no care instructions. The first response (GMCOM-015) built a Commerce Readiness Gate that correctly refuses to publish incomplete content. But partway through rebuilding HY-LOB01-C04 against that gate, every blocker the gate raised turned into a question back to Phil: what's the price, what's the weight, write the care instructions, pick a Shopify collection ID, fix the duplicated sales copy.

That's the actual defect. A gate that fails closed is correct. A system that responds to its own gate by handing the work to the owner is not. Phil's correction: GM Commerce should function as an autonomous commerce director — Phil directs it and approves its output; he should never be the fallback worker inside his own software.

**Governing completion test**, verbatim from Phil, referenced throughout this document:

> Does this let Phil accomplish work that would otherwise require additional employees, without making Phil the missing employee inside the software? If routine work still falls back to Phil or Crystal when it could reasonably be retrieved, researched, inferred, derived, remembered, recommended, generated, or verified by the system, the workflow is not complete — regardless of whether the technical pipeline functions.

This is not new scope. `VISION.md` already says GM Commerce exists to give Gathering Moss "the leverage of a much larger company without adding a large payroll" and to work "with minimal human intervention while keeping Phil and Crystal in control of important business decisions." HY-LOB01-C04 is the first real product to actually hit that standard, and it failed it. This document is the design response.

---

## 1. Product vision

GM Commerce is not a listing form, a Skrybix-to-Shopify pipe, or a CRUD app with an AI-generated-copy feature bolted on. It is meant to operate as a **commerce director**: a function that would otherwise be a merchandiser, copywriter, product researcher, pricing analyst, and marketplace operations person, all reporting to Phil.

Concretely, that means the system:

- **Assembles** everything knowable about a product from every system that already knows it (Skrybix, prior listings, store policy, GM Commerce's own history) before ever treating something as unknown.
- **Researches** what it doesn't yet know but reasonably could (general care guidance for a species, a defensible price range from comparable past sales) rather than leaving a blank field.
- **Verifies** its own output adversarially — checks for the exact failure classes that have actually happened (placeholder language, duplicate copy, unsupported claims, inconsistent pricing) — before anything reaches a human.
- **Recommends**, proactively, not just reactively: pricing suggestions, restock/bundle ideas, aging-listing nudges, anomalies worth a second look.
- **Remembers**: corrections Phil makes become durable, owner-controlled rules, not repeated corrections.
- **Defers to Phil only for what only Phil can decide**: final approval before anything goes public, true business strategy (what to carry, pricing philosophy, brand voice changes), and any fact that is genuinely unknowable to anyone but a human who has physically handled the item.

The owner-preserved decisions already listed in `VISION.md` (what to carry, final pricing strategy, discontinuing products, new product lines, brand-voice/policy changes) are unchanged and are the actual boundary of "owner-only." Everything else is fair game for the system to own.

## 2. Current architecture gap analysis

What's built and sound: guided intake (GMCOM-006), the AI Provider abstraction (GMCOM-007), the multi-stage Listing Quality Engine (GMCOM-008), atomic/versioned/validated generation (GMCOM-009), the photo pipeline (GMCOM-011), the real Shopify draft publisher (GMCOM-012), and the Commerce Readiness Gate (GMCOM-015). These are real, tested, and mostly reusable — see §10.

Where the gap actually is — every one of these is a place the system currently (or as originally re-planned mid-GMCOM-015) treats "I don't have this fact" as "ask Phil," rather than "can I derive, research, recall, or default this first":

| Area | Current/as-replanned behavior | Gap |
|---|---|---|
| Price | Free-text human entry, re-entered risk on every regen (real bug: AI schema always returned `price: null` and finalize silently overwrote a human-entered value on every regenerate) | No comparable-sales research; no persistent price history to suggest from |
| Weight | Free-text human entry every time | No template defaults for standardized non-plant SKUs; no Skrybix-side capture for plants |
| Inventory policy | Free-text human entry every time | No store-wide default |
| Shopify category / collections | Human types a Shopify GID by hand | No deterministic category→collection mapping; asks the owner to know Shopify's internal IDs, which is absurd |
| Shipping expectations | Human writes freeform text per listing | Not reusable store policy text |
| Exact-item disclosure | Human writes freeform text per listing | Fully derivable from source_system (Skrybix = exact item; SKU Generator = representative) |
| Care instructions | AI told to leave it null unless "given facts" include it — which they never do | No permission/mechanism for the AI to actually research general species care |
| Sales-summary near-duplicate of description | Detected by the gate, then left for a human to notice and rewrite | No automatic repair; a real, known failure mode with no fix loop |
| Alt-text duplicates | Prompted against, not guaranteed | No automatic duplicate detection + repair after generation |
| Collection "confirmation" | A separate human confirmation step, modeled as an approval gate | Confirming a deterministic mapping is not a decision; it's friction with no judgment behind it |
| Cross-listing consistency | None | No check that a new listing's price/category/tone is consistent with how similar past products were handled |
| Knowledge persistence | None — every generation starts from zero except the row it's overwriting | No durable place for "what Gathering Moss has already decided" to live and be reused |
| Recommendations | None — the system is purely reactive to what a human initiates | No proactive surface at all |
| Learning from correction | None — Phil's edits vanish into that one listing, teach the system nothing | Repeated corrections stay repeated forever |

This session's in-flight work (before the pause) was already fixing the first eight rows — see §10 for exactly what to keep.

## 3. Manual-handoff inventory

Every point where a human is currently (or was about to be) required, classified as **(A) genuinely owner-only** or **(B) currently manual but should be system-derived**:

**(A) Genuinely owner-only — keep as human actions:**
- Final approve/reject of a completed package before it can be marked reviewed or published.
- The actual choice to publish to a given marketplace (Shopify vs. Etsy vs. both) for a given SKU.
- Business policy content itself (what the shipping policy text *says*, what the default inventory policy *is*) — Phil sets the policy once; the system then applies it everywhere without being asked again.
- Price *as a business decision* — but see §9: the system should propose a number, Phil should adjust or confirm, not originate it from a blank field.
- What to carry, discontinue, or launch (unchanged from `VISION.md`).
- A physical, per-item fact that truly exists nowhere yet — today, most concretely: a specific plant's rooting status/condition at intake time, until Skrybix captures it (§8).

**(B) Currently manual, should become system-derived (in-flight or planned):**
- Weight (template default for standardized products; still §8's Skrybix gap for unique plants).
- Inventory policy (store default).
- Shopify category and collection assignment (deterministic mapping — no Shopify ID should ever be manually typed).
- Shipping expectations text (reusable policy).
- Exact-item disclosure (derived from source system).
- Care instructions (AI research).
- Sales-summary/alt-text duplicate repair (automatic).
- A *starting* price suggestion (comparable-sales research; final number still A).

Nothing on list (B) should ever again surface to Phil as a blank field or a blocking gate finding under normal operation — only as something to confirm, override, or be notified didn't have a confident answer.

## 4. Autonomous intelligence architecture

Five layers, replacing the current flat "gather facts → one or two AI calls → persist" pipeline:

**Fact layer — the Product Truth record.** One reconciled, versioned record per SKU, assembled from every source GM Commerce has: Skrybix's export, `commerce_details`, prior listing versions, and confirmed AI research. This is what today's `listing_packages.source_facts` was reaching for but doesn't fully deliver — it's currently AI-input scaffolding, not a queryable, owner-visible truth record.

**Research layer — a distinct AI job from copywriting.** Two different kinds of AI calls, currently conflated into one "write me a listing" prompt:
- *Copywriting*: turn known facts into customer-facing text. Must never invent facts.
- *Research*: go find out something the system doesn't know yet but reasonably could — general species care guidance, a defensible price range from comparable history. Writes to the Product Truth record with a confidence and a source description, never straight into customer copy. (Care-instructions research was mid-implementation this session under exactly this framing — see §10.)

**Verification layer — adversarial, not self-reported.** Today's `quality_summary` is the AI grading its own homework. Add explicit, mostly-deterministic checks that run on every generation, each with a defined repair-or-fail-closed behavior (the pattern already proven this session for sales-summary/alt-text duplication): placeholder/internal-language detection (built), near-duplicate detection (built, needs the repair loop finished), price-sanity vs. comparable history (new), category/tag consistency vs. similar past products (new).

**Orchestration layer — extends GMCOM-009's proven lock, doesn't replace it.** Reserve → apply store defaults and Product Truth lookups → AI draft → AI research (care, comparables) → AI self-review/revise → automatic defect repair → verification pass → Commerce Readiness Gate → present completed package. The atomic reserve/finalize/release mechanism stays exactly as built; it's the right foundation for a longer pipeline, not a reason to avoid one.

**Recommendation layer — genuinely new, not a repackaging of the gate.** The system surfaces things nobody asked it to check: "this listing has been in review 6 days," "this Hoya's suggested price is 20% under your last 3 Hoya sales," "these two in-review SKUs look similar enough to consider bundling." Reactive form-filling and proactive recommendation are different capabilities; today there is only the former.

## 5. Persistent Gathering Moss knowledge design

A durable knowledge store, distinct from any single listing, is the actual prerequisite for most of §4. Today's in-flight `lib/commerce-readiness/defaults.ts` (store policy text, weight templates, category→collection mapping as TypeScript constants) is the right *content* but the wrong *place* — it's Claude-editable, not Phil-editable, which fails the "durable, owner-controlled" requirement outright.

Target shape (owner-editable via a real UI, not just code, in a later migration phase — see §10):

- **Store policies**: shipping text, default inventory policy, disclosure templates. Versioned; Phil edits directly.
- **Category knowledge**: care-guidance research, cached per genus/species rather than re-researched (and re-paid-for) on every single cutting listing. Phil can correct a cached entry once; the correction sticks for every future listing of that species.
- **Pricing history**: a durable ledger of what comparable SKUs actually sold for, feeding price *suggestions* (never automatic price-setting).
- **Brand voice corrections**: durable, reusable adjustments to tone/wording patterns Phil repeatedly makes, instead of the same phrasing correction happening on every listing forever.
- **Taxonomy/collection mapping**: same data as today's `defaults.ts`, moved to durable owner-editable config once the UI exists.

## 6. Research, verification, provenance, and confidence model

Every fact in the Product Truth record carries: **value, source category** (one of the six below), **confidence**, **timestamp**, and, for AI research, **what kind of knowledge it drew on** — general horticultural/species knowledge, comparable internal sales history, etc. This is deliberately *not* fabricated citation text: the current AI provider (OpenAI Responses API, no web-search tool wired in) has no live browsing grounding, and inventing plausible-looking source URLs would be a worse failure than honestly saying "general trained knowledge, medium confidence." If Phil wants real live-web-grounded research later, that's a provider capability upgrade (wiring a web-search tool into `openai-provider.ts`), tracked as a distinct future enhancement, not assumed here.

The six source categories (from Phil's original message, now the canonical taxonomy):

1. Skrybix inventory/cutting data
2. Product/SKU commerce data (GM Commerce's own stored facts)
3. Gathering Moss store/category defaults (policy, deterministic)
4. Deterministic derived rules (e.g., exact-item disclosure from source system)
5. AI research (confidence-scored, source-typed, never fabricated citations)
6. Genuine per-item human exception

Verification checks are explicit and each has a defined outcome — repair automatically, or fail closed with a specific reason — never a silent pass-through and never a pass-through to a human without first attempting repair.

Confidence surfaces to Phil at review time, not hidden: a completed package shows what's fact, what's policy, what's researched (and how confidently), not a uniform wall of text that all reads as equally certain.

## 7. Owner-authority and durable-learning model

Unchanged, hard rule: Phil or Crystal approves before anything reaches a real marketplace. Nothing in this reset touches that.

What's new is *how* the system reduces repeat work without ever silently taking authority it hasn't been given:

- Every durable default/policy/mapping is owner-editable (Phase C UI, §10) — today's TypeScript constants are an interim shim, not the destination.
- Learning is proposed, not automatic: if the system notices a repeated correction pattern (e.g., Phil consistently adjusts a suggested Hoya price down ~15%), it *proposes* updating the default/rule for Phil to confirm — it does not silently rewrite its own defaults.
- Full audit trail: every autonomous value traceable to its source category and, where it was owner-influenced, to the specific correction/approval that produced it.

## 8. Skrybix data and workflow gap analysis

Read directly from `lib/sources/skrybix.ts`: the entire commerce-export contract (`SkrybixCommerceRecord`) exposes only identity/state — `sku`, `displayName`, `parentSourceRecordId`, `plantRecordType`, `state`, `selectionState`, timestamps. **No rooting status, condition, or weight exists in the handoff at all.** This is the real root cause behind "why does GM Commerce keep asking Phil for facts about a plant Skrybix already tracked" — it doesn't, yet.

These are facts someone is already looking at the physical plant to determine, in Skrybix, at cutting-creation time — capturing them there is strictly cheaper than re-asking in GM Commerce later, and keeps Skrybix authoritative for plant facts (unchanged from the existing "Skrybix remains separate and authoritative for plants" decision).

Recommended smallest-appropriate addition to Skrybix's own commerce-export contract (a cross-repo proposal, not something GM Commerce can implement itself): optional `rootingStatus` and `conditionNotes` fields at cutting-record creation, exposed through the same `/api/commerce/v1/plants` endpoint GM Commerce already consumes. A rough size-class weight field is a reasonable stretch addition but lower priority than the two content fields.

Price is correctly **not** a Skrybix fact — Skrybix is plant identity, not commerce. It stays GM-Commerce-owned, sourced from comparable-sales research (§4) with Phil's final confirmation, never from Skrybix.

Until Skrybix implements this, GM Commerce's `/commerce/[sku]` remains the interim capture point for these two fields — explicitly labeled as covering a Skrybix gap, not a permanent design choice, and this section should be handed to whoever owns Skrybix's backlog (Phil directly, or a GitHub Issue against `skrybix-webapp`) rather than staying implicit.

## 9. Completed-package review design

`/review` should stop being an edit form and become a **completed package review**:

- Present the assembled listing the way a customer would see it — title, photos in order, price, description — not a stack of blank-or-prefilled inputs.
- Show a compact, expandable provenance/confidence summary: what's fact, what's policy-derived, what's AI-researched and how confidently.
- Surface the system's own verification results (extended `quality_summary`) — what it checked, what it found, what it fixed automatically.
- Primary actions are **Approve**, **Reject with reason**, or a **targeted regenerate** ("redo care details," "resuggest price") — not raw free-text editing as the default path. Direct editing remains available for the genuine exception case, but stops being the primary review motion.

This is the literal implementation of Phil's own definition: "inspect the completed listing, correct exceptional inaccuracies if necessary, approve or reject."

## 10. Migration plan

Nothing built so far is wasted — most of it is exactly the right foundation, just needs to be finished under this framing instead of as isolated point-fixes.

**Keep as-is, unaffected by this reset:**
GMCOM-009's atomic reserve/finalize/release lock, the provider-neutral AI abstraction, the real Shopify GraphQL client, the photo pipeline, the Commerce Readiness Gate's fail-closed mechanism (its *content* changes as fields move from "ask" to "derive," its mechanism doesn't).

**Phase A — finish what was already mid-flight this session** (these are real prerequisites for the reset, not a competing side effort):
price ownership moved to `commerce_details` (closes the real regeneration data-loss bug), deterministic store defaults for inventory policy/shipping text/exact-item disclosure/category/collections, AI-researched care instructions with calibrated-confidence language, automatic sales-summary and alt-text duplicate repair. Safe to finish and land under this document's framing — they are literally rows 1–8 of §2's gap table.

**Phase B** — formalize the Product Truth record and the six-category provenance model (extends the `content_provenance` column already added this session from "which AI call wrote this field" to the full source/confidence/timestamp model in §6).

**Phase C** — move durable defaults/policies/mappings from TypeScript constants to an owner-editable, DB-backed config with a minimal settings UI.

**Phase D** — research/verification layer extension: comparable-pricing research, adversarial price/category-consistency checks.

**Phase E** — completed-package review UI redesign (§9).

**Phase F** — recommendation surface (§4's recommendation layer) — additive, sequenced last since nothing else depends on it.

**Phase G** — deliver §8's Skrybix gap proposal to Skrybix's own backlog — cross-repo coordination, not GM Commerce code.

Etsy publishing stays paused, per Phil's explicit instruction, until Phase A–C land and HY-LOB01-C04 passes the completion test end-to-end as the proof case.

## 11. Autonomy and intelligence test strategy

- Operationalize the governing completion test as an actual repeatable check, not a vibe: for a sampled real product run through the full pipeline, count how many fields/actions required Phil/Crystal input beyond genuine (A)-category facts and final approval; a milestone isn't done if that count is nonzero for anything on the (B) list in §3.
- New regression-test category: a fixture-driven "manual-handoff regression test" asserting a real product profile, run through the full pipeline with zero manual entry beyond genuinely owner-only facts, produces a Commerce-Readiness-Gate-passing package. This puts the completion test in CI, not just in one-off human review.
- Keep and extend the existing deterministic test suite (102 tests as of this pause) for the gate, defaults, and repair loops — this layer is already strong and doesn't need reinvention, only extension as new checks land.
- Adversarial-content regression fixtures drawn from HY-LOB01-C04's actual real defects (internal-language leak, near-duplicate summary, duplicate alt text) — these are real, already-observed failure modes, the strongest possible regression cases.

## 12. Revised roadmap and implementation sequence

`ROADMAP.md`'s milestones 2 and 3 (SKU-to-listing-draft, Shopify draft publishing) are already built, ahead of the document's own sequencing — but built to the old "ask a human" standard this reset corrects. Rather than insert a new milestone number, the honest framing is: **Milestones 2 and 3 need the Phase A–E retrofit in §10 applied to them before they're actually done**, and Milestone 4 (Etsy) and Milestone 5 (sale/inventory coordination) should be built on the reset architecture from the start rather than repeating the same gap and needing their own future reset.

Immediate next steps, pending Phil's review of this document:
1. Finish Phase A (already substantially coded — price ownership, deterministic defaults, care research, duplicate repair).
2. Formalize Phase B's provenance model.
3. Resume HY-LOB01-C04's real rebuild end-to-end through the corrected pipeline as the proof case for the completion test (§11).
4. Only then resume Etsy planning — unchanged from Phil's explicit instruction.

**Assumptions this reset discards:**
- "Human review" = filling out or editing a form. It means inspecting a finished package and deciding.
- "The gate found a blocker" = "ask Phil." It means "did the system try every non-owner source first."
- Store policy as a TypeScript constant is the destination. It's an interim shim on the way to owner-editable durable config.
- One AI call handles both research and copywriting. They're different jobs that happen to share infrastructure.
- Etsy is simply "next" per `ROADMAP.md`'s literal order. Paused until the reset's completion test passes for Shopify.
