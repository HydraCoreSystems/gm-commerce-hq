# Copy-Ready Listing Output Specification

## Purpose

Provide an immediately usable sales output from the canonical Listing Package so Gathering Moss can use GM Commerce before automated marketplace publishing is complete.

Primary channels:

- Palm Street live sales,
- Overgrown Oasis,
- Whatnot,
- Facebook purges and auctions,
- manual Shopify or Etsy entry when needed,
- any future channel requiring copy and paste rather than an API.

## User outcome

Phil or Crystal opens an approved Listing Package and receives one clean output screen containing every field needed to create a listing manually, with one-click copying and a printable/downloadable document.

## Required fields

- SKU
- product display name
- proposed listing title
- short sales title
- full description
- sales summary
- price
- category
- tags/keywords
- care details when present
- quality warnings or missing information, clearly separated from customer-facing text
- approved hero image and ordered approved supporting images when the Commerce Asset Package is available
- alt text for each approved image when available
- generated/reviewed timestamps and package version for internal traceability

## Required interface

- One obvious page reached from the Review Queue after a package is reviewed.
- Large readable field cards.
- A **Copy** button beside every copyable field.
- A **Copy complete listing** action that places a clean, labeled text version on the clipboard.
- A **Print / Save as PDF** layout using the browser's native print flow.
- A downloadable `.docx` or equivalent editable document may follow, but browser print and clipboard output must not wait on it.
- No marketplace-specific rewriting in Version 1. This output reads from the canonical Listing Package.

## Output modes

### Field mode

Each field is isolated for fast copy/paste into a platform form.

### Complete listing mode

A clean plain-text block:

```text
SKU: ...
Title: ...
Price: ...
Category: ...

Description:
...

Care details:
...

Tags:
...
```

Omit empty optional fields rather than displaying `null`, placeholders, or AI language.

### Print mode

A professional internal Listing Sheet suitable for Crystal or Phil to reference during a live sale or purge. It must fit normal letter-size printing and remain readable on mobile.

## Data rules

- Read from the reviewed canonical Listing Package.
- Never regenerate or rewrite content merely to display it.
- Clearly distinguish internal warnings from customer-facing copy.
- Use the current approved Listing Package version.
- If no reviewed package exists, show the exact blocking step and link back to Review.
- Image fields appear only from approved commerce assets; raw originals are never presented as marketplace-ready assets.

## Acceptance criteria

- A real reviewed plant Listing Package can be opened as a Listing Sheet.
- Every required populated field can be copied independently.
- The complete listing can be copied in one action.
- The page prints cleanly to letter-size PDF.
- Empty fields are omitted cleanly.
- Internal quality notes cannot accidentally be copied as customer-facing description.
- Mobile use is practical for live sales.
- Existing intake, generation, review, and photo workflows remain unchanged.

## Out of scope

- Shopify or Etsy API calls,
- marketplace-specific title rewriting,
- inventory updates,
- sales synchronization,
- elaborate document-template design,
- email delivery.
