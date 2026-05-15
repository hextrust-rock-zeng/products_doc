# Fiat Transfer Initiation — Requirements Spec

**Date:** 2026-05-12
**Status:** Under Dev Review
**Author:** rock.zeng

---

## 1. Background

OTC trade settlement at Hex Trust involves both crypto legs (handled automatically by the settlement engine via on-chain sweeps) and fiat legs. Today, all fiat legs are handled manually:

1. Operator identifies a trade needing fiat settlement in HexAdmin
2. Operator logs into the bank console (Zand / BCB) separately to initiate or check the transfer
3. Operator returns to HexAdmin and manually fills in bank transaction details (txId, amount, value date) to confirm the settlement

This process has three problems:
- Operators context-switch between two systems for every fiat settlement
- There is no audit trail of who initiated the actual bank transfer
- No approval gate exists before funds move
- Incoming fiat transfers have no way to be automatically detected or matched to trades

---

## 2. Scope

Hex Trust has existing API integrations with **Zand Bank** and **BCB Bank**, both supporting:
- **Outgoing payment initiation** (push)
- **Account statement retrieval** (pull — incoming transaction detection)

This feature adds fiat transfer initiation and incoming transaction detection directly into HexAdmin, removing the need for operators to use bank consoles for routine settlement operations.

**Out of scope for this release:**
- House-to-client free transfers (deferred — requires separate risk/compliance sign-off)
- Banks not yet integrated with Hex Trust's banking API

---

## 3. Use Cases

**Delivery priority order:** UC1 (CL2) > UC2 (ML1) > UC5 (House-to-House) > UC3 (CL1) = UC4 (ML2)

---

### 3.1 UC1 — Outgoing Fiat: Hex Pays Client (CL2)

**Trigger:** A trade has completed the crypto leg and is waiting for fiat payout to the client (e.g., client sold crypto and is owed fiat).

#### 3.1.1 API Eligibility

The "Initiate Fiat Settlement" action is available when the bank API can handle the specific transfer. The existing manual confirmation flow (fill in txId / amount / value date) is **always displayed** on every trade regardless of eligibility.

**Zand Bank** — eligible only if ALL three conditions are met:
- Settlement currency = AED
- Client's bank account is UAE-based
- Client's bank account has an IBAN configured

**BCB Bank** (US-based account) — routing decision based on client bank account data:

| Client bank account data | Transfer method | API eligible |
|---|---|---|
| Routing number present, no swift code | Fedwire / routing number | Yes |
| Routing number present, swift code is US | Fedwire / routing number | Yes |
| Routing number present, swift code is non-US | SWIFT + Standard Chartered intermediary | Yes |
| No routing number, swift code is non-US | SWIFT + Standard Chartered intermediary | Yes |
| No routing number, swift code is US | Manual — missing data | No |
| No routing number, no swift code | Manual — missing data | No |

> When routing via intermediary bank, always use **Standard Chartered Bank** as the intermediary. The BCB API `intermediary_bank` object supports both a `bic` field and a `routing_number` field (valid for USD SWIFT transfers only). Pass Standard Chartered's **Fedwire/ABA routing number** in the `routing_number` field — since BCB is a US bank, the first hop is a domestic Fedwire transfer to Standard Chartered's US correspondent account. Ignore any intermediary bank accounts configured on the client's bank account record.

**All other banks:** always manual flow only.

#### 3.1.2 Flow

1. Operator opens the Client Leg 2 settlement screen in HexAdmin
   - **All trades** always show the existing manual confirmation action
   - Eligible trades additionally show **"Initiate Fiat Settlement"**
   - For API-ineligible trades, the reason is displayed (e.g., "IBAN not configured", "Currency not supported by Zand", "Missing routing number and SWIFT code")
2. Operator selects a trade and clicks "Initiate Fiat Settlement"
3. HexAdmin shows a review screen:
   - From: Hex's client settlement bank account (Zand or BCB)
   - To: client's registered bank account
   - Transfer method: derived automatically per eligibility rules above
   - Fields used for transfer (IBAN, routing number, SWIFT, intermediary) — shown on screen (optional/nice-to-have display)
   - Amount, currency, payment reference (derived from trade ID)
4. Operator (maker) submits → trade status moves to **Pending Approval**; request appears in the **Fiat Transfers** sub-tab under the Pending Requests tab
5. Checker reviews the transfer details in the **Fiat Transfers** sub-tab in HexAdmin, then confirms the approval on the **HexSafe mobile app** (a new mobile notification template for fiat transfers is required)
   - Maker cannot approve their own submission
6. On approval: system calls the Zand or BCB API to initiate the transfer → status moves to **Bank Transfer Initiated**
7. Bank confirms processing (via callback or polling) → status moves to **Fiat Settled**; bank transaction ID and timestamp are recorded automatically
8. On rejection: status reverts to Unsettled; Slack notification sent to ops team and trading team

#### 3.1.3 Failure handling

If the bank API call fails after checker approval:
1. System automatically retries up to **3 times with exponential backoff**
2. If all retries exhausted → status moves to **Transfer Failed**; Slack notification sent to ops team and trading team
3. Ops can trigger a **manual retry** from HexAdmin — this requires a **fresh maker-checker cycle** (same flow as initiating from scratch)
4. If the issue is unresolvable, ops falls back to the **existing manual Fiat Confirmation** flow

> The existing manual confirmation flow (fill in txId / amount / value date) is **always available** on every trade regardless of whether API initiation was attempted, is in-flight, or has failed.

> **Slack notifications** are sent to both the **ops team** and the **trading team** on: rejection by checker, transfer failure after retries exhausted, or any other terminal failure state. Notifications are not sent on transient failures where retry is still pending.

---

### 3.2 UC2 — Outgoing Fiat: Hex Pays LP (ML1)

Identical to UC1 in flow, eligibility rules, approval flow, failure handling, and notification behaviour, with the following differences:

- **From:** Hex's **market** settlement bank account (may differ from the client settlement account)
- **To:** LP's registered bank account
- Eligibility and BCB routing rules are evaluated against the LP's bank account data
- LP bank accounts are stored in the same structure as client bank accounts
- The existing manual confirmation flow is always displayed alongside the initiation option

---

### 3.3 UC3 — Incoming Fiat: Client Pays Hex (CL1)

**Priority: lower than UC5, UC1, UC2.**

**Trigger:** A trade is awaiting receipt of fiat from the client (e.g., client buying crypto with fiat).

#### 3.3.1 Incoming transaction detection

Both Zand and BCB push webhooks for **all** incoming transactions on Hex's settlement accounts (not only ones we initiated). The Zand and BCB adaptors consume these webhooks and publish to the `bank-deposit` Kafka topic. The settlement engine consumes this topic to detect and store incoming transactions — **no polling job is needed**.

Each stored transaction records: bank reference, amount, currency, value date, sender name, sender account number.

#### 3.3.2 Flow

1. Operator opens the Client Leg 1 settlement screen in HexAdmin
2. For each trade pending fiat receipt, an inline section shows unmatched incoming bank transactions filtered by currency and approximate amount range
3. Operator selects one or more transactions whose combined amount covers the trade's expected amount and clicks "Confirm Settlement"
   - The system validates that the sum of selected transactions is **greater than or equal to** the expected trade amount
   - If the total is less than required, confirmation is blocked — operator waits for remaining transactions to arrive
   - It is all-or-nothing: no partial settlement state
4. On confirmation: trade leg moves to **Fiat Settled**; bank transaction references, amounts, and value dates are recorded

#### 3.3.3 Transaction amount tracking

A single incoming bank transaction may cover more than one trade (or cover a trade with surplus). The system must track how much of each transaction has been used:

- When a transaction is selected to confirm a trade, the **amount used** for that trade is recorded against that transaction
- Any **remaining unused balance** on a partially-consumed transaction stays available for matching to other trades
- When multiple transactions are selected for one trade and their combined total exceeds the trade amount, the system allocates from **smallest to largest** — this ensures at most one transaction has a remaining unused balance, keeping the tracking simple
- A transaction with a fully consumed balance is no longer shown in the matching list

#### 3.3.4 Unmatched transactions *(lower priority)*

Incoming transactions not matched to any trade remain visible in a dedicated **Unmatched Transactions** view for ops to investigate.

**No maker-checker required** for incoming confirmation — this is a record action, not a fund movement.

---

### 3.4 UC4 — Incoming Fiat: LP Pays Hex (ML2)

**Priority: same as UC3 (lower than UC5, UC1, UC2).**

Identical to UC3, with the following differences:

- Webhook events target Hex's **market** settlement bank accounts
- Inline transaction list appears on the **Market Leg 2** settlement screen
- Matched against LP's expected payment instead of client's

---

### 3.5 UC5 — House-to-House Bank Transfer

**Priority: after UC1 and UC2, before UC3 and UC4.**

**Context:** Ops sometimes needs to move funds between Hex's own bank accounts (e.g., from client settlement account to market settlement account to cover a position).

#### 3.5.1 Account source

Available house bank accounts are sourced from bank accounts configured on **House-type trading clients** (broker entities) in the Trading Counterparty system — the same place client bank accounts are managed today, filtered to House type.

#### 3.5.2 Flow

1. Operator opens the **Internal Bank Transfer** sub-tab under the **Trade Settlement** screen in HexAdmin
2. Operator selects:
   - **From:** one of Hex's configured house bank accounts (Zand or BCB)
   - **To:** another of Hex's configured house bank accounts
   - Amount, currency, notes / reference
3. Zand/BCB eligibility and BCB routing rules apply — transfer method is derived automatically
4. Maker submits → **Pending Approval**; request appears in the **Fiat Transfers** sub-tab under the Pending Requests tab
5. Checker reviews the transfer details in the **Fiat Transfers** sub-tab in HexAdmin, then confirms the approval on the **HexSafe mobile app** (same flow as UC1 — new mobile fiat transfer template applies here too)
   - Maker cannot approve their own submission
6. On approval: bank API called → transfer initiated; bank transaction ID recorded
7. On rejection: transfer reverts; Slack notification sent to ops team and trading team
8. Failure handling: same as UC1 (auto-retry → Transfer Failed → Slack notification to ops and trading team → fresh maker-checker cycle for retry; fallback to manual Fiat Confirmation if unresolvable)

#### 3.5.3 Risk controls

- Only pre-configured Hex house accounts are selectable as source and destination — no free-form account entry
- Currency must match the From account's supported currency (e.g., Zand only supports AED)
- Every house-to-house transfer is included in the full audit trail (see Section 4.2)

---

## 4. Architecture

### 4.1 Existing Bank Integration Layer

Three repos already form the bank integration stack:

| Repo | Role |
|---|---|
| `hexsafe-2-bank-gateway` | Central JRPC gateway; exposes `bank_InitiateTransfer`; routes to Zand or BCB adaptor |
| `hexsafe-2-zand-bank-adaptor` | Zand Bank API client; handles domestic + international transfers; publishes to `bank-deposit` / `bank-withdrawal` Kafka topics via webhooks |
| `hexsafe-2-bcb-bank-adaptor` | BCB Bank API client; exposes `bcb_bank_AuthorizePayment`; publishes to `bank-deposit` / `bank-withdrawal` Kafka topics via webhooks |

The `htm-settlement-engine` is already an allowed client of both adaptors.

### 4.2 Component Responsibilities for This Feature

```
HexAdmin UI
    ↓ JRPC
htm-settlement-engine       ← owns fiat leg state machine
    ↓ bank_InitiateTransfer (JRPC)
hexsafe-2-bank-gateway      ← routes by bank identifier (Zand / BCB)
    ↓                            ↓
Zand adaptor              BCB adaptor
    ↓ webhook                    ↓ webhook
         bank-deposit / bank-withdrawal Kafka topics
                    ↓
          htm-settlement-engine  ← consumes to update fiat leg status
```

- **`htm-settlement-engine`** owns the fiat leg state machine, calls `bank_InitiateTransfer` for outgoing transfers, and consumes `bank-deposit` / `bank-withdrawal` Kafka events to track status
- **`hexsafe-2-bank-gateway`** requires BCB routing to be added (currently only Zand is wired in); BCB API endpoint is `POST https://api.bcb.group/v5/payments/authorise`
- **`hexsafe-2-bcb-bank-adaptor`** intermediary bank field confirmed: the BCB API `intermediary_bank` object supports both `bic` and `routing_number`; pass Standard Chartered's Fedwire/ABA number in `routing_number`
- **HexAdmin UI** adds the initiation screens, Fiat Transfers sub-tab, and incoming transaction matching UI

### 4.3 Incoming Transaction Detection (UC3 / UC4)

Both Zand and BCB push webhooks for **all** incoming transactions on Hex's settlement accounts — not only ones initiated through our API. No polling job is needed. The settlement engine consumes the `bank-deposit` Kafka topic published by both adaptors.

### 4.4 Fiat Leg State Machine

Fiat legs extend the existing `SettlementStatus` enum in `htm-htm-settlement-engine/internal/constant/status.go` with minimal new states. Existing terminal states (`INCOMING_CLIENT_LEG_SETTLED`, `OUTGOING_LP_LEG_SETTLED`, etc.) are reused.

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
  → INCOMING_CLIENT_LEG_SETTLED                 (reuse: bank confirms)
  → FIAT_TRANSFER_FAILED                        (new: retries exhausted)
  → UNSETTLED                                   (on rejection or cancellation)
```

*ML1 — Outgoing fiat (Hex pays LP):*
```
UNSETTLED
  → OUTGOING_LP_LEG_FIAT_PENDING_APPROVAL       (maker submits)
  → OUTGOING_LP_LEG_INITIATED                   (reuse: bank transfer submitted after approval)
  → OUTGOING_LP_LEG_SETTLED                     (reuse: bank confirms)
  → FIAT_TRANSFER_FAILED                        (new: retries exhausted)
  → UNSETTLED                                   (on rejection or cancellation)
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

## 5. Cross-Cutting Requirements

### 5.1 Approval — Two-Step: HexAdmin Review + HexSafe Mobile Confirm

All outgoing transfer initiations (UC1, UC2, UC5) follow the same two-step approval flow:

1. **Maker** submits the transfer in HexAdmin → request appears in the **"Fiat Transfers"** sub-tab under the existing **Pending Requests** tab
2. **Checker** reviews the full transfer details (from/to accounts, amount, currency, transfer method) in the Fiat Transfers sub-tab in HexAdmin
3. **Checker** confirms the approval on the **HexSafe mobile app** using the existing **"Fiat Transfer Initiation"** template (see below)
4. Maker cannot approve their own submission

**Roles:** Both **Operator** and **Operator Manager** can act as maker or checker. The only constraint is that the same person cannot approve their own submission.

#### Mobile approval template

The existing **"Fiat Transfer Initiation"** template is reused. It is a multi-page swipeable card with a 5-minute countdown timer and Cancel / Confirm actions.

**Fields displayed for UC1 / UC2 (trade-linked transfers):**

| Field | Value |
|---|---|
| Trade ID | Trade ID (e.g., OTC-20251107-0089) |
| Ticker | Settlement currency |
| Amount | Settlement amount |
| Counterparty Name | Client or LP name |
| From Bank | Hex's settlement bank name |
| From Account Holder Name | Hex's account holder name |
| From Routing Number | Routing number (if applicable) |
| From Account Number / IBAN | Account number or IBAN |
| From SWIFT Code | SWIFT code (if applicable) |
| To Bank | Client's / LP's bank name |
| To Account Holder Name | Client's / LP's account holder name |
| To Routing Number | Routing number (if applicable) |
| To Account Number / IBAN | Account number or IBAN |
| To SWIFT Code | SWIFT code (if applicable) |
| Transaction ID | **Hidden / not shown** — transfer has not been submitted to the bank yet at approval time |
| Note | Payment reference derived from trade ID |
| Initiated by | Maker's name |
| Initiated at | Submission timestamp |

**Fields for UC5 (house-to-house transfers):** same template with these adjustments:
- Trade ID → replaced with **Transfer Reference** (internal reference ID)
- Counterparty Name → not shown (both accounts are Hex house accounts)

### 5.2 Audit Trail

Every state transition on a fiat transfer is logged:

| Event | Data captured |
|---|---|
| Initiated | Operator ID, timestamp, trade ID, leg, from/to accounts, amount, currency, transfer method |
| Approved / Rejected | Checker ID, timestamp |
| Bank API called | Timestamp, API request reference |
| Bank confirmed | Bank transaction ID, value date, timestamp |
| Failed | Failure reason, retry count |

Bank transaction IDs returned from Zand / BCB are stored and visible in the trade detail view.

### 5.3 Notifications

Slack notifications are sent to both the **ops team** and the **trading team** in the following events:

| Event | Condition |
|---|---|
| Transfer rejected by checker | Immediately on rejection |
| Transfer Failed | Only after all automatic retries are exhausted (not on transient retry failures) |
| Any other terminal failure | Immediately |

No email notifications in scope for this release.

### 5.4 Transfer Method Display (Optional / Nice-to-Have)

On the initiation review screen, display which fields will be used for the transfer so the operator can verify before submitting. For example:

- "Transfer via: SWIFT + Standard Chartered intermediary"
- Fields: IBAN, SWIFT code, intermediary Fedwire number
- Or: "Transfer via: Fedwire / routing number"
- Fields: Routing number, account number

---

## 6. Out of Scope / Deferred

| Item | Reason |
|---|---|
| House-to-client free transfer | Needs separate risk and compliance sign-off before scoping |
| Banks not yet integrated (non-Zand, non-BCB) | Manual flow remains for these |
| Email notifications | Not required for this release |
| Auto-matching of incoming transactions to trades | Ops manually selects matches; auto-match may be considered in a future release |
| Unmatched Transactions view (UC3/UC4) | Lower priority; to be scheduled after core incoming confirmation is delivered |
