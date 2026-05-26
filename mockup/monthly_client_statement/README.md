# Monthly Client Statement — UI mockup & sample statements

This folder accompanies the **Monthly Client Statement Application** PRD
review.  It contains:

| File | Purpose |
|---|---|
| `statements_mockup.html` | Interactive HTML mockup of the new **Statements** tab in HexSafe (under *Settings*).  Single file, no external dependencies — open it in any modern browser. |
| `sample-statements/SimplifiedStatement_sample.pdf` | Reference example of the Simplified-view monthly statement (PDF). |
| `sample-statements/SimplifiedStatement_sample.docx` | Same as above, in DOCX. |
| `sample-statements/DetailedStatement_sample.pdf` | Reference example of the Detailed-view monthly statement (PDF). |
| `sample-statements/DetailedStatement_sample.docx` | Same as above, in DOCX. |

## How to view

1. Download / clone the repo locally.
2. Double-click `statements_mockup.html` → opens in your default browser.
3. Use the *Mockup view* dropdown (top-right of the page) to switch between
   the three rendering states:
   - **Default** — typical landing view
   - **Bulk-selected** — three statements selected, bulk-download bar visible
   - **Empty** — current filters yield no statements

The HTML is self-contained (inline CSS + JS) — it works offline, no install
needed.

## What the mockup shows

- New **Statements** sub-tab placed at the end of the *Settings* tabs
  (right of *Transaction Policy*).
- Three multi-select filters in one row: **Year**, **Service**,
  **Enterprise**.  Each opens a dropdown panel with checkboxes.
- Monthly groups, ordered newest → oldest.  Latest month is expanded by
  default; older months are collapsed (click month header to expand).
- Within each month, statements are sub-grouped by service type:
  **Custody → OTC Trading → Loans**.  Each service block lists one row per
  Enterprise.
- Each row shows: month, publish date, format hint (PDF · CSV) and any
  pending-item badge (e.g. *2 unsettled trades*, *3 pending TR items*).
- Single per-row action: **Download**.
- Multi-select + bulk **Download as ZIP** bar appears once one or more
  rows are selected.

## Reference: sample statements

`sample-statements/` carries the actual PDFs and DOCX files for both views,
generated from the latest design.  These are what the client would
download when they click *Download* on a row.

Open these alongside the mockup to see how each row in the UI maps to the
final artefact the client receives.

## Open design items (still to resolve)

- Download format combinations (PDF only? PDF + CSV? Single combined PDF
  for multi-select vs. ZIP-of-PDFs?)
- Per-row "View" overlay — is in-browser PDF preview needed in v1 or is
  download-only sufficient?
- Pending-item badge behaviour — show / hide once the client has acted?
- Mobile / narrow-viewport layout (not in this mockup).

## Layout reference

```
Settings ▸ Statements
├── Filter bar    [ Year ▾ ] [ Service ▾ ] [ Enterprise ▾ ]
├── Bulk action bar (only when ≥1 row selected)
└── Month groups (newest first)
    └── April 2026                             [4 statements]  [☐ Select all]
        ├── ■ CUSTODY
        │   ☐  Custody Statement — HT Markets MENA FZE        ⇩
        │   ☐  Custody Statement — Meridian Capital Pte Ltd   ⇩
        ├── ■ OTC TRADING
        │   ☐  OTC Trading Statement — HT Markets MENA FZE    ⇩
        └── ■ LOANS
            ☐  Loan Statement — HT Markets MENA FZE           ⇩
```
