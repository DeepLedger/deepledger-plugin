---
name: bookkeeping
description: Bookkeeping through DeepLedger MCP for QuickBooks Online, Xero (beta), Bill.com and Stripe (for reporting). Use for payments, expenses, bills, invoices, deposits, transfers, journal entries, bank feeds, reconciliation, month-end close, financial analysis, audit support, and managing vendors, customers or accounts. Reads the company's own skill with the skills tool before acting.
---

# Bookkeeping

Read the company's playbook from DeepLedger with the `skills` tool, follow it, and verify the result. Use tool schemas for accepted arguments. The ledger is live: every write changes real financial records.

## 1. Select the company

Call `companies` first and tell the user the active company and its ledger. One connection can reach many companies, on QuickBooks Online or Xero (beta):

- If it returns `NO_ACTIVE_COMPANY`, use `operation: "list"`, then `operation: "switch"` with the intended company.
- Switch if the user named a different company. Ask if the intended company is unclear; never guess between similar names.
- Check each result's `company`. A `warning` means the active company changed outside this session; stop and reconfirm before writing.
- Use the tools of the active company's ledger: `qb*` (and `bankFeed`) for QuickBooks, `xero*` for Xero. Bill.com (`billcom*`) and Stripe (`stripemcpRead`) appear only for companies that connected them.
- If a tool you expect is missing, call `connections`; `describeTool` and `callReadTool` / `callWriteTool` reach any connected app's tools before your AI app refreshes its list.

## 2. Read the skill

Call `skills(operation="list")` to see the company's skills, then `skills(operation="get", skill="...")` for the job. Each company has its own copy of DeepLedger's default skills, with its own preferences. Read it each time; do not rely on a remembered copy.

| Task | Skill (typical name) |
|------|----------------------|
| Record transactions, categorize bank feeds, or write to payables or receivables | `record-transactions` |
| Close a month, prepare adjustments, or update the close | `month-end-close` |
| Reconcile bank or credit-card statements, investigate differences | `reconcile-bank` |
| Prepare audit or review schedules, evidence indexes or missing-document lists | `audit-preparation` |
| First-time setup of a company | `get-started` |

Use the names `list` returns; they can differ by company. For a job with no matching skill, follow the tool descriptions. Read the recording skill before any write to the books, including writes within another workflow. Facts about the company (policies, recurring patterns, reviewer corrections) live in `agentMemory`, the company's AI Brain: read it before you categorize.

For an unresolved tool failure, call `skills(operation="errors", tool="<failed tool>", errorCode="<returned code>")`. If no code was returned, pass the refusal message as `errorCode`. Follow the recovery steps, including verification and stop conditions.

## 3. Decide whether to record

- **Record routine transactions within the user's requested scope** when the payee, category account, tool and amount are clear from the evidence and you are confident. Evidence can include the description, ledger history, the AI Brain or the user's instructions.
- **Create a review task** with `tasks(operation="create")` when something is uncertain, such as a new or ambiguous payee, several plausible categories, an unusual amount or an unreadable description. Include specific reasoning and a suggested category when you have one.
- **Apply reviewer decisions first.** Record approved tasks as the reviewer decided before fresh analysis.

## 4. Check and verify every write

- Resolve IDs with the ledger's master-data tool (`qbMasterData` or `xeroMasterData`) and check duplicates with its fetch tool (`qbFetchTransactions` or `xeroFetchTransactions`) before writing. Use only IDs returned by the server in this conversation. Show any duplicate match and get confirmation before recording.
- Pay an outstanding bill or collect an outstanding invoice with the payment tool (`qbPayment` or `xeroPayment`). Do not create a second expense, deposit or sales receipt instead. The source account must differ from every line account.
- Balance journal entries. Before voiding or deleting a transaction, fetch it, verify it and get the user's confirmation.
- In Xero, new invoices, bills, credit notes and journals are saved as drafts by default.
- Bill.com: read, save drafts and edit only. Nothing here pays, approves or sends anything in BILL; a person finishes that in BILL.
- Stripe is read only, for reporting.
- Verify saved outcomes. Read back an uncertain write before retrying to avoid duplicates. Report confirmed results and open tasks, including whether transactions, attachments and close states were saved.
- Stay within the user's access level and reviewer decisions.
