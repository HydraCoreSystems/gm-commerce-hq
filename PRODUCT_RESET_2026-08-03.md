# GM Commerce — Product & Architecture Reset

**Status: Revision 2, awaiting Phil's review, then Copilot's independent re-review before implementation resumes.**

## Revision 2 changelog (2026-08-03)

Revision 1 was directionally approved but rejected as an implementation plan. This revision incorporates eleven specific corrections Phil gave after reading Revision 1 alongside Copilot's independent GMCOM-016 audit, plus a follow-up expansion of the knowledge-repository design to include OneDrive as the durable evidence library and a broader "trusted source of plant knowledge" company objective. In order:

1. Verified external research moves from a future enhancement to foundational architecture, with live retrieval, source ranking, citations, evidence spans, freshness, conflict detection, and rejected-source reasoning designed in from the start (§4, §6, §12).
2. The generation pipeline's order is corrected: research and reconciliation happen *before* copywriting, not after (§4).
3. "Product Truth" is expanded from a flat fact record into a claim-level evidence and decision model (§5).
4. Real image understanding — actual visual inspection, not deterministic signal-only photo QA — is added as a core capability (§4).
5. The Skrybix gap analysis is expanded well beyond rooting status/condition (§8).
6. Core commerce recommendations (price, taxonomy, collections, marketplace suitability, photography needs, SEO, merchandising) move into the standard completed-package workflow; only broader proactive portfolio recommendations stay deferred (§9).
7. The durable-learning model is refined with explicit rules for auto-recording, scope inference, local-vs-generalized reuse, and when owner confirmation is actually required (§7).
8. Copilot's evidence artifacts and autonomy metrics are adopted directly (§13).
9. Independent validation is architecturally separated from generation — a second pass in the same context is not verification (§4, §6).
10. This revision responds directly to Copilot's ten-question critique — **blocked**, see §11, pending access to the actual document.
11. A first-class Gathering Moss Intelligence Repository is added, including — per Phil's same-session follow-up — a two-layer design with OneDrive as the durable original-evidence library underneath a structured claims/citations repository, an ingestion pipeline, anti-dumping-ground governance, a knowledge lifecycle, explicit protections against bad learning, and a publication/editorial pipeline supporting Gathering Moss's broader objective of becoming a trusted public source of plant knowledge, not just a seller using AI-generated copy (§10).

Phase A's "AI-researched care instructions" is also corrected in the migration plan (§12): relying on pretrained model knowledge alone, even with calibrated confidence language, is not the completed capability. It's a legitimate interim step, explicitly labeled as incomplete until live, verifiable research is wired in.

---

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

- **Assembles** everything knowable about a product from every system that already knows it (Skrybix, prior listings, store policy, GM Commerce's own history, the Intelligence Repository — §10) before ever treating something as unknown.
- **Researches** what it doesn't yet know but reasonably could, with real evidence and citations, not just recalled trained knowledge (§4, §6) — rather than leaving a blank field or asserting an uncited claim.
- **Sees**: inspects actual photo content, not just deterministic technical signals (§4).
- **Verifies** independently — a genuinely separate check, not the same model re-reading its own output — for the exact failure classes that have actually happened (placeholder language, duplicate copy, unsupported claims, inconsistent pricing) before anything reaches a human.
- **Recommends**, as a standard part of every completed package, not a bolted-on extra: price, category/collections, marketplace fit, photography gaps, SEO angle, merchandising ideas (§9).
- **Remembers**: corrections Phil makes become durable, owner-controlled, appropriately-scoped rules — never repeated corrections, never silently over-generalized (§7).
- **Defers to Phil only for what only Phil can decide**: final approval before anything goes public, true business strategy (what to carry, pricing philosophy, brand voice changes), and any fact that is genuinely unknowable to anyone but a human who has physically handled the item.

A second, connected objective, added in this revision at Phil's direction: **Gathering Moss should become a leading, trusted source of plant knowledge — not merely a seller that happens to use AI-generated content.** The same verified knowledge foundation built to support listings (§10, §11) is meant to eventually support care guides, species/cultivar profiles, educational articles, customer support, and other public-facing content that Gathering Moss can stand behind and be cited for. This is a *later*, downstream use of the same repository — see §10's publication/editorial layer and §12's migration sequencing — not a reason to slow down the commerce-facing work it's built on first.

The owner-preserved decisions already listed in `VISION.md` (what to carry, final pricing strategy, discontinuing products, new product lines, brand-voice/policy changes) are unchanged and are the actual boundary of "owner-only." Everything else is fair game for the system to own.

## 2. Current architecture gap analysis

What's built and sound: guided intake (GMCOM-006), the AI Provider abstraction (GMCOM-007), the multi-stage Listing Quality Engine (GMCOM-008), atomic/versioned/validated generation (GMCOM-009), the photo pipeline (GMCOM-011), the real Shopify draft publisher (GMCOM-012), and the Commerce Readiness Gate (GMCOM-015). These are real, tested, and mostly reusable — see §12.

Where the gap actually is:

| Area | Current/as-replanned behavior | Gap |
|---|---|---|
| Price | Free-text human entry, re-entered risk on every regen (real bug: AI schema always returned `price: null` and finalize silently overwrote a human-entered value on every regenerate) | No comparable-sales research; no persistent price history to suggest from |
| Weight | Free-text human entry every time | No template defaults for standardized non-plant SKUs; no Skrybix-side capture for plants |
| Inventory policy | Free-text human entry every time | No store-wide default |
| Shopify category / collections | Human types a Shopify GID by hand | No deterministic category→collection mapping; asks the owner to know Shopify's internal IDs |
| Shipping expectations | Human writes freeform text per listing | Not reusable store policy text |
| Exact-item disclosure | Human writes freeform text per listing | Fully derivable from source_system (Skrybix = exact item; SKU Generator = representative) |
| Care instructions | AI told to leave it null unless "given facts" include it — which they never do | No permission/mechanism for the AI to research general species care, and no *verified, cited* research even where attempted (Revision 1's mistake — see §12) |
| Sales-summary near-duplicate of description | Detected by the gate, then left for a human to notice and rewrite | No automatic repair; a real, known failure mode with no fix loop |
| Alt-text duplicates | Prompted against, not guaranteed | No automatic duplicate detection + repair after generation |
| Collection "confirmation" | A separate human confirmation step, modeled as an approval gate | Confirming a deterministic mapping is not a decision; it's friction with no judgment behind it |
| Photo quality review | Deterministic signals only — blur score, exposure mean, perceptual-hash near-duplicate detection | No actual visual inspection at all: no identity check, no hero-quality judgment, no angle-coverage check, no visible-condition read, no crop/background/duplicate-view judgment, no image-grounded alt text |
| Self-review / verification | The same model, same call context, immediately re-reading its own draft | Not independent verification — shares the exact blind spots that produced the draft; no deterministic-or-evidence-grounded validator layer |
| Cross-listing consistency | None | No check that a new listing's price/category/tone is consistent with how similar past products were handled |
| Knowledge persistence | None — every generation starts from zero except the row it's overwriting | No durable place for "what Gathering Moss has already decided or already verified" to live, be cited, and be reused |
| Original evidence (research papers, supplier docs, photos, growing notes) | Doesn't exist as a system concept at all | No durable, organized evidence library; nothing for a claim to cite back to |
| Recommendations | None — the system is purely reactive to what a human initiates | No proactive surface, and no *core* per-listing recommendation either |
| Learning from correction | None — Phil's edits vanish into that one listing, teach the system nothing | Repeated corrections stay repeated forever |
| Public knowledge / content authority | Not a system concept | No path from verified internal knowledge to any public-facing educational asset |

This session's in-flight work (before the pause) was already fixing several of these rows, but under an incomplete model in places (most importantly: "research" without verification/citation, and orchestration ordering that ran copywriting before research was reconciled) — see §12 for exactly what to keep versus correct.

## 3. Manual-handoff inventory

Every point where a human is currently (or was about to be) required, classified as **(A) genuinely owner-only** or **(B) currently manual but should be system-derived**:

**(A) Genuinely owner-only — keep as human actions:**
- Final approve/reject of a completed package before it can be marked reviewed or published.
- The actual choice to publish to a given marketplace (Shopify vs. Etsy vs. both) for a given SKU.
- Business policy content itself (what the shipping policy text *says*, what the default inventory policy *is*) — Phil sets the policy once; the system then applies it everywhere without being asked again.
- Price *as a final business decision* — the system proposes a number with comparable-sales reasoning; Phil adjusts or confirms, never originates from a blank field (§9).
- What to carry, discontinue, or launch (unchanged from `VISION.md`).
- A physical, per-item fact that truly exists nowhere yet — today, most concretely: a specific plant's rooting status/condition at intake time, until Skrybix captures it (§8).
- Establishing a new durable rule from an observed correction pattern (§7) — proposed by the system, confirmed once by Phil, then applied automatically.
- Approving public-facing educational content before publication (§10's editorial layer) — same "approve, don't originate" posture as commerce listings.

**(B) Currently manual, should become system-derived:**
- Weight (template default for standardized products; still §8's Skrybix gap for unique plants).
- Inventory policy (store default).
- Shopify category and collection assignment (deterministic mapping — no Shopify ID should ever be manually typed).
- Shipping expectations text (reusable policy).
- Exact-item disclosure (derived from source system).
- Care instructions (verified, cited AI research — not trained-knowledge recall alone).
- Sales-summary/alt-text duplicate repair (automatic).
- A *starting* price suggestion (comparable-sales research; final number still A).
- Photo quality/completeness assessment (real image understanding — §4).
- Taxonomy, marketplace-suitability, and merchandising notes on a completed package (§9) — presented as recommendations, not blank fields.
- Reapplying an already-confirmed durable rule to a new matching case (§7) — no re-confirmation needed once the rule itself is approved.

Nothing on list (B) should ever again surface to Phil as a blank field or a blocking gate finding under normal operation — only as something to confirm, override, or be notified didn't have a confident answer.

## 4. Autonomous intelligence architecture

**Corrected orchestration order** (Phil's explicit correction — research and reconciliation happen before writing, not after):

```
Gather facts → identify gaps → research → verify/reconcile → recommend → write
  → independently validate → repair → readiness gate → completed-package review
```

This replaces both today's flat "gather facts → one or two AI calls → persist" pipeline and Revision 1's still-wrong "draft → research → self-revise" ordering. What each stage actually does:

1. **Gather facts** — pull everything already known: Skrybix export, `commerce_details`, prior listing versions, applicable Intelligence Repository knowledge (§10) for this SKU's genus/category.
2. **Identify gaps** — explicitly enumerate what's missing or stale relative to what a complete package needs (care guidance, a defensible price, taxonomy, photo completeness) *before* any generation happens. This step doesn't exist at all today; today "missing" is only discovered after the fact, by the readiness gate.
3. **Research** — for each identified gap, retrieve real evidence (§6): live retrieval where the provider supports it, source ranking, and citation capture. Writes claims to the evidence model (§5), not directly to customer copy.
4. **Verify/reconcile** — cross-check new research against existing claims and each other: detect contradictions, establish precedence (owner-confirmed fact always outranks AI research — §7), flag anything that couldn't be reconciled rather than silently picking one.
5. **Recommend** — from the now-reconciled facts, produce the core per-listing recommendations: price, taxonomy/collection assignment, marketplace suitability, photography gaps, SEO angle, merchandising notes (§9). This happens *before* copy is written, because the copy should reflect the recommendation, not the other way around.
6. **Write** — copywriting, consuming only the reconciled facts and recommendations already produced. Must never invent a fact; this is the one stage that most resembles today's single AI call, now with a much better-supplied input.
7. **Independently validate** — a genuinely separate check (§6), not the same model re-reading its own output: deterministic checks wherever possible (denylist, near-duplicate, alt-text uniqueness, length bounds, required-field presence), plus evidence-grounded validator calls for judgment-requiring claims, given only the claim and its evidence, explicitly asked to find fault.
8. **Repair** — targeted, automatic fixes for anything the validator catches (the pattern already built this session for sales-summary/alt-text duplication), escalating to a hard failure — never a silent pass-through to review — if repair genuinely can't resolve it.
9. **Readiness gate** — the existing Commerce Readiness Gate, now mostly a structural safety net (defense-in-depth) rather than the primary place blockers are discovered, since stages 2–8 should have already resolved almost everything it checks.
10. **Completed-package review** — human presentation (§9).

The atomic reserve/finalize/release lock (GMCOM-009) wraps the whole sequence exactly as it does today; nothing about this ordering changes that guarantee.

**Layers, replacing the old flat pipeline:**

**Fact/evidence layer.** The claim-level evidence and decision model — see §5 for the full design. This is the actual foundation everything else reads from and writes to.

**Research layer — foundational, not a future enhancement.** Distinct from copywriting: its job is to go find out something the system doesn't know yet but reasonably could, and to do so *verifiably*. Includes, from the start (§6, §12): live retrieval (not trained-knowledge recall alone), source ranking (preferring higher-authority sources), citation capture, evidence-span extraction (the specific passage a claim is grounded in, not just a source name), freshness tracking, conflict detection across sources, and explicit rejected-source reasoning (why a source was *not* used, so that reasoning is inspectable later, not silently discarded).

**Image understanding layer — new.** Real visual inspection of actual photo content, not just deterministic technical signals. Covers: identity consistency (does this photo actually show the claimed product — catches mismatched photos), hero selection (informed by actual visual quality, not just `order_index`), angle coverage (are there enough distinct views for this product type), visible condition (a genuine input to condition claims, not just typed notes), crop suitability, background distractions, duplicate-view detection (visually similar shots the perceptual-hash check can miss — two photos of the same angle from a slightly different position), missing-shot detection ("no photo shows the root system for a rooted-cutting claim"), and truly image-grounded alt text. Architecturally: the existing provider-neutral AI abstraction already supports this without a new integration pattern — it needs a genuinely multimodal request type (image bytes/URLs in the call, not text-only facts), which today's alt-text and listing-generation calls deliberately don't use (see the existing "text-only, not vision" note in `lib/photos/alt-text.ts` — that Version 1 boundary is exactly what this layer removes).

**Independent validation layer — architecturally separate from generation.** Today's "self review + revision" (GMCOM-008) is the same model, in the same call, immediately extending its own reasoning — not independent verification, and it shares the exact blind spots that produced the draft. Real independent validation splits into: deterministic checks (already built and genuinely independent — regex/word-overlap/length/presence checks don't share the generator's blind spots by construction) and evidence-grounded validator calls, given *only* the claim plus its cited evidence — not the generation prompt or history — and explicitly instructed to try to refute the claim, not polish it. See §6.

**Recommendation layer — core, not a future add-on (Revision 1's mistake).** Price, taxonomy/collection assignment, marketplace suitability, photography completeness, SEO strategy, and merchandising notes are produced for *every* completed package as part of the standard pipeline (stage 5 above), not a "nice to have" surfaced later. Only *broader, cross-SKU, proactive portfolio* recommendations — "these three listings have been in review 6+ days," "your Hoya collection's average price has drifted," bundling suggestions across unrelated SKUs — stay deferred to a later phase (§12), since they depend on enough listings existing to compare across, not on anything architecturally harder.

**Learning layer.** Durable, owner-controlled, precisely scoped — see §7 for the full refined model.

## 5. Claim-level evidence & decision model

"Product Truth" as originally scoped in Revision 1 (Draft: a flat fact record per SKU) is insufficient — Phil's correction is to model it at the level of individual **claims**, each capable of representing:

- **Sources** — one or more, each independently identifiable (a Skrybix field, a specific research document, a specific prior owner decision).
- **Evidence spans** — for research-derived claims, the actual passage or excerpt the claim is grounded in, not just a source name. This is what makes a claim auditable rather than a bare assertion with a citation-shaped label.
- **Confidence** — a genuine, calibrated estimate, distinguishable at a glance from a merely-present value.
- **Freshness** — when this claim was established/last verified, and whether it's due for revalidation (§10's knowledge lifecycle).
- **Contradictions** — when two sources disagree, both are retained, not silently overwritten; the contradiction itself is a first-class, visible fact until resolved (§6).
- **Precedence** — the rule that resolves a contradiction when one is needed: owner-confirmed fact outranks verified research, which outranks a single-source claim, which outranks an unverified assertion. Precedence is a property of the *model*, not an ad hoc per-field decision.
- **Applicability/scope** — does this claim apply to this exact SKU, this genus, this category, this marketplace, or globally? (Directly needed by §7's learning-scope rules and §10's cross-workflow knowledge reuse.)
- **Owner overrides** — Phil's explicit correction to any claim, which by construction outranks everything else for that claim's scope (§7).

A claim record, conceptually: `{ subject (SKU/genus/category/policy), predicate (e.g. "care.light_requirement"), value, source(s), evidence_span(s), confidence, established_at, last_verified_at, applicability_scope, contradicts (other claim IDs, if any), superseded_by (if any), owner_override (if any) }`.

This is the schema-level foundation for §10's Intelligence Repository — the Repository is where this model lives durably and across workflows; a SKU's assembled "Product Truth" (used by §4's pipeline) is a *view* over the subset of claims applicable to that SKU, not a separate flat table maintained in parallel.

## 6. Research, verification, provenance, and confidence model

**Source categories** (the commerce-fact-ownership taxonomy, unchanged from Revision 1, still the right classification for *where a value came from*):

1. Skrybix inventory/cutting data
2. Product/SKU commerce data (GM Commerce's own stored facts)
3. Gathering Moss store/category defaults (policy, deterministic)
4. Deterministic derived rules (e.g., exact-item disclosure from source system)
5. AI research (verified, cited, confidence-scored)
6. Genuine per-item human exception

**Evidence-type taxonomy** (added this revision — a different, complementary axis: the *epistemic status* of a piece of knowledge, independent of which of the six categories produced it):

- External published research
- Established horticultural consensus
- Supplier/manufacturer claims
- Community reports (lower default authority — unverified by construction until corroborated)
- Gathering Moss's own observations
- Gathering Moss's own controlled experiments (highest-authority *research*-type evidence, since Phil/Crystal directly produced it)
- Owner decisions (outranks all research-type evidence per §5's precedence rule)
- AI-derived recommendations (never their own evidence — see §10's protections)

A single claim's source category (the six above) and evidence type (the eight above) are recorded together — e.g., a care claim might be `source: AI research`, `evidence_type: established horticultural consensus`, with a specific evidence span and citation.

**Research is foundational, not a future enhancement** (Phil's explicit correction to Revision 1). From the earliest implementation phase (§12), research claims must carry:

- **Live retrieval** where the active AI provider supports it — the current OpenAI Responses API integration (`lib/ai/providers/openai-provider.ts`) has no web-search tool wired in today; enabling it is a real, near-term provider-capability change, not a distant one.
- **Source ranking** — preferring higher-authority sources (a peer-reviewed or established horticultural reference over an unverified forum post) when multiple sources speak to the same claim.
- **Citations** with **evidence spans** — the specific passage, not just a source name, per §5.
- **Freshness** — every retained claim has an age, and a revalidation policy appropriate to how fast that kind of fact changes (a marketplace policy needs far more frequent revalidation than a species' general light requirements).
- **Conflict detection** — when two retrieved sources disagree, that disagreement is recorded and surfaced, not silently resolved by picking whichever came back first.
- **Rejected-source reasoning** — when a source was considered and NOT used (low authority, contradicted by a higher-authority source, off-topic), that reasoning is retained too, so a later audit can see the system actually considered it rather than simply never having found it.

This is deliberately honest about what's real versus recalled: research that draws only on a model's trained knowledge, with no live retrieval and no citation, is a *lower-confidence, distinguishable* category of claim, never presented with the same confidence as a cited, evidence-spanned one. Revision 1's "AI-researched care instructions... calibrated confidence language" was a half-measure — calibrated language about an uncited claim is still an uncited claim. §12 corrects this explicitly.

**Independent validation** (architecturally distinct from generation — §4): deterministic checks (denylist, near-duplicate, alt-text uniqueness, length/presence bounds — already built, genuinely independent by construction) plus evidence-grounded validator calls for anything requiring judgment, given only the claim and its cited evidence, never the generation context, and explicitly instructed to attempt to refute rather than polish. Each validator has a defined outcome: repair automatically, or fail closed with a specific, inspectable reason — never a silent pass-through.

Confidence, freshness, and evidence surface to Phil at completed-package review time (§9), not hidden behind a uniform wall of equally-confident-looking text.

## 7. Owner-authority and durable-learning model

Unchanged, hard rule: Phil or Crystal approves before anything reaches a real marketplace or public publication. Nothing in this reset touches that. Refined this revision with explicit mechanics for how the system reduces repeat work without ever silently taking authority it hasn't been given:

- **Every owner correction is recorded automatically.** Any edit Phil or Crystal makes during review is itself a correction event — before/after value, what was corrected, in what context — with no separate "flag this as a correction" step required.
- **Rationale and provenance are retained.** What was corrected, why (when Phil provides a reason), and which listing/context it arose from.
- **A proposed scope is inferred, not assumed.** The system proposes whether a correction looks like a one-off fix to this exact item, or a pattern that should apply more broadly ("all Hoya listings," "all plant care text," "all pricing in this category") — it does not default to either extreme.
- **One-off corrections stay local.** A correction that looks like it only applies to this exact item is never silently generalized.
- **Applicable corrections are reused automatically, and this is testable.** Once a correction's broader scope has been confirmed (next bullet), it applies automatically to future matching cases — covered by the manual-handoff regression test category (§13), so "does this correction actually get reused" is a checkable fact, not an assumption.
- **Explicit confirmation is required only for consequential, policy-level changes** — establishing a *new standing rule* from an observed pattern. Once that rule is confirmed, applying it to the next matching case does not require Phil to re-approve every individual instance; only the decision to make it a rule at all is consequential enough to need his sign-off.
- **Every rule is inspectable, editable, reversible, and subordinate to Phil.** A real "Rules" surface (part of §10's Repository, exposed in Store Settings) lists every active durable rule, its scope, its origin correction, and lets Phil edit or delete any of them. Structurally, a standing rule is always a *default* — Phil's explicit action in the moment always overrides it, never the reverse.
- **Full audit trail.** Every autonomous value is traceable to its source category (§6) and, where owner-influenced, to the specific correction or approval that produced it.

## 8. Skrybix data and workflow gap analysis

Read directly from `lib/sources/skrybix.ts`: the entire commerce-export contract (`SkrybixCommerceRecord`) exposes only identity/state — `sku`, `displayName`, `parentSourceRecordId`, `plantRecordType`, `state`, `selectionState`, timestamps. This revision expands the gap analysis well beyond the two fields Revision 1 named:

| Missing from today's Skrybix handoff | Why it matters |
|---|---|
| Structured taxonomy (genus/species/cultivar) | Today only a free-text `displayName` — no reliable way to group, compare, or apply genus-level care knowledge (§10) across listings |
| Synonyms / common names | SEO and search value; prevents duplicate-seeming listings under different names for the same plant |
| Physical observations (leaf/node count, vine length, pot size) | Whatever Skrybix's own physical handling already produces at intake — currently discarded rather than handed off |
| Dimensions and weight | Real shipping/commerce facts, currently either defaulted crudely (§2) or re-asked in GM Commerce |
| Cost | Feeds real margin-aware pricing recommendations (§9) — a genuinely new commerce capability, not just a nice-to-have |
| Live inventory state | Prevents overselling a specific cutting across systems — ties directly to `ROADMAP.md` Milestone 5 |
| Photo associations | Skrybix may already have its own photos of the mother plant before GM Commerce's own photo pipeline ever runs — currently not reused at all |
| Sales/outgoing reconciliation | Read-only today — when a cutting sells via GM Commerce, nothing informs Skrybix's own record; a real bidirectional gap |
| Lifecycle data (mother→cutting propagation history) | Real provenance/story content — a genuine merchandising and educational-content asset (§10) currently lost |
| Provenance (where the mother plant originated) | Collector-value/story angle; same category as lifecycle data |
| Marketplace feedback | Zero feedback loop today — Skrybix never learns "this cultivar sells fast" from GM Commerce's own outcomes |
| Rooting status / condition notes | Revision 1's original two fields — still the most immediately actionable gap, since someone is already physically inspecting the plant in Skrybix at cutting-creation time |

Capturing these in Skrybix, at the point a human is already handling the physical plant, is strictly cheaper than re-asking later in GM Commerce, and keeps Skrybix authoritative for plant facts (unchanged from the existing "Skrybix remains separate and authoritative for plants" decision). This is a cross-repo proposal, not something GM Commerce can implement unilaterally — it should be handed to whoever owns Skrybix's backlog (Phil directly, or a GitHub Issue against `skrybix-webapp`), sequenced by which fields unlock the most value first: rooting status/condition and taxonomy/synonyms are the highest-value near-term additions; cost and lifecycle/provenance are higher-value but larger asks.

Price and inventory-state reconciliation are commerce-domain, not Skrybix-domain — they stay GM-Commerce-owned (§9), informed by Skrybix data where relevant (e.g., cost feeding a margin-aware price recommendation) but never originating from Skrybix directly.

Until Skrybix implements any of this, GM Commerce's own `/commerce/[sku]` remains the interim capture point — explicitly labeled as covering a Skrybix gap, not a permanent design choice.

## 9. Completed-package review design

`/review` should stop being an edit form and become a **completed package review**, and — this revision's correction — the "completed" package includes real recommendations as standard content, not a future add-on:

- Present the assembled listing the way a customer would see it — title, photos in order, price, description.
- Present the **core recommendations** produced at pipeline stage 5 (§4) alongside it: a suggested price with comparable-sales reasoning, confirmed taxonomy/collection assignment, an assessment of marketplace fit, a note on photography completeness ("no root-system shot found for this rooted-cutting claim" — from §4's image understanding layer), SEO strategy notes, and any merchandising angle.
- Show a compact, expandable evidence/confidence summary per claim (§5, §6): what's fact, what's policy-derived, what's cited research (and how confidently), what's an unverified assertion (clearly distinguished, never presented with equal weight).
- Surface the independent validator's results (§4, §6) — what it checked, what it found, what it repaired automatically.
- Primary actions are **Approve**, **Reject with reason**, or a **targeted regenerate** ("redo care details," "resuggest price") — not raw free-text editing as the default path. Direct editing remains available for the genuine exception case.

This is the literal implementation of Phil's own definition: "inspect the completed listing, correct exceptional inaccuracies if necessary, approve or reject" — where "completed" now genuinely includes the recommendations a real commerce director would have already produced, not just prose.

## 10. Gathering Moss Intelligence Repository

This is not a code repository, a collection of prompts, or additional columns on `listing_packages`. It is meant to be Gathering Moss's persistent institutional memory — a system capability the rest of GM Commerce (and, later, Skrybix, marketplace adapters, audits, and public content) reads from and writes to, architected so the business becomes measurably smarter tomorrow than it is today, per Phil's explicit framing.

### Two connected layers

**Layer 1 — OneDrive Evidence Library.** Durable *original* materials, in Phil's existing OneDrive capacity: research papers and PDFs, botanical references, supplier/manufacturer documents, original photographs and videos, growing observations and experiment records, care notes, product documentation, catalog exports, owner-created knowledge, marketplace policy captures, customer-question research, historical listings, published educational content, and any other supporting evidence for a factual claim. OneDrive stores the *files*, organized and preserved with stable identities, content hashes, version information, ownership/permissions, and source metadata — it is explicitly **not** the knowledge database or reasoning system.

**Layer 2 — the Intelligence Repository itself.** Structured entities: atomic claims (§5), citations, evidence relationships, confidence, source authority, freshness, contradictions, owner decisions, applicability, and learning outcomes. Every retained claim that cites original material must be able to locate and cite the specific OneDrive file (and, where applicable, the evidence span within it) that supports it — the Repository is the index and reasoning layer; OneDrive is the archive it points into.

### Ingestion pipeline

```
Discover or add source → preserve original in OneDrive → identify and hash
  → extract text/media metadata → classify → create atomic claims
  → verify against other sources → record citations and evidence spans
  → promote approved knowledge → retrieve for future work
  → monitor freshness and revalidate
```

Governance, to keep OneDrive from becoming an unsearchable dumping ground:
- Folder and naming conventions, with a stable document ID independent of filename or location.
- Source-type metadata on every item (which of §6's evidence types it is).
- Content-hash-based duplicate detection.
- OCR/text extraction for scanned or image-based documents, so content is actually searchable, not just archived.
- Version handling — a superseded document is retained (never deleted outright) but clearly marked as superseded, with a pointer to what replaced it.
- Citation anchors — a stable way to reference a specific passage/page/timestamp within a source, not just the document as a whole.
- Retention rules, appropriate to source type (a supplier catalog PDF and a customer-question research note don't need the same retention policy).
- Explicit permissions, matching who's actually authorized to see or edit a given category of material.
- Research status and verification status as first-class, queryable fields (discovered / under review / verified / rejected / superseded).
- Explicit relationships to what a source applies to — plant, product, genus, species, marketplace, or policy — the same applicability/scope concept from §5, applied at the source-document level.

### Knowledge lifecycle

```
Discover → verify → reconcile → retain as evidence → promote to usable knowledge
  → apply → measure outcome → revalidate, revise, or retire
```

Nothing retrieved is treated as permanent truth on arrival. It's retained as *evidence* first; promotion to knowledge the system will actually act on requires verification and reconciliation against what's already known (§6). Once applied, its real-world outcome is measured (did a price recommendation built on this evidence hold up; did a care claim get contradicted by a customer report), and that outcome feeds back into whether the claim is revalidated, revised, or retired.

**Authority hierarchy** (restating §5's precedence rule at the Repository level): owner-confirmed knowledge is always highest-authority. Verified research may inform recommendations but stays clearly distinguishable from owner policy and product-specific fact — never silently merged into the same confidence tier. Conflicting information is retained and resolved transparently (§6's conflict detection), never silently overwritten.

**Automatic retrieval.** The system retrieves applicable knowledge automatically for every future task that could use it — Phil should never have to repeat an answer, preference, correction, or business rule the system has already been given once and had confirmed.

**How knowledge learned in one workflow becomes safely useful elsewhere** — concretely, per Phil's own examples:
- A corrected Hoya care instruction improves future Hoya listings (genus-scoped reuse, §7).
- A pricing override informs future price *recommendations* for similar items without becoming an uncontrolled universal rule (scoped, proposed, confirmed — §7).
- A rejected phrase becomes a durable brand-language restriction (a policy-level rule, confirmed once, applied everywhere — §7).
- A marketplace compliance discovery updates every future adapter's checks for that marketplace (scoped to the marketplace, not the SKU).
- A customer question revealing missing information flags that future listings for similar products should proactively answer it (feeds §9's recommendation stage).
- Sales performance changes future merchandising and channel recommendations (a measured-outcome feedback loop, per the lifecycle above).

**Explicit protections against bad learning** (verbatim requirements, restated as hard architectural constraints, not aspirations):
- Unverified research must not become accepted truth.
- One-off corrections must not be generalized without appropriate scope (§7).
- Marketplace-specific rules must not corrupt canonical product truth.
- Old knowledge must not remain active after it becomes stale or superseded (freshness/revalidation, above).
- AI-generated claims must not become their own supporting evidence (a claim's evidence must trace to a source outside the claim itself — this is why §4's independent validation is architecturally separate from generation).
- Repetition across copied sources must not be mistaken for independent verification (source ranking and provenance tracking, §6, must distinguish "three sites repeating the same original claim" from "three independently-sourced confirmations").
- Performance correlation must not automatically be treated as causation (a sales-performance-driven merchandising suggestion is a *recommendation* Phil evaluates, never an automatic rule).
- No learned behavior may override Phil's authority (§7, restated here as a Repository-level invariant, not just a workflow-level one).

### Reusable, cross-system capability

The Repository is architected as a capability available to Skrybix, GM Commerce, marketplace adapters, audits, recommendations, and future Gathering Moss systems — not knowledge trapped inside one listing workflow. Concretely, this means the Repository is its own service/data boundary (however it's eventually hosted), with a defined read/write contract, not a table that only `app/actions.ts` happens to query.

### Publication and editorial layer — supporting the public-knowledge objective

The broader company objective (§1): Gathering Moss becomes a trusted source of plant knowledge, not merely a seller. The same verified Repository is meant to eventually support care guides, species/cultivar profiles, educational articles, customer support answers, social and email content, internal growing procedures, research summaries, comparison and troubleshooting resources, AI-assisted plant-knowledge tools, and publicly citable Gathering Moss reference content.

Public content must be **generated from the verified Repository, not from an unconstrained prompt** — the same architectural discipline as commerce copy (§4's "write" stage consumes only reconciled facts), extended to a distinct publication pipeline:

```
Research → verification → internal knowledge → owner/editorial approval
  → public educational asset → audience feedback and performance
  → new research questions → repository improvement
```

Every factual public claim must be traceable to evidence or an explicitly-labeled Gathering Moss observation, must distinguish external research from Gathering Moss's own findings, must preserve genuine uncertainty rather than presenting it as settled, and must actively avoid repeating unsupported claims that are already common (and often wrong) in the wider plant community — the whole point of building a *verified* repository rather than another AI-content site.

This layer is explicitly a **later** phase (§12) — it depends on the Repository having enough verified, mature knowledge to publish responsibly from, which depends on the commerce-facing work (§4–§9) actually running and accumulating real evidence first. Building it before the Repository has real content would just produce more unverified plant-knowledge content, the opposite of the objective.

### Measurable learning evaluations

- Does an owner correction improve the next applicable recommendation?
- Does the correction avoid affecting unrelated products (scope correctly inferred, §7)?
- Does verified research get reused without unnecessary repeated research?
- Is stale knowledge identified and revalidated on schedule?
- Are contradictions surfaced and resolved correctly, rather than silently dropped?
- Can every generated claim be traced back to current evidence or owner knowledge?
- Does owner effort (§13's "owner minutes per SKU") decrease over time?
- Does factual accuracy (§13's "factual-error rate") improve over time?
- Do recommendations improve based on measured outcomes, not just repeat unchanged?
- Is the system accumulating institutional capability — reusable, citable, compounding — rather than merely accumulating text?

## 11. Response to Copilot's "Required critique of Claude's reset proposal"

**Blocked.** I don't have read access to `docs/autonomous-commerce-intelligence-audit.md` — the repository it lives in returned 404 both via an unauthenticated `git clone` and via a direct fetch, and the `gh` CLI isn't installed in this environment to attempt an authenticated request. I don't have the actual text of Copilot's ten questions, and I'm not willing to fabricate plausible-sounding questions and "answer" them — that would misrepresent an independent critique I haven't actually read.

To close this out honestly, I need one of:
- Read access granted to `hydracoresystems-listing-audit-engine` (or wherever the audit doc actually lives) for the credential this environment uses, or
- The ten questions pasted directly, or
- The full audit document pasted or attached.

Once I have the actual text, I'll add the response table here as its own revision, before implementation resumes — this section is a placeholder, not a skip.

## 12. Migration plan

Nothing built so far is wasted — most of it is exactly the right foundation. This revision reorders the phases so research/evidence architecture lands early, not late, per Phil's correction that it's foundational.

**Keep as-is, unaffected by this reset:**
GMCOM-009's atomic reserve/finalize/release lock, the provider-neutral AI abstraction, the real Shopify GraphQL client, the photo pipeline's deterministic ingestion/derivative generation, the Commerce Readiness Gate's fail-closed mechanism (its *content* changes as fields move from "ask" to "derive"; its mechanism doesn't).

**Phase A — deterministic defaults and structural fixes** (mechanical, low-risk, real prerequisites): price ownership moved to `commerce_details` (closes the real regeneration data-loss bug), deterministic store defaults for inventory policy/shipping text/exact-item disclosure/category/collections, automatic sales-summary and alt-text duplicate repair.

**Correction to Revision 1's Phase A:** "AI-researched care instructions" using only pretrained model knowledge, even with calibrated-confidence language, is **not** a completed capability — it's a plausible-sounding but uncited assertion. It stays explicitly labeled interim/incomplete until Phase B's real research layer (live retrieval, citations, evidence spans) is in place. Do not present calibrated-but-uncited care text as though it satisfies the "AI research" requirement.

**Phase B — claim-level evidence model + Repository foundation.** Formalize §5's claim/evidence schema; stand up the Intelligence Repository's core structure (§10, Layer 2) with at minimum: claims, sources, evidence spans, confidence, freshness, contradictions, precedence, applicability. Wire live retrieval and citation capture into the research layer (§6) — this is where "research" actually becomes foundational rather than aspirational.

**Phase C — OneDrive Evidence Library + ingestion pipeline.** Stand up the OneDrive-backed Layer 1 (§10): folder/naming conventions, hashing, duplicate detection, source-type metadata, citation anchors, and the discover→preserve→hash→extract→classify→verify→cite→promote pipeline. Sequenced alongside/after Phase B since claims need somewhere real to cite back to.

**Phase D — image understanding.** Wire a genuinely multimodal request type into the AI provider abstraction; build the identity/hero/angle-coverage/condition/crop/background/duplicate-view/missing-shot checks (§4).

**Phase E — independent validation layer.** Separate the validator from the generator architecturally (§4, §6): deterministic checks formalized as a distinct pass; evidence-grounded validator calls for judgment-requiring claims.

**Phase F — core recommendation stage.** Price/taxonomy/collection/marketplace-suitability/photography/SEO/merchandising recommendations as a standard pipeline stage (§4 stage 5, §9) — no longer deferred.

**Phase G — owner-editable durable config + Rules surface.** Move today's TypeScript-constant defaults to DB-backed, owner-editable config; build the "Rules" UI (§7) for inspecting/editing/revoking durable learned rules.

**Phase H — completed-package review UI redesign** (§9).

**Phase I — broader proactive portfolio recommendations** — genuinely deferred, since it depends on cross-SKU comparison volume, not on anything architecturally harder.

**Phase J — Skrybix gap proposal delivered to Skrybix's own backlog** (§8) — cross-repo coordination, not GM Commerce code.

**Phase K — publication/editorial layer** (§10) — deliberately last; depends on the Repository having accumulated real, verified content.

Etsy publishing stays paused, per Phil's explicit instruction, until at minimum Phases A–F land and HY-LOB01-C04 passes the completion test end-to-end as the proof case, and until Copilot's independent re-review (§11, once unblocked) of this revision is complete.

## 13. Autonomy and intelligence test strategy

- Operationalize the governing completion test as a repeatable check: for a sampled real product run through the full pipeline, count how many fields/actions required Phil/Crystal input beyond genuine (A)-category facts and final approval; a milestone isn't done if that count is nonzero for anything on the (B) list in §3.
- **Adopt Copilot's evidence artifacts and operational autonomy metrics directly:**
  - Owner minutes per SKU (trending down = the system carrying more real weight).
  - Unnecessary escalation rate (how often the system asks Phil something it should have resolved itself — the direct operational expression of the completion test).
  - Factual-error rate (claims later found wrong, via customer complaint, Phil's own correction, or revalidation).
  - Correction reuse rate (how often a durable rule created from one correction successfully prevents a repeat elsewhere — §7, §10).
  - Citation coverage (% of research-sourced claims carrying real evidence/citation vs. bare trained-knowledge assertion) and freshness (age distribution of retained knowledge; how much is overdue for revalidation).
  - Staff-equivalent throughput per owner hour — the most direct expression of the governing completion test as a number: fully-processed, Shopify-ready SKUs per hour of Phil/Crystal time actually spent.
- New regression-test category: a fixture-driven "manual-handoff regression test" asserting a real product profile, run through the full pipeline with zero manual entry beyond genuinely owner-only facts, produces a Commerce-Readiness-Gate-passing package. Puts the completion test in CI, not just one-off human review.
- Keep and extend the existing deterministic test suite (102 tests as of this pause) for the gate, defaults, and repair loops.
- Adversarial-content regression fixtures drawn from HY-LOB01-C04's actual real defects (internal-language leak, near-duplicate summary, duplicate alt text) — real, already-observed failure modes, the strongest possible regression cases.
- §10's ten measurable learning evaluations, tracked over time, not just checked once at ship.

## 14. Revised roadmap and implementation sequence

`ROADMAP.md`'s milestones 2 and 3 (SKU-to-listing-draft, Shopify draft publishing) are already built, ahead of the document's own sequencing — but built to the old "ask a human" standard this reset corrects. The honest framing: **Milestones 2 and 3 need the Phase A–H retrofit in §12 applied to them before they're actually done**, and Milestone 4 (Etsy) and Milestone 5 (sale/inventory coordination) should be built on the reset architecture from the start.

Immediate next steps, pending Phil's review of this revision and Copilot's independent re-review:
1. Resolve §11's blocker — get access to or the text of Copilot's ten-question critique, and complete the response table.
2. Finish Phase A (mechanical defaults/fixes, already substantially coded), with the corrected — not overstated — characterization of care-instruction research.
3. Build Phase B (claim/evidence model + Repository foundation) and Phase C (OneDrive Evidence Library) together, since Phase B's citations need Phase C's evidence store to point into.
4. Resume HY-LOB01-C04's real rebuild end-to-end through the corrected pipeline as the proof case for the completion test (§13).
5. Only then resume Etsy planning — unchanged from Phil's explicit instruction.

**Assumptions this reset discards:**
- "Human review" = filling out or editing a form. It means inspecting a finished, recommendation-complete package and deciding.
- "The gate found a blocker" = "ask Phil." It means "did the system try every non-owner source, and every research/derivation path, first."
- Store policy as a TypeScript constant is the destination. It's an interim shim on the way to owner-editable durable config.
- One AI call handles both research and copywriting. They're different jobs, and research isn't complete without live retrieval, citations, and independent verification.
- A model re-reading its own draft is verification. It isn't — independent validation is architecturally separate.
- Recommendations are a future nice-to-have. They're core to what "completed package" means.
- OneDrive would be the knowledge database. It's the evidence archive underneath a structured Repository, not a substitute for one.
- Etsy is simply "next" per `ROADMAP.md`'s literal order. Paused until the reset's completion test passes for Shopify and Copilot's re-review is complete.
