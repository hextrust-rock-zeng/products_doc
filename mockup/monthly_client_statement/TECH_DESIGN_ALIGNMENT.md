# Statement Service — Tech Design Alignment

**Epic:** [feature-version-tracker#1998](https://github.com/hextrust-internal/feature-version-tracker/issues/1998)
**PRD:** [Monthly Client Statement Application — PRD v1](https://app.notion.com/p/3712d315558981f7b447d4c53d2ccfe0)
**Reviewers:** William Cheung (approver) · Soujanyo Ray Chaudhuri (custody data)
**Author:** Rock Zeng — Product
**Status:** Draft for design session

---

## What we're building (30-second version)

A backend service that, at 00:00 UTC on the 1st of each month, generates one
PDF statement per `{Client, Service, Period}` (v1: Custody + OTC Trading),
runs pre-creation checks, auto-publishes to the client-facing Reporting Hub
(or holds in Pending Approval for opt-out-list clients), and keeps an
append-only audit trail. Detailed view only — Simplified was dropped.

Reference output (final design, stakeholder-reviewed):
[sample-statements/](https://github.com/hextrust-rock-zeng/products_doc/tree/feature/monthly-statement-mockup/mockup/monthly_client_statement/sample-statements)

---

## Decision points for this session

### D1. Standalone service vs. extend fee-calculation module

The HexAdmin mockup places Client Statements under **Fee Calculation** in the
nav. Repos `fee-calculation-engine` / `fee-calculation-processor` already
exist and run a monthly cron over per-client custody data — structurally the
closest thing to what we need.

| Option | For | Against |
|---|---|---|
| **A. New `statement-service`** | Clean ownership, no coupling to fee logic, free choice of stack | Another service to operate; duplicate cron/scheduling machinery |
| **B. Extend fee-calculation-*** | Reuses monthly cron + per-client iteration + AUC data pulls it already does | Couples statement lifecycle to fee releases; who owns that module today? |

**Need from William:** who owns fee-calculation today, and his call on A vs B.

### D2. Historical balance snapshots — does the data exist?

The statement needs **opening and closing balances per asset per vault** at
exact period boundaries (23:59:59 UTC month-end), one month after the fact.

**Need from Soujanyo:**
- Can `balance-service` reproduce a point-in-time balance per vault/asset
  retroactively, or does it only hold current state?
- If snapshot-based: what's the snapshot cadence and retention?
- If event-sourced: what's the cost of replaying a month for ~all clients?
- Same question for **staked / vesting / pending-TR** sub-balances — the
  statement splits these out (see sample PDF, Section 2 Part I).

This is the single biggest risk to the timeline. If point-in-time data
doesn't exist, we need a snapshot job running **before** the statement
service can ever produce a correct statement — that becomes ticket #2.

### D3. Source inventory per statement section

| Statement section | Likely source | Confirm |
|---|---|---|
| Opening/closing balances per asset/vault | balance-service | Soujanyo |
| Transaction history per vault | custody tx store (which service?) | Soujanyo |
| Pending transactions (TR + quorum) | Travel Rule status + maker-checker? | both |
| Staking position + claimable rewards | kiln-adaptor / staking service? | William |
| NFT balances | which service owns NFT positions? | William |
| OTC trade list + settlement movements | htm-settlement-engine | William |
| USD prices @ period close | asset-evaluator | William |
| Fee scheme + fee amounts | fee-calculation-engine | William |
| Client/Enterprise/Relationship metadata | enterprise-account-master? | William |

### D4. PDF renderer

A working Python/ReportLab prototype produced the sample PDFs (single source
of truth for layout). Options:

| Option | For | Against |
|---|---|---|
| **A. Port to Go** (service stack) | One language, one deploy unit | Go PDF libs are weaker; full re-implementation of ~2,900-line layout |
| **B. Keep Python renderer as sidecar** | Prototype IS the spec; zero re-implementation drift | Second runtime in the service |
| **C. HTML → headless-Chrome PDF** | Easier templating, designers can touch it | New rendering pipeline; pixel fidelity vs samples needs re-validation |

Prototype source: `consolidated_statement.py` (can be moved into the service
repo whichever way we go).

### D5. State machine + idempotency

```
DRAFT ──► PENDING_APPROVAL ──► PUBLISHED ──► RE_ISSUED ──► PUBLISHED
   └─────────────────────► PUBLISHED (auto-publish default)
   └─► FAILED (pre-creation check; no PDF, Slack alert)
```

- Cron re-runs must be idempotent per `{client, service, period}`.
- RE_ISSUED = silent replace (audit log only, no client notification).
- Confirm: event-driven (queue) vs. batch-loop, and which queue tech is
  standard in the HexSafe stack.

### D6. Non-functional

- **Storage:** S3-compatible, path `statements/{client}/{service}/{YYYY-MM}/`,
  indefinite retention while active, 7y post-offboarding. Bucket policy owner?
- **Volume estimate:** clients × 2 services × 12 months — trivial; design for
  correctness over throughput.
- **Observability:** per-state-transition metrics; failed-check alerting to
  Slack (Production Support + Customer Support channels).
- **Access control:** read API consumed by hexadmin-api (BFF) — confirm
  gateway pattern matches existing HexAdmin modules.

---

## Out of scope for the session

- UI implementation (HexAdmin queue + Reporting Hub) — mockups final,
  separate frontend tickets.
- Notification templates — wording owned by Customer Support (PRD §13).
- Loans (v2), CSV export (v2), Simplified view (dropped).

## Pre-reading

1. PRD §8 (pipeline), §11 (retention) — 5 min
2. `Custody_Detailed_sample.pdf` — 3 min, this is the contract
3. This doc — 5 min
