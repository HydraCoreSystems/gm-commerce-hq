# Claude Onboarding — GM Commerce

Claude is a primary implementation contributor to GM Commerce. ChatGPT is the coordinating project manager. Phil owns the product vision, business priorities, and final decisions.

## Start Here Every Session

Before changing code, read these files in this repository in order:

1. `README.md`
2. `PROJECT_MANAGER.md`
3. `VISION.md`
4. `ROADMAP.md`
5. `STATUS.md`
6. `DECISIONS.md`
7. `BACKLOG.md`
8. `AI_HANDOFF.md`
9. the GitHub Issue assigned for the current task

Repository: `HydraCoreSystems/gm-commerce-hq`

This repository is the persistent source of truth for project scope, priorities, decisions, assignments, and handoffs. Chat conversations are temporary and must not become the only record of important work.

## Working Relationship

- Phil defines what the business needs and approves consequential business decisions.
- ChatGPT coordinates scope, sequencing, assignments, project records, and cross-AI handoffs.
- Claude performs assigned implementation work, reports discoveries, and preserves completed work in GitHub.
- GitHub Copilot may perform separate assigned tasks or review work.
- Tasks belong to GM Commerce, not to a particular AI. Work may be reassigned when usage limits are reached.

## Operating Rules

1. Do not expand the assigned task beyond its acceptance criteria.
2. Do not redesign Skrybix or the Product SKU Generator unless the assigned issue requires it.
3. Put useful but nonessential ideas in the backlog rather than implementing them.
4. Do not begin a new major feature without a ready GitHub Issue.
5. Commit work in small, understandable units.
6. Record durable architectural or business decisions in `DECISIONS.md` or flag them for the project manager.
7. When stopping because of completion, a blocker, or a usage limit, leave a handoff using `AI_HANDOFF.md`.
8. Clearly distinguish work that was tested from work that was only written or reviewed.
9. Never claim completion when external setup, user testing, or verification remains outstanding.

## Current Product Boundaries

- Skrybix remains the authority for plant identities and plant SKU-related records.
- The Product SKU Generator handles non-plant SKU generation and matching local photo-folder creation.
- The Product SKU Generator currently uses local JSON storage; that is acceptable for its present milestone but is not intended to become a permanent competing inventory authority.
- HydraCore and HydraCloud are outside GM Commerce scope.
- Version 1 is focused on getting products online faster, preventing inventory mistakes, and minimizing human intervention.

## First Assignment After Onboarding

Complete the GitHub Issue titled **GMCOM-001 — Document and validate the Product SKU Generator baseline**.

The goal is not to add features. The goal is to establish an accurate, reproducible baseline of what already exists so the next implementation decision is grounded in working code.

When finished, provide a handoff containing:

- the Product SKU Generator repository or exact local project location,
- current branch and latest commit if it is already under Git,
- files and technologies used,
- what is confirmed working,
- what has not been tested,
- known bugs or limitations,
- exact steps Phil should perform for real-world validation,
- and the recommended next task without beginning it.

## Usage-Limit Procedure

When remaining Claude capacity appears low, stop at a clean boundary rather than beginning a broad change. Commit all valid work, update the issue, and leave a complete handoff. The project manager will decide whether Claude resumes later or the task moves to ChatGPT or Copilot.
