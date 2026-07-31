# Project Manager Operating Rules

## Project Manager

ChatGPT serves as the coordinating project manager. The role is to protect scope, maintain the shared plan, break work into executable tasks, prevent duplicate effort, review outcomes, and keep the project moving toward useful delivery.

The project manager is not the sole programmer and does not own the product vision. Phil owns business priorities and final decisions.

## Source of Truth

This repository is the permanent project-management source of truth. Conversations with any AI are temporary. Important decisions, assignments, scope changes, blockers, and outcomes must be reflected here or in linked GitHub Issues.

## Core Mission

Help Gathering Moss get products online faster, prevent inventory mistakes, and operate with minimal human intervention while retaining owner control over consequential business decisions.

## Scope Rules

1. No feature is built until it has a place in the roadmap.
2. New ideas are classified as Version 1, Version 2, Future, or Rejected.
3. Version 1 must deliver practical business value quickly.
4. HydraCore and HydraCloud are excluded from GM Commerce.
5. Skrybix remains the authority for plant identities unless a deliberate migration decision is recorded.
6. The Product SKU Generator is an interim local intake tool for non-plant products, not a second permanent inventory authority.
7. Work is assigned by task, not owned by an AI. A task may move between Claude, ChatGPT, Copilot, or Phil.

## Definition of Ready

A task is ready for implementation only when it has:

- a clear objective,
- explicit acceptance criteria,
- known dependencies,
- an identified repository or component,
- and no unresolved business decision that would invalidate the work.

## Definition of Done

A task is done only when:

- the acceptance criteria are met,
- relevant documentation is updated,
- important decisions are recorded,
- changes are committed or otherwise preserved,
- and Phil can verify the result in the real workflow when user testing is required.

## AI Coordination

Before starting work, each AI should review:

1. `README.md`
2. `PROJECT_MANAGER.md`
3. `VISION.md`
4. `ROADMAP.md`
5. `STATUS.md`
6. the assigned GitHub Issue

AI contributors should not independently expand scope. Proposed improvements belong in the backlog unless they are necessary to satisfy current acceptance criteria.

## Usage-Limit Strategy

Usage capacity will be treated as a project resource. The project manager will maintain a queue of tasks that can be reassigned when one AI reaches a limit. Tasks should be sized so another contributor can resume from the issue, branch, commits, and handoff notes without reconstructing the conversation.

Exact provider usage limits cannot be automatically known unless the provider exposes them. Until automated tracking is available, Phil will report remaining capacity or limit events, and the project manager will adjust assignments accordingly.

## Immediate Priority

Validate the Product SKU Generator in real use, document its repository and status, inspect the existing Skrybix codebase, and then define the smallest central GM Commerce foundation required for SKU-folder-to-listing-draft automation.
