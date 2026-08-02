# Marketplace Adapter Contract

## Purpose

Define the smallest stable boundary needed to publish one reviewed Listing Package and one approved Commerce Asset Package to multiple sales channels without embedding marketplace logic in the core workflow.

This contract is intentionally narrow. It exists to accelerate Shopify draft publishing after the copy-ready output is usable.

## Core rule

GM Commerce owns canonical commerce content and approved assets. Marketplace adapters translate that approved material into channel-specific draft records.

Adapters must not become alternate sources of truth for product identity, listing copy, or asset approval.

## Required adapter interface

```ts
export type MarketplaceKey = "shopify" | "etsy";

export interface MarketplaceDraftInput {
  sku: string;
  listingPackageVersion: number;
  title: string;
  description: string;
  shortTitle?: string | null;
  price?: number | null;
  category?: string | null;
  tags: string[];
  careDetails?: string | null;
  assets: Array<{
    assetId: string;
    pathOrUrl: string;
    altText: string;
    position: number;
    isHero: boolean;
  }>;
}

export interface MarketplaceDraftResult {
  marketplace: MarketplaceKey;
  externalId: string;
  externalUrl?: string | null;
  state: "draft";
  createdAt: string;
  warnings: string[];
}

export interface MarketplaceAdapter {
  key: MarketplaceKey;
  validate(input: MarketplaceDraftInput): Promise<string[]>;
  createDraft(input: MarketplaceDraftInput): Promise<MarketplaceDraftResult>;
}
```

Exact TypeScript naming may be adjusted during implementation, but the responsibilities and boundaries must remain.

## Preconditions

An adapter may create a draft only when:

- the Listing Package exists and is human-reviewed,
- the requested Listing Package version is current,
- the Commerce Asset Package is human-approved,
- at least one approved hero image exists when the destination requires images,
- required destination credentials are configured,
- destination-specific required fields pass validation.

## Adapter responsibilities

- Translate the canonical title, description, price, category, tags, and care details into destination fields.
- Upload only approved commerce derivatives.
- Preserve approved image order and hero position.
- Send approved alt text where the marketplace supports it.
- Create a **draft**, not a publicly active listing, during the first release.
- Return the external draft identifier and URL.
- Return understandable warnings for destination limitations or truncation.

## Core application responsibilities

- Source identity and SKU ownership.
- Listing generation, review, editing, and versioning.
- Asset processing, ordering, alt text, and approval.
- Destination selection.
- Human authorization to create a draft.
- Persistence of marketplace relationships and draft status.

## Failure rules

- Never mark a destination draft as created until the marketplace confirms it.
- Retrying after a timeout must not intentionally create duplicates; use an idempotency strategy where supported and a local operation record where not.
- Never silently truncate content. Show destination-specific warnings before or after draft creation.
- Never fall back to raw photos when approved assets are missing.
- Credentials, request bodies, prompts, and raw secret-bearing responses must not enter ordinary logs.

## First implementation

Shopify is the first automated adapter.

Version 1 success means:

1. The user selects Shopify from an approved product.
2. GM Commerce validates readiness.
3. GM Commerce creates a Shopify **draft** product/listing with approved content and images.
4. The returned Shopify draft link is stored and displayed.
5. The user reviews and activates it in Shopify manually.

Etsy follows only after the Shopify draft path works end to end.
