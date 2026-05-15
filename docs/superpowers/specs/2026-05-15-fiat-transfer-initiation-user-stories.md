# Fiat Transfer Initiation — User Stories

**Date:** 2026-05-15
**Status:** Draft
**Author:** rock.zeng
**Related spec:** [2026-05-12-fiat-initiation-on-settlement-design.md](./2026-05-12-fiat-initiation-on-settlement-design.md)
**Related tech design:** [2026-05-15-fiat-transfer-initiation-tech-design.md](./2026-05-15-fiat-transfer-initiation-tech-design.md)

---

## Delivery Priority

**Phase 1 (UC1 + UC2):** US-01 → US-08  
**Phase 2 (UC5):** US-09 → US-11  
**Phase 3 (UC3 + UC4):** US-12 → US-16  
**Cross-cutting (all phases):** US-17, US-18

---

## Phase 1 — Outgoing Fiat: Hex Pays Client / LP (UC1 + UC2)

---

### US-01 — View Fiat Settlement Eligibility on CL2 / ML1 Trade

**As an** Operator,  
**I want to** see whether a CL2 or ML1 trade is eligible for API-based fiat settlement initiation,  
**so that** I know whether to use the new initiation flow or fall back to manual confirmation.

**Acceptance Criteria:**

- [ ] For each CL2 / ML1 trade, the settlement screen shows the "Initiate Fiat Settlement" button if the trade is eligible
- [ ] **Zand Bank** eligibility — all three conditions must be met:
  - Settlement currency = AED
  - Client / LP bank account is UAE-based
  - Client / LP bank account has an IBAN configured
- [ ] **BCB Bank** eligibility — based on client / LP bank account data:
  - Routing number present (with or without SWIFT, or with non-US SWIFT) → eligible
  - No routing number + non-US SWIFT → eligible (SWIFT + Standard Chartered intermediary)
  - No routing number + US SWIFT → ineligible
  - No routing number + no SWIFT → ineligible
- [ ] For ineligible trades, the reason is displayed inline (e.g. "IBAN not configured", "Currency not supported by Zand", "Missing routing number and SWIFT code")
- [ ] For all other banks: always ineligible; manual confirmation only
- [ ] The existing Manual Fiat Confirmation action is always visible regardless of eligibility or trade status
- [ ] ⚠️ **Open item:** Eligibility flag (`isFiatTransferAvailable`) to be returned by the settlement engine per trade — approach to be confirmed with Dmitry / Hao

---

### US-02 — Initiate Outgoing Fiat Settlement: Hex Pays Client (CL2)

**As an** Operator (maker),  
**I want to** initiate a fiat settlement for a CL2 trade directly from HexAdmin,  
**so that** I don't need to log into the bank console to start the transfer.

**Acceptance Criteria:**

- [ ] Clicking "Initiate Fiat Settlement" opens a review screen showing:
  - From: Hex's client settlement bank account (bank name, account number)
  - To: Client's registered bank account (bank name, account number)
  - Amount and currency
  - Payment reference (derived from trade ID)
  - Transfer method (Fedwire / IBAN / SWIFT + Standard Chartered intermediary)
- [ ] Operator (maker) submits the review screen
- [ ] Trade status moves to **Pending Approval**
- [ ] The submission is recorded in the audit trail: operator ID, timestamp, trade ID, leg (CL2), from/to accounts, amount, currency, transfer method
- [ ] Maker cannot approve their own submission
- [ ] The existing Manual Fiat Confirmation action remains visible on the trade after submission

---

### US-03 — Initiate Outgoing Fiat Settlement: Hex Pays LP (ML1)

**As an** Operator (maker),  
**I want to** initiate a fiat settlement for an ML1 trade directly from HexAdmin,  
**so that** I don't need to log into the bank console to pay the LP.

**Acceptance Criteria:**

- [ ] Same flow and acceptance criteria as US-02, with the following differences:
  - From: Hex's **market** settlement bank account (may differ from client settlement account)
  - To: LP's registered bank account
  - Eligibility and BCB routing rules evaluated against the LP's bank account data
  - Leg recorded as ML1 in audit trail

---

### US-04 — Review Pending Fiat Transfers as Checker

**As an** Operator (checker),  
**I want to** see all pending fiat transfer requests in a dedicated sub-tab under Pending Requests,  
**so that** I can review and action them without leaving HexAdmin.

**Acceptance Criteria:**

- [ ] A "Fiat Transfers" sub-tab appears under the Pending Requests tab
- [ ] Table displays the following columns:
  - Request ID
  - Status
  - Trade ID / Transfer Ref
  - Type (Client Settlement CL2 / LP Settlement ML1 / House Transfer)
  - Counterparty (client or LP name; blank for House Transfer)
  - Amount (amount + currency)
  - From Account (bank name + account number)
  - To Account (bank name + account number)
  - Initiator (maker name + email)
  - Created At
  - Updated At
- [ ] Status filter: All / Pending Approval / Approved / Rejected / Failed
- [ ] Type filter: All / Client Settlement / LP Settlement / House Transfer
- [ ] Free-text search by Trade ID, Transfer Reference, or Counterparty name
- [ ] Checker cannot see Approve action on requests they submitted themselves

---

### US-05 — Approve Fiat Transfer as Checker

**As an** Operator (checker),  
**I want to** approve a pending fiat transfer in HexAdmin and confirm on my mobile app,  
**so that** the bank transfer is only initiated after proper dual-party authorisation.

**Acceptance Criteria:**

- [ ] Checker clicks Approve on a pending fiat transfer in the Fiat Transfers sub-tab
- [ ] `custody-authz-service` sends a mobile push notification to the checker's HexSafe mobile app
- [ ] Mobile shows the "Fiat Transfer Initiation" template with:
  - Trade ID (UC1/UC2) or Transfer Reference (UC5)
  - Amount and currency
  - Counterparty name (UC1/UC2 only; not shown for UC5)
  - From bank: bank name, account holder name, routing number (if applicable), account number / IBAN, SWIFT code (if applicable)
  - To bank: same fields
  - Transaction ID: **not shown** (transfer has not been submitted to the bank yet)
  - Note: payment reference derived from trade ID
  - Initiated by: maker name
  - Initiated at: submission timestamp
- [ ] Checker approves on the mobile app
- [ ] On approval: settlement engine calls the bank API; trade status moves to **Bank Transfer Initiated**
- [ ] Approval recorded in audit trail: checker ID, timestamp
- [ ] Checker can also **Reject** from HexAdmin:
  - Trade status reverts to **Unsettled**
  - Slack notification sent to ops team and trading team
  - Rejection recorded in audit trail

---

### US-06 — Bank Confirms Transfer: Status Moves to Fiat Settled

**As an** Operator,  
**I want** the trade to automatically move to Fiat Settled when the bank confirms the transfer,  
**so that** I don't need to manually update the settlement status.

**Acceptance Criteria:**

- [ ] When the bank webhook confirms the transfer, trade status moves to **Fiat Settled**
- [ ] Bank transaction ID and value date are recorded automatically
- [ ] Bank transaction ID is visible in the trade detail view

---

### US-07 — Handle Transfer Failure and Retry

**As an** Operator,  
**I want to** be notified when a fiat transfer fails and be able to retry it from HexAdmin,  
**so that** I can resolve settlement failures without switching to the bank console.

**Acceptance Criteria:**

- [ ] If the bank API call fails after checker approval: system auto-retries up to 3 times with exponential backoff
- [ ] No Slack notification is sent during transient retry attempts
- [ ] After all 3 retries are exhausted: trade status moves to **Transfer Failed** (red "Transfer Failed" badge in Status column)
- [ ] Slack notification sent to ops team and trading team on Transfer Failed
- [ ] A "Retry" button appears in the Actions column on the trade row
- [ ] Clicking Retry opens a confirmation dialog showing transfer details (From, To, amount, method)
- [ ] Confirming Retry initiates a **fresh maker-checker cycle** (same flow as initiating from scratch: maker submits → checker approves on mobile)
- [ ] The existing Manual Fiat Confirmation action remains visible and accessible at all times regardless of failure state

---

### US-08 — Manual Fiat Confirmation Always Available

**As an** Operator,  
**I want** the manual fiat confirmation action (fill in txId / amount / value date) to always be available on a trade,  
**so that** I can fall back to it if the API initiation flow cannot be used or has failed.

**Acceptance Criteria:**

- [ ] The Manual Fiat Confirmation action is visible on every CL2 and ML1 trade at all times
- [ ] It remains available regardless of whether API initiation has been attempted, is in-flight, pending approval, or has failed
- [ ] Using Manual Fiat Confirmation does not require or affect the API initiation state

---

## Phase 2 — House-to-House Bank Transfer (UC5)

---

### US-09 — Initiate House-to-House Bank Transfer

**As an** Operator (maker),  
**I want to** initiate a transfer between Hex's own bank accounts from HexAdmin,  
**so that** I can rebalance funds between settlement accounts without using the bank console.

**Acceptance Criteria:**

- [ ] An "Internal Bank Transfer" sub-tab is available under the Trade Settlement screen in HexAdmin
- [ ] "From" selector shows only pre-configured Hex house bank accounts (Zand or BCB), sourced from House-type trading clients in the Trading Counterparty system — no free-form account entry
- [ ] "To" selector shows the same list, excluding the selected From account
- [ ] Fields: From account, To account, Amount, Currency, Notes / Reference
- [ ] Currency must match the From account's supported currency (e.g. Zand supports AED only)
- [ ] Transfer method is derived automatically per the same Zand / BCB eligibility rules
- [ ] Transfer Reference is a UUID generated by the system
- [ ] Maker submits → trade status moves to **Pending Approval**; request appears in the Fiat Transfers sub-tab under Pending Requests
- [ ] Submission is recorded in the audit trail: operator ID, timestamp, transfer reference, from/to accounts, amount, currency, transfer method

---

### US-10 — Approve House-to-House Transfer as Checker

**As an** Operator (checker),  
**I want to** approve a pending house-to-house transfer in HexAdmin and confirm on mobile,  
**so that** internal fund movements require proper dual-party authorisation.

**Acceptance Criteria:**

- [ ] Same approval flow as US-05 with the following differences:
  - Mobile template shows Transfer Reference instead of Trade ID
  - Counterparty Name is not shown (both accounts are Hex house accounts)
- [ ] On approval: bank API called; bank transaction ID recorded
- [ ] On rejection: transfer reverts; Slack notification sent to ops team and trading team
- [ ] Failure handling: same as US-07 (auto-retry → Transfer Failed → Slack → fresh maker-checker cycle)

---

### US-11 — Risk Controls for House-to-House Transfer

**As a** Risk / Compliance Officer,  
**I want** house-to-house transfers to be restricted to pre-configured accounts only,  
**so that** operators cannot initiate ad-hoc transfers to arbitrary accounts.

**Acceptance Criteria:**

- [ ] Only pre-configured Hex house accounts are selectable as source and destination — no free-form account entry
- [ ] The system rejects any attempt to submit a transfer where the currency does not match the From account's supported currency
- [ ] Every house-to-house transfer is included in the full audit trail

---

## Phase 3 — Incoming Fiat Confirmation (UC3 + UC4)

---

### US-12 — View Incoming Bank Transactions on CL1 Trade

**As an** Operator,  
**I want to** see incoming bank transactions associated with a CL1 trade directly in HexAdmin,  
**so that** I can identify which transactions to use for confirming settlement without checking the bank console.

**Acceptance Criteria:**

- [ ] For each CL1 trade pending fiat receipt, an inline section shows unmatched incoming bank transactions filtered by currency and approximate amount range
- [ ] Each transaction displays: bank reference, amount, currency, value date, sender name, sender account number, remaining available balance
- [ ] Transactions already fully consumed by other trades are not shown
- [ ] Transactions are received automatically via bank webhooks — no manual refresh required

---

### US-13 — Confirm Incoming Fiat Settlement: Client Pays Hex (CL1)

**As an** Operator,  
**I want to** select the incoming transactions that cover a CL1 trade and confirm settlement,  
**so that** the trade is marked as settled and the bank references are recorded.

**Acceptance Criteria:**

- [ ] Operator selects one or more transactions for a trade and clicks "Confirm Settlement"
- [ ] System validates that the sum of selected transaction amounts is **≥** the expected trade amount
- [ ] If sum < expected: confirmation is blocked; operator waits for remaining transactions to arrive
- [ ] If sum ≥ expected: trade leg moves to **Fiat Settled**; bank transaction references, amounts, and value dates are recorded
- [ ] Settlement is all-or-nothing — no partial settlement state
- [ ] No maker-checker required for incoming confirmation (record action, not a fund movement)

---

### US-14 — Transaction Amount Tracking for Incoming Fiat

**As an** Operator,  
**I want** a single incoming bank transaction to be usable across multiple trades if it covers more than one,  
**so that** I'm not blocked by clients or LPs sending one combined payment for multiple trades.

**Acceptance Criteria:**

- [ ] When a transaction is selected to confirm a trade, the amount used for that trade is recorded against the transaction
- [ ] The remaining unused balance on the transaction stays available for matching to other trades
- [ ] When multiple transactions are selected for one trade and the total exceeds the trade amount, the system allocates smallest-to-largest — at most one transaction will have a remaining unused balance
- [ ] A transaction whose full balance has been consumed no longer appears in the matching list

---

### US-15 — Confirm Incoming Fiat Settlement: LP Pays Hex (ML2)

**As an** Operator,  
**I want to** select the incoming transactions that cover an ML2 trade and confirm settlement,  
**so that** LP fiat receipts are recorded against the correct trade.

**Acceptance Criteria:**

- [ ] Same flow and acceptance criteria as US-13, with the following differences:
  - Inline transaction list appears on the Market Leg 2 settlement screen
  - Transactions are matched against the LP's expected payment
  - Incoming transactions target Hex's market settlement bank accounts

---

### US-16 — Unmatched Incoming Transactions View *(lower priority)*

**As an** Operator,  
**I want to** see incoming transactions that have not been matched to any trade,  
**so that** I can investigate unexpected or unidentified bank receipts.

**Acceptance Criteria:**

- [ ] A dedicated Unmatched Transactions view shows all incoming transactions with remaining unused balance not matched to any trade
- [ ] Displays: bank reference, amount, currency, value date, sender name, sender account number, received at timestamp

---

## Cross-Cutting

---

### US-17 — Fiat Transfer Slack Notifications

**As an** Ops team member or Trader,  
**I want to** receive a Slack notification when a fiat transfer is rejected or reaches a terminal failure,  
**so that** I can take action without continuously monitoring HexAdmin.

**Acceptance Criteria:**

- [ ] Slack notification sent to **both** the ops team channel and the trading team channel for:
  - Transfer rejected by checker (immediately on rejection)
  - Transfer Failed after all automatic retries are exhausted
  - Any other terminal failure (immediately)
- [ ] No Slack notification is sent for transient failures where automatic retry is still pending
- [ ] Notification includes: trade ID (or transfer reference for UC5), amount, currency, failure reason

---

### US-18 — Fiat Transfer Audit Trail

**As a** Compliance Officer,  
**I want** every state transition on a fiat transfer to be logged with full details,  
**so that** there is a complete audit trail of who initiated, approved, and when each transfer was executed.

**Acceptance Criteria:**

- [ ] The following events are logged:

| Event | Data captured |
|---|---|
| Initiated | Operator ID, timestamp, trade ID / transfer ref, leg, from/to accounts, amount, currency, transfer method |
| Approved / Rejected | Checker ID, timestamp |
| Bank API called | Timestamp, API request reference |
| Bank confirmed | Bank transaction ID, value date, timestamp |
| Failed | Failure reason, retry count |

- [ ] Bank transaction IDs returned from Zand / BCB are stored and visible in the trade detail view
