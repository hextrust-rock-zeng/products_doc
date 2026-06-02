# Monthly Client Statement — UI mockup & sample statements

This folder accompanies the **Monthly Client Statement Application** PRD
review.  It contains:

| File | Purpose |
|---|---|
| `statements_mockup.html` | Interactive HTML mockup of the new **Monthly Statements** screen inside the new **Reporting Hub** in HexSafe. Reporting Hub sits in the avatar dropdown menu (between *Settings* and *Help Center*) and opens as a **full-page takeover** — no left sidebar, no per-Enterprise context. A *Back to Custody Wallet* button in the top bar returns the user to the main HexSafe app. Single file, no external dependencies — open it in any modern browser. |
| `sample-statements/Custody_Simplified_sample.{pdf,docx}` | Custody — Simplified view. |
| `sample-statements/Custody_Detailed_sample.{pdf,docx}` | Custody — Detailed view. |
| `sample-statements/OTCTrading_Simplified_sample.{pdf,docx}` | OTC Trading — Simplified view. |
| `sample-statements/OTCTrading_Detailed_sample.{pdf,docx}` | OTC Trading — Detailed view. |
| `sample-statements/Loans_Simplified_sample.{pdf,docx}` | Loans — Simplified view (positions + period interest). |
| `sample-statements/Loans_Detailed_sample.{pdf,docx}` | Loans — Detailed view (positions + period interest + margin events). |

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

- New **Reporting Hub** entry in the avatar dropdown menu (between
  *Settings* and *Help Center*). The dropdown is rendered open in the
  mockup with *Reporting Hub* highlighted to make the placement explicit.
- Breadcrumb under the topbar — *Avatar menu › Reporting Hub › Monthly
  Statements* — so reviewers can see the access path.
- Reporting Hub in v1 hosts a single page — **Monthly Statements**.
- Reporting Hub opens **full-page** — no left sidebar, no enterprise /
  currency selector visible. The user enters from the avatar menu and
  returns via the *Back to Custody Wallet* button in the top bar.
- The page shows all monthly statements the signed-in user has access
  to (across the entities in their Relationship). Enterprise scoping
  / row attribution is intentionally hidden from the v1 UI.
- Two multi-select filters in one row: **Year** and **Service**. Each
  opens a dropdown panel with checkboxes.
- Monthly groups, ordered newest → oldest. Latest month is expanded by
  default; older months are collapsed (click month header to expand).
- Within each month, statements are sub-grouped by service type:
  **Custody → OTC Trading → Loans**. One row per service per month
  (since the page is already entity-scoped).
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
Top bar       [ ← Back to Custody Wallet ]                  [ avatar ▾ ]
Reporting Hub ▸ Monthly Statements
├── Filter bar    [ Year ▾ ] [ Service ▾ ]
├── Bulk action bar (only when ≥1 row selected)
└── Month groups (newest first)
    └── April 2026                             [3 statements]  [☐ Select all]
        ├── ■ CUSTODY
        │   ☐  Custody Statement — Apr 2026                   ⇩
        ├── ■ OTC TRADING
        │   ☐  OTC Trading Statement — Apr 2026               ⇩
        └── ■ LOANS
            ☐  Loan Statement — Apr 2026                      ⇩
```
