# GM Commerce Roadmap

## Milestone 0 — Product Intake Starter

**Goal:** Create unique non-plant SKUs and matching photo folders with minimal effort.

**Current state:** Built by Claude as the local Product SKU Generator. Awaiting real-world validation and repository registration.

**Exit criteria:**

- numbering behaves as intended across categories and attributes,
- folders are created in the correct OneDrive location,
- categories can be managed without file editing,
- counters and history survive restart,
- local data is backed up,
- and known limitations are documented.

## Milestone 1 — Central Commerce Foundation

**Goal:** Define and create the smallest durable product and listing data foundation needed by GM Commerce.

**Required outcomes:**

- central product record model,
- clear relationship to Skrybix plant identities,
- clear relationship to Product SKU Generator records,
- listing-draft status model,
- marketplace identifier storage,
- and documented source-of-truth rules.

No elaborate dashboard, analytics engine, or unnecessary migration belongs here.

## Milestone 2 — SKU Folder to Listing Draft

**Goal:** Detect a prepared SKU photo folder and generate a complete listing draft for owner review.

**Initial draft fields:**

- SKU,
- product category and attributes,
- selected photos,
- title,
- description,
- price,
- Shopify product type,
- tags,
- and listing status.

## Milestone 3 — Shopify Draft Publishing

**Goal:** Publish an approved draft to Shopify as a draft product and store the returned Shopify identifiers and status.

## Milestone 4 — Etsy Publishing

**Goal:** Reuse the shared product and listing model to create and manage an Etsy listing without duplicating product truth.

## Milestone 5 — Sale and Inventory Coordination

**Goal:** Detect sales and prevent overselling or stale duplicate listings across connected channels.

## Version 2 and Later

Possible later work includes broader marketing assistance, performance analytics, automated repricing recommendations, deeper GM Money integration, and more autonomous merchandising. These must not delay the Version 1 sales workflow.
