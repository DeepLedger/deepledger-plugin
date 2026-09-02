---
name: accounts-receivable
description: Manage accounts receivable — create invoices, receive payments, apply credits, monitor aging, and follow up on overdue balances. Use when the user mentions invoices, customer payments, AR aging, collections, or outstanding receivables.
---

# Accounts Receivable Skill

Manage the full AR lifecycle: invoicing customers, receiving payments, applying credits, depositing funds, and monitoring overdue balances.

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
- Create or edit an invoice
- Record a customer payment
- Apply a credit memo to a customer balance
- Check outstanding or overdue receivables
- Follow up on collections
- Process a refund
- Deposit customer payments from Undeposited Funds to the bank

## The AR Flow

```
Invoice → ReceivePayment → Undeposited Funds → Deposit → Bank Account
```

- `qbInvoice` creates the receivable (AR goes up, no cash yet)
- `qbReceivePayment` applies cash against the invoice (AR goes down, Undeposited Funds goes up)
- `qbDeposit` moves funds from Undeposited Funds into the bank (matches the bank statement)

For immediate-pay sales with no invoice: `qbSalesReceipt` → Undeposited Funds → `qbDeposit`.

## Workflow: Create an Invoice

1. **Lookup** — `qbMasterData` for customer ID, item/service IDs, sales terms
2. **Check memory** — `agentMemory` for customer billing patterns (typical items, amounts, terms)
3. **Duplicate check** — `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)` to verify no duplicate
4. **Build invoice** — Set `customerId`, `txnDate`, `dueDate`, `lines` with items/amounts
5. **Decide** — Proceed if user-requested, CPA-approved (use `effectiveCategory` verbatim), or history-consistent (consistency rule — see record-transactions skill); otherwise create a CPA task. Show: customer, total, due date, line items
6. **Record** — `qbInvoice` to create
7. **Attach** — `qbAttachFile` (entityType = "Invoice") — signed contract, PO, or supporting doc from portal, local file, drive, or user upload; preferred for audit-ready books
8. **Optional** — Set `emailStatus: "NeedToSend"` to auto-email the invoice from QB

## Workflow: Receive Payment

1. **Find open invoices** — `qbFetchTransactions(transactionType="Invoice", outstandingOnly=true, entityId=customerId)`
2. **Match** — Identify which invoice(s) this payment applies to
3. **Record** — `qbReceivePayment` with `customerId`, `totalAmount`, `paymentDate`, and `invoices` array
4. **Partial payments** — If payment is less than invoice total, apply the amount received; the invoice stays partially outstanding
5. **Overpayments** — QB creates an unapplied credit; the `unappliedAmount` in the response shows the excess
6. **Attach** — `qbAttachFile` (entityType = "Payment") — bank remittance or payment confirmation; preferred for audit-ready books
7. **Update memory** — if this income stream is now confirmed recurring (2+ occurrences), write an `agentMemory` `patterns` entry (customer, amount range, frequency, account)

## Workflow: Deposit to Bank

When Undeposited Funds has a balance:

1. **Check** — `qbFetchTransactions` on the Undeposited Funds account to see pending items
2. **Group** — Match deposits to the lump sums on the bank statement (multiple payments often deposit as one line)
3. **Record** — `qbDeposit` with `depositAccountId` (the bank account) and line items from Undeposited Funds
4. **Verify** — Deposit total should match the bank statement line

## Workflow: Apply Credit Memo

When a customer has a credit balance:

1. **Create credit** — `qbCredit(creditType="customer")` with `customerId`, `txnDate`, `lines`
2. **Apply to payment** — When receiving the next payment, include `creditMemos` array in `qbReceivePayment`
3. Use credit memos for: returns, billing adjustments, promotional discounts

## Workflow: Issue Refund

When cash must be returned to the customer:

1. **Lookup** — `qbMasterData` for customer ID, item IDs
2. **Duplicate check** — `qbFetchTransactions(transactionType="RefundReceipt", entityId=customerId, startDate, endDate)`
3. **Record** — `qbRefundReceipt` with `customerId`, `depositAccountId` (bank account the refund comes from), `txnDate`, `lines`

## Workflow: AR Aging Review

1. **Pull aging** — `qbReports(reportType="AgedReceivables")` for summary, `AgedReceivablesDetail` for line items
2. **Calculate DSO** — Days Sales Outstanding = AR / (Annual Revenue / 365)
3. **Bucket analysis** — Present in aging buckets: Current, 1-30, 31-60, 61-90, 90+
4. **Flag concerns**:
   - Invoices over 90 days → discuss bad debt write-off with CPA
   - Rising DSO trend → collection process needs attention
   - Single customer > 30% of AR → concentration risk
5. **Recommendations** — Prioritized list of customers to follow up with

## Safety Checklist

- [ ] `qbMasterData` lookup completed — valid customer and item IDs
- [ ] Duplicate check via `qbFetchTransactions` — no matching open invoice
- [ ] Check for existing open invoices before creating new ones for same customer/service
- [ ] Verify payment amount matches or is less than outstanding balance
- [ ] Decide gate passed — user-requested, CPA-approved, or history-consistent; otherwise a CPA task created (user confirmation only for duplicate/anomaly/wrong-type/void interrupts)
- [ ] Newly confirmed recurring income streams written as `patterns` entries

## Common Mistakes to Avoid

- Recording a `qbDeposit` when an outstanding invoice exists → use `qbReceivePayment` first
- Forgetting the Undeposited Funds step — ReceivePayment does NOT put money in the bank
- Creating a duplicate invoice for a customer who already has one outstanding for the same service
- Applying a payment to the wrong invoice when a customer has multiple open
- Skipping `customerId` on invoices — breaks the AR aging report
