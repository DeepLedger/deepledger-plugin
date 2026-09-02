---
name: bank-reconciliation
description: Reconcile bank and credit card accounts — process unrecorded bank transactions, fix duplicates and uncategorized items, then complete the reconciliation in QuickBooks Online via browser. Use when the user mentions reconciliation, bank matching, uncleared items, account health, or closing the books.
---

# Bank Reconciliation Skill

Keep bank and credit card accounts clean and audit-ready. Reconciliation runs in two phases:

1. **Prep phase (MCP tools)** — health check, record missing transactions, resolve duplicates and uncategorized items
2. **Reconcile phase (browser)** — open QB Online reconciliation UI, mark transactions cleared, enter statement balance, finalize

## Which company? (do this first)

One DeepLedger connection reaches every company the user can access. Before
anything else in this skill:

1. Call `qbCompanyProfile` (operation `profile`, the default). Its `company`
   field names the company you are in. Say the company name to the user in your
   first line.
2. If it answers `NO_ACTIVE_COMPANY`, call `qbCompanyProfile` with
   `operation: "list"`, then `operation: "switch"` with the `organizationId` the
   user means. Never guess between similar names; ask.
3. If the user names a different company than the active one, switch first.
   The switch is proven by a CompanyInfo read; a `qbConnected: false` answer
   means QuickBooks is not connected for that company and only tasks, documents,
   memory and the bank feed will work.
4. Every tool result carries `company`. If one carries `warning` (the active
   company was moved from the portal or another client), stop and confirm the
   company with the user before writing.

## Trigger

Activate when the user wants to:
- Check account health or scores
- Process unrecorded bank feed transactions
- Find and resolve duplicate transactions
- Clean up uncategorized or mis-categorized transactions
- Complete a full reconciliation in QuickBooks Online

## Tool Coverage

| Capability | How |
|------------|-----|
| Flag duplicates, uncategorized, outliers, past-due | `qbFetchTransactions` + `qbReports(reportType="GeneralLedger")` |
| Fetch unprocessed bank transactions to record | `bankFeed` |
| Investigate specific transactions | `qbFetchTransactions` |
| Void supported transaction types | `qbVoidTransaction` |
| Mark transactions as cleared in QB | Browser → QB Reconcile UI |
| Enter statement ending balance | Browser → QB Reconcile UI |
| Complete / finalize reconciliation | Browser → QB Reconcile UI |

## Workflow: Account Health Check

Run this first to find what needs fixing before touching the reconciliation screen.

1. **Get accounts** — `qbMasterData(entityTypes=["account"])` → filter for Bank and Credit Card types
2. **Run health check** — for each account, pull the period's activity with `qbFetchTransactions` (accountId + date range) and `qbReports(reportType="GeneralLedger")`
3. **Review for flags**:
   - **Duplicates** — same amount + date + vendor; high severity if > $1,000
   - **Uncategorized** — booked to "Ask My Accountant" or "Uncategorized"; high if > $500
   - **Outliers** — amount more than 5× the entity's historical median (charter §6)
   - **Past-due** — transactions with `dueDate` older than 7 days (checks `dueDate` field, not QB cleared status)
4. **Score** — compute 0–100 per account. Penalty: high = -5, medium = -3, low = -1

### Score Thresholds

| Score | Status | Action |
|-------|--------|--------|
| 95–100 | Audit-ready | Proceed to reconcile |
| 90–94 | Good | Minor cleanup first |
| 80–89 | Needs attention | Resolve flags before reconciling |
| < 80 | Critical | Fix issues before touching the reconciliation screen |

## Workflow: Process Bank Feed Transactions

`bankFeed` returns transactions that exist at the bank but are **not yet recorded in QB**. Record each one before reconciling.

1. **Fetch** — `bankFeed(action="fetch", accountId?, sinceDate?)`
2. **For each transaction**:
   - Check `agentMemory` for applicable `policies`/`patterns`, then pull the entity's 6-month QB history (`qbMasterData` + `qbFetchTransactions`) — categorization is realtime, never a stored mapping
   - **Decide gate passes** (user-requested, CPA-approved, or history-consistent per charter §6) → record using the correct tool (see table below)
   - **Unknown or ambiguous** → `tasks(operation="create")` with reasoning
3. **Duplicate check before recording** — `qbFetchTransactions` (report scan, same accountId + date range) to confirm it isn't already in QB

### Bank Line → Recording Tool

| Bank Line Type | Tool |
|----------------|------|
| Vendor payment / expense | `qbExpense` |
| Customer deposit / payment received | `qbDeposit` |
| Transfer between accounts | `qbTransfer` |
| Payroll or complex entry | `qbJournalEntry` |
| Unclear / needs CPA | `tasks(operation="create")` |

## Workflow: Resolve Duplicate Flags

1. **Pull both entries** — `qbFetchTransactions` to confirm details of each
2. **Verify it's a true duplicate** — same amount, vendor, and purpose (two similar charges from one vendor can both be legitimate)
3. **Check voidable type** — `qbVoidTransaction` supports: `BillPayment`, `Invoice`, `Payment`, `SalesReceipt`, `CreditMemo`, `Purchase`, `RefundReceipt`, `Transfer`
   - Expenses recorded via `qbExpense` are `Purchase` entities in the QB API — void them with `transactionType="Purchase"`
   - `Bill`, `JournalEntry`, `Deposit`, `VendorCredit` cannot be voided via tools — flag for CPA to handle in QB
4. **Confirm with user** before voiding — preserves audit trail, unlike delete
5. **Re-run** the health check (`qbFetchTransactions` scan) to verify the flag is cleared

## Workflow: Resolve Uncategorized Flags

1. **Fetch transaction details** — `qbFetchTransactions` report scan (accountId + dates) to find entries in "Ask My Accountant" or "Uncategorized"
2. **Infer the account** — `qbFetchTransactions` 6-month scan for the vendor and apply the consistency rule; check `agentMemory` `policies`/`patterns` for client rules
3. **Re-categorize if the rule passes** — update with the correct account
4. **Escalate if uncertain** — `tasks(operation="create")` with `aiReasoning` explaining what's unknown

## Workflow: Complete Reconciliation in QB Online (Browser)

Once the health score is clean and all bank feed items are recorded:

1. **Open QB Online** — navigate to Bookkeeping → Reconcile in the browser
2. **Select account** — choose the Bank or Credit Card account to reconcile
3. **Enter statement details**:
   - Beginning balance (should match QB's calculated opening balance)
   - Ending balance from the bank statement
   - Statement end date
4. **Mark transactions cleared** — check off each transaction that appears on the bank statement; the running difference should approach $0.00
5. **Investigate any remaining difference** — use `qbFetchTransactions` in a separate call to look up suspicious items while the reconciliation screen stays open
6. **Stop and report** — show the user the current difference and a summary of uncleared items; **do not click "Finish now"** — wait for the user to review and explicitly confirm before finalizing
7. **Finalize only on explicit user confirmation** — proceed to click "Finish now" only when the user says the difference is acceptable and instructs you to complete. Never auto-finalize even if the difference is $0.00
8. **Download reconciliation report** — QB generates a PDF summary after finalization; save or attach for audit records

> **Hard rule: never click "Finish now" / "Complete reconciliation" without explicit user instruction.** The user may need to fix transactions, adjust the statement balance, or defer finalization to the CPA.

## Safety Checklist

- [ ] Health check run (`qbFetchTransactions` + `qbReports`) and flags resolved before opening reconciliation screen
- [ ] Bank feed processed — no unrecorded transactions remaining for the period
- [ ] Duplicate check completed before recording any bank feed item
- [ ] Voidable transaction type confirmed before calling `qbVoidTransaction`
- [ ] Uncategorized items resolved or flagged — none left in "Ask My Accountant"
- [ ] Difference reported to user and user has explicitly confirmed to finalize — never auto-complete
- [ ] User confirmation before any void or re-categorization

## Common Mistakes to Avoid

- Opening the reconciliation screen before fixing health flags — unresolved duplicates and uncategorized items make the difference impossible to close
- Attempting to `qbVoidTransaction` on `Bill`, `JournalEntry`, `Deposit`, or `Expense` — not supported, will fail
- Voiding without confirming it's a true duplicate — two similar charges may both be legitimate
- Recording a bank feed transaction without checking if it already exists in QB first
- Clicking "Finish now" without explicit user instruction — even at $0.00 difference, always stop and confirm first
- Force-finishing a reconciliation with a non-zero difference — creates a discrepancy that compounds every month
