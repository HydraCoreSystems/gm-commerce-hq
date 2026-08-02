# GM Commerce Photo Pipeline Scope

## Purpose

Define the Version 1 photo-preparation workflow required before GM Commerce creates a marketplace draft.

## Core principle

Raw uploads are source material. Approved, standardized derivatives are commerce assets.

GM Commerce must never silently overwrite originals or alter the product in a misleading way.

## Version 1 workflow

1. Discover supported image files in the SKU photo folder.
2. Preserve every original unchanged.
3. Inspect each image for:
   - orientation and rotation,
   - dimensions and aspect ratio,
   - blur or obvious focus problems,
   - severe under/overexposure,
   - duplicated or near-duplicated images,
   - background/clutter concerns,
   - missing scale or important product views,
   - likely hero-image suitability.
4. Present a guided review showing what will be changed and what requires human attention.
5. Create standardized derivatives in a separate output location.
6. Apply safe, repeatable transformations only:
   - correct EXIF rotation,
   - crop/resize to approved Gathering Moss standards,
   - modest exposure/white-balance/color correction,
   - sharpening and noise reduction when appropriate,
   - background cleanup only when it does not misrepresent the item,
   - file-format conversion and compression,
   - standardized filenames and ordering.
7. Allow the user to choose/reorder the hero image and supporting images.
8. Require human approval of the prepared set.
9. Persist an asset manifest containing originals, derivatives, order, dimensions, file sizes, processing status, warnings, and approval state.
10. Make only approved derivatives available to Shopify, Etsy, listing sheets, or other output adapters.

## Truthfulness guardrails

The system must not:

- add leaves, blooms, roots, nodes, texture, color, accessories, or product features,
- hide meaningful damage or condition,
- materially change plant or product color,
- create a false lifestyle setting and present it as a product photograph,
- replace the actual item with an AI-generated rendition,
- discard or overwrite the original files.

Any generative or object-removal operation must be explicitly identified, previewed, and approved. Version 1 should favor conventional deterministic editing over generative modification.

## Initial Gathering Moss derivative standard

These values must be configurable rather than buried in code.

- Preserve full-resolution originals.
- Produce a high-quality square marketplace derivative for general use.
- Produce an uncropped or minimally cropped detail derivative when square cropping would remove important product information.
- Use sRGB output.
- Prefer JPEG for ordinary product photographs and PNG only when transparency or lossless graphics are genuinely needed.
- Strip unnecessary metadata while retaining internal provenance in the asset manifest.
- Balance quality and file size without visible compression damage.

Exact pixel dimensions and compression targets should be finalized against current Shopify and Etsy requirements when their output adapters are implemented.

## Required GUI behavior

A user should see:

- which images were found,
- which are usable,
- which need attention,
- what changes are proposed,
- the proposed hero image,
- the final order,
- before/after previews,
- one obvious next action,
- a clear Approved for Listing state.

The interface must not expose image-processing jargon unless the user opens advanced details.

## Architecture direction

The photo pipeline should use a provider-neutral processing interface so deterministic local processing, cloud image services, and future AI vision/review services can be changed without rewriting the commerce workflow.

Suggested conceptual flow:

`Original Assets -> Inspection -> Processing Plan -> Derivative Generation -> Human Review -> Approved Asset Manifest -> Output Adapters`

## Out of scope for the first implementation

- automatic background generation,
- synthetic lifestyle scenes,
- replacing the photographed product,
- video processing,
- marketplace publishing itself,
- destructive editing of originals.
