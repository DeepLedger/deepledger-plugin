---
name: accounts-payable
description: Manage accounts payable — enter bills, pay vendors, apply vendor credits, monitor aging, and track outstanding obligations. Use when the user mentions bills, vendor payments, AP aging, bill pay, or outstanding payables.
---

# Accounts Payable Skill

Manage the full AP lifecycle: entering vendor bills, recording payments, applying vendor credits, and monitoring what you owe.

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
- Enter a vendor bill
- Pay a vendor bill
- Apply a vendor credit
- Check outstanding or overdue payables
- Review vendor spending
- Process a batch of bill payments
- Track payment terms and due dates

## The AP Flow

```
Bill → BillPayment → Bank Account
```

- `qbBill` creates the payable (AP goes up, no cash out yet)
- `qbBillPayment` pays the bill from a bank or credit card account (AP goes down, cash goes down)

For immediate-pay purchases with no bill: `qbExpense` records the payment directly.

## When to Use Bill vs Expense

| Situation | Tool | Why |
|-----------|------|-----|
| Vendor sends an invoice to pay later | `qbBill` | Creates AP, tracks the obligation |
| Paid vendor immediately (card swipe, ACH) | `qbExpense` | No AP needed, money already left |
| Paying a previously entered bill | `qbBillPayment` | Clears the AP, records the payment |

**Key rule**: If a `qbBill` already exists for this vendor+amount+date, use `qbBillPayment` — never create a duplicate `qbExpense`.

## Workflow: Enter a Bill

1. **Lookup** — `qbMasterData` for vendor ID and expense account IDs
2. **Check memory** — `agentMemory` for client `policies` and recurring `patterns`; the vendor's account comes from QB history (consistency rule), never a stored mapping
3. **Duplicate check** — `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)` to verify no existing bill for same vendor+amount
4. **Build bill** — Set `vendorId`, `txnDate`, `dueDate`, `lines` with account/amount
5. **Decide** — Proceed if user-requested, CPA-approved (use `effectiveCategory` verbatim), or history-consistent (consistency rule — see record-transactions skill); otherwise create a CPA task. Show: vendor, total, due date, expense categories
6. **Record** — `qbBill`
7. **Attach** — `qbAttachFile` (entityType = "Bill") — fetch from portal, local file, drive, or user upload; preferred for audit-ready books
8. **Learn** — if the charge is now confirmed recurring (2+ occurrences), write an `agentMemory` `patterns` entry (vendor, amount range, frequency, account)

## Workflow: Pay a Bill

1. **Find outstanding bills** — `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)`
2. **Select bills** — Identify which bill(s) to pay (by amount, date, or vendor)
3. **Choose payment account** — `qbMasterData` for Bank/CC account IDs
4. **Record** — `qbBillPayment` with `vendorId`, `paymentDate`, `bankAccountId`, `bills` array (billId + amount per bill)
5. **Partial payments** — Pay less than the full bill amount; the bill remains partially outstanding
6. **Multiple bills** — Pay several bills to one vendor in a single BillPayment using the `bills` array
7. **Attach** — `qbAttachFile` (entityType = "BillPayment") — remittance advice or bank confirmation; preferred for audit-ready books
8. **Decide** — The bill already encodes the account, so no re-inference: proceed if user-requested or CPA-approved. Show: vendor, bills being paid, total, payment account

## Workflow: Record an Expense (Immediate Payment)

1. **Check for existing bill** — `qbFetchTransactions(transactionType="Bill", outstandingOnly=true, entityId=vendorId)` — if a matching bill exists, use `qbBillPayment` instead
2. **Lookup** — `qbMasterData` for vendor ID, source account (bank/CC), category account
3. **Duplicate check** — `qbFetchTransactions(transactionType="Purchase", entityId=vendorId)` matching amount and date
4. **Record** — `qbExpense` with `paymentType`, `accountId` (source), `vendorId`, `lines` (category accounts)
5. **Attach** — `qbAttachFile` (entityType = "Purchase") — receipt from portal, local file, drive, or user upload; preferred for audit-ready books
6. **Rule** — Source account (where money comes from) must differ from line accountId (what it was spent on)

## Workflow: Apply Vendor Credit

When a vendor issues a credit or refund:

1. **Create credit** — `qbCredit(creditType="vendor")` with `vendorId`, `txnDate`, `lines` (account + amount)
2. **Apply to payment** — In `qbBillPayment`, include the VendorCredit in the `bills` array with `txnType: "VendorCredit"`
3. Use for: vendor refunds, billing adjustments, returned goods

## Workflow: AP Aging Review

1. **Pull aging** — `qbReports(reportType="AgedPayables")` for summary, `AgedPayablesDetail` for line items
2. **Calculate DPO** — Days Payable Outstanding = AP / (Annual COGS / 365)
3. **Bucket analysis** — Present in aging buckets: Current, 1-30, 31-60, 61-90, 90+
4. **Flag concerns**:
   - Bills past due → late payment penalties, vendor relationship risk
   - AP growing faster than revenue → cash flow pressure
   - Large upcoming payments → cash planning needed
5. **Recommendations** — Bills to prioritize for payment, vendors to negotiate terms with

## Workflow: Vendor Spending Analysis

1. **Pull report** — `qbReports(reportType="VendorExpenses")` for spending by vendor
2. **Identify** — Top vendors by total spend, fast-growing vendors, new vendors
3. **Compare** — Current period vs prior period
4. **Flag** — Vendor spend up 20%+ without corresponding revenue growth

## Safety Checklist

- [ ] `qbMasterData` lookup completed — valid vendor and account IDs
- [ ] Duplicate check — no existing bill or expense for same vendor+amount+date
- [ ] Check for outstanding bills before recording an expense
- [ ] Source account differs from category account on expense lines
- [ ] Decide gate passed — user-requested, CPA-approved, or history-consistent; otherwise a CPA task created (user confirmation only for duplicate/anomaly/wrong-type/void interrupts)
- [ ] Newly confirmed recurring charges written as `patterns` entries

## Common Mistakes to Avoid

- Recording an `qbExpense` when an outstanding `qbBill` already exists → use `qbBillPayment`
- Forgetting `vendorId` on expenses → breaks vendor spending reports and audit trail
- Source account same as category account → accounting error
- Creating a BillPayment without checking for outstanding bills first
- Wrong `entityType` when attaching documents: Expense = "Purchase" in the QB API
- Paying a bill from the wrong bank account
