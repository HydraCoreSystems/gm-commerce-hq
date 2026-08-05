# Phase F Slice 6 Handoff — SEO Recommendation Service

## Task

- Issue: Phase F Slice 6 (SEO recommendation service) implementation
- Objective: Implement the SEO recommendation service producing one `Recommendation` (kind `seo`) with one complete §18 `SelectionTrace` per invocation, following the exact patterns of merged Slices 2–5
- Repository: `HydraCoreSystems/gm-commerce`
- Branch: `agent/phase-f-slice-6-seo`

## PR

- **PR #28**: https://github.com/HydraCoreSystems/gm-commerce/pull/28
- **Head SHA**: `151cf3ca36dbdaa33d5ee8fcd319903023fb249a`
- **Base SHA**: `a46a04295da74fea46d2b38294284b74d08579aa` (brief's verified base; confirmed against `origin/main` HEAD before branching)
- Mergeability: MERGEABLE (no conflict)

## Work Completed

- Implemented `lib/recommendations/seo.ts`: pure `buildSeoRecommendation` + guarded `SeoRecommendationServiceImpl` via `createSeoService` (symbol-token private constructor, environment bound at construction, atomic persistence through `recordRecommendation`/`gmcom_record_recommendation`, trust boundary identical to Slices 2–5).
- Data sources: SEO-relevant Claims via `queryApplicableClaims` filtered by the `seo.` predicate **prefix** (forward-compatible; no such claims exist today) + listing-package content read directly from the committed `listing_packages` table.
- Deterministic signals with brief-documented thresholds: `title_length` (>=20), `description_length` (>=120), `tags_count` (pass >=3, warning 1–2, fail 0), `product_type` (presence).
- Precedence: owner-override always wins; verified claims beat deterministic signals; under-review/stale surfaced but never override. Claim-vs-claim conflicts use the F2–F5 precedence mechanism. When a claim overrides a deterministic signal, the conflict and rule are recorded in `precedenceDecisions` (vs. `deterministic:<field>` pseudo-participant).
- Confidence: 0.80 deterministic-only; average of claims component (with F2–F5 discounts: conflict 0.90, stale 0.85) and 0.80 when claims contribute; 0 for `no_seo_signals`; bounded [0,1]. Listing-package absence returns `no_seo_signals` with confidence 0, empty signals, and a complete trace (never throws).
- Statuses: `seo_assessed` / `no_seo_signals`. Compliance gate (PASS/FAIL/STALE/NO_CHECK) is context-only, identical to Slices 2–5 — never suppresses the recommendation.
- Added `phase-f-slice6-seo-live` CI job in `.github/workflows/ci.yml` (after the F5 job): deterministic-signal round-trip (confidence 0.80), no-signals case (confidence 0), verified-claim override precedence round-trip (confidence 0.85), FAIL-gate context case, confidence 1.5 DB rejection, malformed precedenceDecisions (missing `rule`) DB rejection.

## Verification

- Tests: **53 passed** (`npx vitest run lib/recommendations/seo.test.ts`), including the test-helper fix (default `listingContent` to present fixture)
- Typecheck: `npx tsc --noEmit` passed
- Build: `npm run build` passed (only pre-existing unrelated `libheif-js` critical-dependency warning)
- CI: all checks green on both push and PR runs, including the new `phase-f-slice6-seo-live` job, `verify`, `schema-from-empty`, `schema-drift-deferred`

## Implementation Decisions (flagged for DECISIONS.md)

1. **Schema reconciliation for listing-package fields**: the brief names `title`, `description`, `tags`, `product_type`, but the committed migrations (GMCOM-007/008/009) name them `proposed_title`, `description`, `tags`, `category`. The service maps `title`→`proposed_title` and `product_type`→`category` (mirroring the F5 `is_hero` discipline of adapting to the real schema).
2. The service does **not** read `listing_packages.seo_title`/`seo_description` — these are undocumented live-migration-only columns; SEO-relevant value enters via `seo.`-prefixed claims.

## Confirmation

- **No migration was added** (schema work deferred, consistent with Slices 2–5).

## Stop Condition

Per the brief: all acceptance criteria met, all tests pass, PR open against `main`, handoff written. **Not starting Slice 7.**
