# GM Commerce Coordinator Handoff

_Current as of 2026-08-07 after Phase H Slice 4 merged (app `main` `606c5a5`). For the current state snapshot see `handoffs/2026-08-07-phase-h-status-and-claude-handoff.md` and `STATUS.md`; this file defines the coordinator's standing operating responsibilities and is not a substitute for live GitHub state._

## Purpose

This document is the durable takeover guide for the AI project coordinator. It
summarizes the current state, explains where work belongs, and defines the
coordinator's operating responsibilities. Verify live GitHub state before acting;
this file is a handoff, not a substitute for repository history, issues, or CI.

Phil owns product vision, priorities, authorization, and final merge decisions.
The coordinator turns those decisions into finite, dependency-ordered work,
maintains headquarters records, prepares complete implementation briefs, checks
results, and keeps the next task unambiguous.

## Repository boundaries

- `HydraCoreSystems/gm-commerce` is the application repository. All application
  code, tests, database migrations, and CI changes are implemented and merged
  there.
- `HydraCoreSystems/gm-commerce-hq` is the project-management headquarters.
  Roadmaps, decisions, slice plans, assignments, issues, and handoffs belong here.
- Skrybix remains authoritative for plant identity and plant SKU records.
- Product SKU Generator remains responsible for non-plant SKU generation and its
  matching local photo folders.
- HydraCore and HydraCloud are outside GM Commerce scope.

Do not place application code in headquarters. Do not place project-management
records in the application repository unless they directly document that code.

## Required startup sequence

At the start of every coordinator session:

1. Read `CLAUDE_ONBOARDING.md`, `README.md`, `PROJECT_MANAGER.md`, `VISION.md`,
   `ROADMAP.md`, `STATUS.md`, `DECISIONS.md`, `BACKLOG.md`, and `AI_HANDOFF.md`.
2. Read the latest product-reset or phase-plan document relevant to the work.
3. Inspect both repositories' open issues, pull requests, default-branch tips, and
   CI state. Treat GitHub as authoritative when this handoff disagrees with it.
4. Confirm the most recently merged dependency before authorizing the next slice.
5. Confirm Phil's authorization. Planning may be proposed without implementation,
   but implementation of a new major slice requires his approval.
6. Give the implementer a standalone brief with exact base commit, branch, scope,
   acceptance criteria, tests, live-CI requirements, exclusions, and stop condition.

## Current application status

Phase F is in progress in `HydraCoreSystems/gm-commerce`.

Completed and merged:

- Phase F Slice 1: generic recommendation persistence and Section 18
  `SelectionTrace` foundation.
- Phase F Slice 2: price recommendation service, PR #24.
- Phase F Slice 3: taxonomy and collections recommendation services, PR #25.
  It merged to `main` as merge commit
  `40e672f011442fc85d2527ba23ad6c5840a5eb1c`.

PR #25's head was `0022a2574324186c8baf25928bf8e63acd750e48`.
Its typecheck, test suite, build, schema-from-empty job, all prior live jobs, and
the new `phase-f-slice3-taxonomy-collections-live` Postgres job passed. Duplicate
CI runs for that SHA were trigger noise, not a product failure.

No new migration was required for Slice 3. The generic recommendation kind/value
and trace schema from Slice 1 covers taxonomy and collections, and Slice 2's
precedence shape guard remains reusable.

Before relying on these SHAs, run:

```bash
gh api repos/HydraCoreSystems/gm-commerce/commits/main --jq '.sha'
gh pr list --repo HydraCoreSystems/gm-commerce --state open
gh run list --repo HydraCoreSystems/gm-commerce --limit 20
```

## Phase F scope and sequencing

The governing product-reset scope defines Phase F as core recommendation services:

- price;
- taxonomy;
- collections;
- marketplace suitability;
- photography;
- SEO;
- merchandising.

Every recommendation service must produce a complete Section 18
`SelectionTrace`.

Price, taxonomy, and collections are merged. The remaining named service areas are
marketplace suitability, photography, SEO, and merchandising. Headquarters does
not currently contain an approved finite Phase F slice-plan document comparable to
`phase-e-slice-plan.md`. Do not invent or claim a final Phase F slice count.

The immediate proposed next unit is Phase F Slice 4: marketplace suitability.
Before implementation starts, the coordinator should record a complete Phase F
slice plan in headquarters, or at minimum create and approve a ready Slice 4 issue
whose scope is checked against the product-reset specification and current
`gm-commerce/main`.

## Proposed Phase F Slice 4

Suggested branch:

```text
agent/phase-f-slice-4-marketplace-suitability
```

Suggested implementation scope:

- Add a marketplace-suitability recommendation service under
  `lib/recommendations/`, following the merged price, taxonomy, and collections
  patterns.
- Produce `kind = 'marketplace_suitability'` recommendations.
- Evaluate marketplaces independently so multiple suitable marketplace candidates
  can coexist.
- Resolve conflicts for the same marketplace with the established precedence
  rules.
- Include marketplace identifier, suitability outcome, reasoning, and bounded
  confidence.
- Produce exactly one complete Section 18 trace per service invocation.
- Record all considered, selected, and rejected claims and every applicable
  precedence decision.
- Surface the Phase E compliance gate as context. A FAIL gate sets
  `blockedFromApproval` but must not suppress recommendation generation.
- Preserve environment, tenant, and subject isolation.
- Reuse existing schema and helpers. Add no migration unless the implementer
  demonstrates a real schema limitation.

Required test coverage should include independent multi-marketplace results,
same-marketplace conflicts, verified-versus-under-review precedence, owner
override precedence, FAIL-gate non-suppression, confidence bounds, deterministic
ordering, isolation, exactly-one-trace cardinality, Section 18 completeness, and
malformed-trace rejection.

Add a live-Postgres CI job mirroring the merged Phase F jobs. It must exercise
precedence round-trip, FAIL-gate context, confidence bounds, trace cardinality, and
malformed-trace rejection against the real database.

Explicitly exclude photography, SEO, merchandising, publishing adapters, remote
publication verification, UI, policy editing, and unrelated schema redesign.

This is a proposed brief, not evidence of Phil's authorization. Confirm approval
and current code before assigning it.

## Coordinator responsibilities

### Plan and authorize work

- Convert the roadmap into finite dependency-ordered slices.
- Keep one branch and one pull request per slice.
- Ensure each slice has a ready headquarters issue or approved plan.
- Never let an implementer infer acceptance criteria from chat history.
- Prevent scope drift, especially into later phases or adjacent products.

### Prepare implementation briefs

Every brief must state:

- repositories and source-of-truth documents to read;
- exact base branch and verified base SHA;
- branch name;
- objective and required behavior;
- files or interfaces likely involved, without over-prescribing design;
- acceptance criteria and edge cases;
- unit, integration, live-Postgres, typecheck, build, and CI expectations;
- explicit exclusions;
- required handoff contents;
- stop condition.

### Review and merge

- Verify the PR targets `main` and is based on the intended dependency.
- Review changed files and behavior, not only the implementer's summary.
- Confirm all required checks completed successfully on the current head SHA.
- Distinguish harmless duplicate CI runs from actual missing or failed checks.
- Verify migrations from an empty database when schema changes exist.
- Confirm tests cover precedence, isolation, compliance-gate context, trace
  completeness, cardinality, and malformed persisted data where applicable.
- Ask Phil for the merge decision. Never infer authorization merely from green CI.
- When Phil explicitly authorizes merge, protect against head movement and merge
  the reviewed SHA.

Example:

```bash
gh pr view <number> --repo HydraCoreSystems/gm-commerce \
  --json state,mergeable,mergeStateStatus,headRefOid,statusCheckRollup
gh pr merge <number> --repo HydraCoreSystems/gm-commerce \
  --merge --match-head-commit <reviewed-full-sha>
```

### Maintain durable records

After each planning or merge boundary:

- update `STATUS.md` so its top authoritative section reflects reality;
- record durable business or architecture decisions in `DECISIONS.md`;
- update the relevant phase-plan document and issue;
- add deferred, nonessential work to `BACKLOG.md`;
- leave a concise handoff in `AI_HANDOFF.md` or `handoffs/`;
- commit and push headquarters changes.

Do not allow important scope, corrections, or sequencing decisions to exist only
in chat.

## Known coordination risks

- Headquarters `STATUS.md` may lag the live application repository. Always verify
  GitHub first, then repair the document.
- Earlier Phase F work exposed incorrect assumed base SHAs. Never accept a task's
  base SHA without checking that it exists and is reachable from the intended
  branch.
- Dependent slices must merge in order. If a branch was cut from an unmerged prior
  slice, either merge the dependency first or explicitly retarget/rebase.
- Browser authentication, Git credentials, and GitHub CLI authentication are
  separate. If `gh` is unauthenticated, use `gh auth login --web`; do not treat
  that as a code or branch failure.
- A compliance gate is contextual for recommendation generation. FAIL blocks
  approval but does not erase or suppress the candidate recommendation.
- Generic recommendation persistence currently supports new kinds without a
  migration. Require evidence before approving schema churn.
- CI comments must describe the scenario the SQL actually exercises. Fix misleading
  owner-override wording if the test is verified-versus-under-review precedence.

## Immediate takeover checklist

1. Verify `gm-commerce/main` contains merge commit `40e672f` or its successor.
2. Update stale headquarters status documents with the completed Phase E and
   Phase F Slice 1-3 history.
3. Draft and commit an authoritative finite Phase F slice plan covering the four
   remaining service areas and dependencies.
4. Present the plan to Phil for approval; do not guess the total slice count.
5. After approval, create the Slice 4 headquarters issue and implementation session
   from current `main`.
6. Track the implementation through PR, review, green CI, explicit merge
   authorization, merge, and headquarters closeout.

## Definition of successful coordination

At any stopping point, another coordinator should be able to identify, from GitHub
alone:

- what is merged;
- what is in progress;
- what comes next and why;
- who authorized it;
- its exact scope and acceptance criteria;
- its dependency and base SHA;
- what was verified;
- what remains blocked or deferred.

If any of those answers exists only in conversation, the coordination job is not
finished.
