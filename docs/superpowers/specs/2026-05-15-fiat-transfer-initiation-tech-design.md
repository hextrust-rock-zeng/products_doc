# Fiat Transfer Initiation — Tech Design

**Date:** 2026-05-15
**Status:** Draft
**Author:** rock.zeng
**Related spec:** [2026-05-12-fiat-initiation-on-settlement-design.md](./2026-05-12-fiat-initiation-on-settlement-design.md)

---

## 1. Architecture

### 1.1 Existing Bank Integration Layer

Three repos already form the bank integration stack:

| Repo | Role |
|---|---|
| `hexsafe-2-bank-gateway` | Central JRPC gateway; exposes `bank_InitiateTransfer`; routes to Zand or BCB adaptor by bank identifier |
| `hexsafe-2-zand-bank-adaptor` | Zand Bank API client; handles domestic + international transfers; publishes to `bank-deposit` / `bank-withdrawal` Kafka topics via webhooks |
| `hexsafe-2-bcb-bank-adaptor` | BCB Bank API client; exposes `bcb_bank_AuthorizePayment` (`POST https://api.bcb.group/v5/payments/authorise`); publishes to `bank-deposit` / `bank-withdrawal` Kafka topics via webhooks |

`htm-htm-settlement-engine` is already an authorised client of both adaptors.

### 1.2 Component Responsibilities for This Feature

```
HexAdmin UI
    │  JRPC (admin_Htm_* methods)
    ▼
hexsafe-2-hexadmin-api-gateway     ← routes admin_* prefix to hexadmin-api
    │  JRPC
    ▼
hexsafe-2-hexadmin-api             ← BFF; handles auth, validation, calls downstream
    │  JRPC (htm_hse client)
    ▼
htm-htm-settlement-engine          ← owns fiat leg state machine
    │  bank_InitiateTransfer (JRPC)
    ▼
hexsafe-2-bank-gateway             ← routes by bank identifier (Zand / BCB)
    │                    │
    ▼                    ▼
Zand adaptor         BCB adaptor
    │  webhook            │  webhook
    └──────────┬──────────┘
               ▼
    bank-deposit / bank-withdrawal (Kafka)
               │
               ▼
htm-htm-settlement-engine          ← consumes to update fiat leg status
```

- **`hexsafe-2-hexadmin-api`** is the BFF layer sitting between HexAdmin UI and all downstream services; all `admin_Htm_*` JRPC methods are registered here and forwarded to the settlement engine via the `htm_hse` client package
- **`htm-htm-settlement-engine`** owns the fiat leg state machine; calls `bank_InitiateTransfer` for outgoing transfers; consumes `bank-deposit` / `bank-withdrawal` Kafka events to track status; integrates with `custody-authz-service` for maker-checker approval
- **`hexsafe-2-bank-gateway`** needs BCB routing added (currently only Zand is wired in); routing decision (Fedwire vs SWIFT+intermediary) is determined here based on beneficiary bank account data
- **`hexsafe-2-bcb-bank-adaptor`** — the BCB API `intermediary_bank` object supports both `bic` and `routing_number`; for USD SWIFT transfers routed via Standard Chartered, pass Standard Chartered's Fedwire/ABA number in `routing_number`
- **`HexAdmin UI`** adds the initiation screens, Fiat Transfers sub-tab under Pending Requests, and incoming transaction matching UI

### 1.3 Incoming Transaction Detection

Both Zand and BCB push webhooks for **all** incoming transactions on Hex's settlement accounts — not only transactions initiated through Hex's API. The adaptors publish these to the `bank-deposit` Kafka topic. The settlement engine consumes this topic to detect and store incoming transactions. **No polling job is needed.**

### 1.4 Fiat Leg State Machine

Fiat legs extend the existing `SettlementStatus` enum in `htm-htm-settlement-engine/internal/constant/status.go` with minimal new states.

**New states to add:**

| New state | Used by |
|---|---|
| `INCOMING_CLIENT_LEG_FIAT_PENDING_APPROVAL` | CL2: awaiting checker approval before bank transfer |
| `OUTGOING_LP_LEG_FIAT_PENDING_APPROVAL` | ML1: awaiting checker approval before bank transfer |
| `FIAT_TRANSFER_FAILED` | CL2, ML1, UC5: terminal failure after retries exhausted |

**State transitions:**

*CL2 — Outgoing fiat (Hex pays client):*
```
UNSETTLED
  → INCOMING_CLIENT_LEG_FIAT_PENDING_APPROVAL   (maker submits)
  → INCOMING_CLIENT_LEG_INITIATED               (reuse: bank transfer submitted after approval)
  → INCOMING_CLIENT_LEG_SETTLED                 (reuse: bank confirms via webhook)
  → FIAT_TRANSFER_FAILED                        (new: all retries exhausted)
  → UNSETTLED                                   (on checker rejection)
```

*ML1 — Outgoing fiat (Hex pays LP):*
```
UNSETTLED
  → OUTGOING_LP_LEG_FIAT_PENDING_APPROVAL       (maker submits)
  → OUTGOING_LP_LEG_INITIATED                   (reuse: bank transfer submitted after approval)
  → OUTGOING_LP_LEG_SETTLED                     (reuse: bank confirms via webhook)
  → FIAT_TRANSFER_FAILED                        (new: all retries exhausted)
  → UNSETTLED                                   (on checker rejection)
```

*CL1 — Incoming fiat (client pays Hex, ops confirms):*
```
UNSETTLED
  → OUTGOING_CLIENT_LEG_SETTLED                 (reuse: ops selects matching bank-deposit transactions)
```

*ML2 — Incoming fiat (LP pays Hex, ops confirms):*
```
UNSETTLED
  → INCOMING_LP_LEG_SETTLED                     (reuse: ops selects matching bank-deposit transactions)
```

---

## 2. Sequence Diagram — Fiat Initiation Flow (UC1 / UC2 / UC5)

> **Open item:** How does the front-end know whether to enable the Initiate Fiat Settlement button? The settlement engine should return an eligibility flag (e.g. `isFiatTransferAvailable`) per trade. Needs confirmation from Dmitry and Hao.

```mermaid
sequenceDiagram
    actor Maker as Operator Maker
    actor Checker as Operator Checker
    participant UI as HexAdmin UI
    participant API as hexadmin-api
    participant SE as htm-settlement-engine
    participant CA as custody-authz-service
    participant Mobile as HexSafe Mobile
    participant GW as bank-gateway
    participant Adaptor as Zand or BCB Adaptor
    participant Bank as Zand or BCB Bank

    Maker->>UI: Open CL2 / ML1 / Internal Transfer screen
    UI->>API: JRPC admin_Htm_GetTradeSettlementInfo
    API->>SE: Fetch trade and account data
    SE-->>API: Trade details, eligibility flag isFiatTransferAvailable
    API-->>UI: Trade details, eligibility flag isFiatTransferAvailable
    UI-->>Maker: Show Initiate Fiat Settlement if eligible, or ineligibility reason

    Maker->>UI: Click Initiate Fiat Settlement
    UI-->>Maker: Review screen — From, To, amount, transfer method

    Maker->>UI: Submit
    UI->>API: JRPC admin_Htm_InitiateFiatTransfer(tradeId, fromAccount, toAccount, amount, currency, method)
    API->>SE: InitiateFiatTransfer(tradeId, fromAccount, toAccount, amount, currency, method)
    SE->>SE: Validate and set status to PENDING_APPROVAL
    SE->>CA: CreateFiatTransferApproval(tradeId, transferDetails)
    CA-->>SE: approvalRequestId
    SE-->>API: OK, status Pending Approval
    API-->>UI: OK, status Pending Approval
    UI-->>Maker: Awaiting checker approval

    Checker->>UI: Open Pending Requests, Fiat Transfers sub-tab
    UI->>API: JRPC admin_Htm_ListPendingFiatTransfers
    API->>CA: ListPendingFiatTransfers
    CA-->>API: Pending transfer details
    API-->>UI: Pending transfer details
    UI-->>Checker: Review transfer details

    Checker->>UI: Click Approve
    UI->>API: JRPC admin_Htm_ApproveFiatTransfer(approvalRequestId)
    API->>CA: ApproveFiatTransfer(approvalRequestId)
    CA->>Mobile: Send mobile approval request to checker
    Note over Mobile: Fiat Transfer Initiation template<br/>Trade ID or Transfer Ref, Amount<br/>From and To account details<br/>Initiated by maker name and timestamp

    Checker->>Mobile: Approve on HexSafe mobile app
    Mobile->>CA: Approval confirmed
    CA->>SE: Kafka: quorum-approval-for-fiat-transfer APPROVED

    SE->>SE: Set status to INITIATED
    SE->>GW: bank_InitiateTransfer(bankId, fromAccount, toAccount, amount, method, intermediary)
    GW->>Adaptor: Route to Zand or BCB adaptor
    Adaptor->>Bank: POST transfer to bank API
    Bank-->>Adaptor: Transfer accepted, bank reference ID
    Adaptor-->>GW: OK
    GW-->>SE: OK, bankReferenceId
    SE->>SE: Record bankReferenceId and audit log

    Bank->>Adaptor: Webhook — transfer completed, bank tx ID and value date
    Adaptor->>Adaptor: Publish to bank-withdrawal Kafka topic
    SE->>SE: Consume bank-withdrawal event
    SE->>SE: Set status to FIAT_SETTLED, record bank tx ID and value date

    Note over CA,SE: On checker rejection in HexAdmin
    CA->>SE: Kafka: quorum-approval-for-fiat-transfer REJECTED
    SE->>SE: Set status to UNSETTLED
    SE->>SE: Send Slack notification to ops and trading team

    Note over SE: On bank API failure after approval
    SE->>SE: Auto-retry up to 3 times with exponential backoff
    SE->>SE: If all retries fail, set status to FIAT_TRANSFER_FAILED
    SE->>SE: Send Slack notification to ops and trading team
```

---

## 3. Sequence Diagram — Incoming Fiat Transaction Confirmation Flow (UC3 / UC4)

```mermaid
sequenceDiagram
    actor Client as Client or LP
    participant Bank as Zand or BCB Bank
    participant Adaptor as Zand or BCB Adaptor
    participant SE as htm-settlement-engine
    actor Operator as Operator
    participant UI as HexAdmin UI

    Client->>Bank: Initiate fiat transfer to Hex settlement account

    Bank->>Adaptor: Webhook — incoming transaction event<br/>bankRef, amount, currency, valueDate, senderName, senderAccount
    Adaptor->>Adaptor: Publish to bank-deposit Kafka topic

    SE->>SE: Consume bank-deposit event
    SE->>SE: Store incoming transaction<br/>bankRef, amount, currency, valueDate, sender, usedAmount=0

    Operator->>UI: Open CL1 or ML2 settlement screen
    UI->>SE: List pending trades and unmatched incoming transactions filtered by currency
    SE-->>UI: Trades and candidate transactions

    UI-->>Operator: Inline transaction list per trade, filtered by currency and amount range

    Operator->>UI: Select transactions for a trade and click Confirm Settlement
    UI->>SE: ConfirmIncomingFiatSettlement(tradeId, leg, selectedTxIds)

    SE->>SE: Validate sum of selected amounts is >= expected trade amount
    alt Sum less than expected amount
        SE-->>UI: Error — insufficient amount, wait for more transactions
    else Sum >= expected amount
        SE->>SE: Allocate smallest-to-largest<br/>Record usedAmount per transaction<br/>Remaining unused balance stays available
        SE->>SE: Set trade leg status to FIAT_SETTLED<br/>Record bankRefs, amounts, valueDates
        SE-->>UI: OK
        UI-->>Operator: Trade marked as Fiat Settled
    end

    Note over SE: No maker-checker required<br/>Incoming confirmation is a record action<br/>not a fund movement
```

---

*Dev effort, estimates, and development plan to be added after architecture review.*
