# Photo Pipeline Priority Handoff

## Decision

Photo preparation is now a required pre-publishing stage. Shopify and Etsy adapters must consume approved standardized derivatives, not raw source uploads.

## Why this was added now

GM Commerce has concentrated on intake and listing copy, but listing photos are equally important to sales. The platform must not reach marketplace publication with a polished description and inconsistent, unoptimized images.

## Sequencing

1. Finish and reconcile GMCOM-009 and GMCOM-010 reliability work.
2. Build the Version 1 photo preparation pipeline.
3. Verify it with real plant and non-plant photo sets.
4. Only then begin Shopify draft publishing.

## Authoritative scope

See `PHOTO_PIPELINE_SCOPE.md` and the 2026-08-02 photo decision in `DECISIONS.md`.
