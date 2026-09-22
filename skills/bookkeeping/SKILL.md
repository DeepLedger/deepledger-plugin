---
name: bookkeeping
description: Bookkeeping for QuickBooks Online through the DeepLedger MCP server. Use for any accounting request — recording payments, bills, invoices, deposits, transfers or journal entries, processing the bank feed, reconciling a statement, closing a month, analyzing financials, preparing audit support, or managing vendors, customers and the chart of accounts. Pulls the current procedure from the server's getGuide tool before acting.
---

# Bookkeeping

DeepLedger keeps its bookkeeping procedures on the server, not in this plugin.
Your job here is to pull the right procedure with `getGuide`, follow it, and
verify the outcome. Tool schemas remain the authority for accepted arguments.
QuickBooks Online is a live system with no sandbox: every write creates,
changes or voids a real financial record.

## 1. Confirm the company

One DeepLedger connection reaches every QuickBooks company the user can open.
Call `qbCompanyProfile` first and say the company name back to the user. If it
answers `NO_ACTIVE_COMPANY`, call it with `operation: "list"`, then
`operation: "switch"` with the `organizationId` the user means; never guess
between similar names. If the user names a different company than the active
one, switch before doing anything else. Every tool result carries `company`;
if one carries `warning`, the active company was moved from outside this
session, so stop and reconfirm before writing.

## 2. Pull the procedure with getGuide

Before starting any accounting task, call `getGuide` with the `guideType`
that matches the request, then follow its `steps`, `safetyChecklist` and
`commonMistakes`. Do not work from a remembered copy; the published guide is
the shared procedure for every DeepLedger client and it changes without a
plugin release.

| The user wants to... | Call |
|----------------------|------|
| Record, enter, book or log a payment, purchase, expense, bill, bill payment, sale, invoice, customer payment, refund, credit, deposit, transfer or journal entry; categorize or record bank feed items; any accounts payable or receivable work that writes to QuickBooks | `getGuide(guideType="transaction_recording")` |
| Close a month, prepare adjusting entries, run the close checklist, draft or update the Close Sheet | `getGuide(guideType="month_end_closing")` |
| Reconcile a bank or credit-card statement, explain a reconciliation difference, prepare a reconciliation workbook | `getGuide(guideType="reconciliation")` |
| Compare reports, explain a profit or cash change, compute ratios or trends, analyze budget variances | `getGuide(guideType="financial_analysis")` |
| Assemble audit or review support: supporting schedules, an evidence index, a missing-document list | `getGuide(guideType="audit_preparation")` |
| Recover from a tool failure you cannot resolve from its response | `getGuide(guideType="error_recovery", tool="<failed tool>", errorCode="<returned code>")` |

Requests with no dedicated guide (looking up or editing vendors, customers,
accounts, items and classes with `qbMasterData`; agent memory; review tasks;
documents; custom reports; onboarding a new client) follow the tool's own
description. If that work ends in a QuickBooks write, load
`transaction_recording` first; its protocol applies to every write.

## 3. Read the result

- Workflow guides answer in `steps`, `safetyChecklist` and `commonMistakes`
  and carry no `content`.
- `error_recovery` answers in `content` (markdown) plus `codes` (every error
  code the playbook covers), `recovery` (matched actions with verification
  and stop conditions) and `topics` (narrative sections readable with
  `topic`). Pass `errorCode` to get one entry instead of the whole playbook;
  a pre-flight refusal with no code can be looked up by pasting its message.
- On `success: false`, branch on `errorCode`, never on the message:
  - `GUIDE_NOT_AVAILABLE`: the guide is missing or in draft. This is not an
    empty checklist. Tell the user the procedure is unavailable and hand off
    work that depends on it. Do not invent the procedure.
  - `GUIDE_FETCH_FAILED`: storage read failed. Retry once; if it persists,
    treat it like `GUIDE_NOT_AVAILABLE`.
  - `NO_GUIDE_FOR_TOOL`: no playbook for that tool; the summary lists the
    tools that have one.
  - `NO_ENTRY_FOR_CODE`: the playbook loaded but has no such code. Read
    `codes` and ask again with the closest one.
  - `NO_TOPIC_FOR_GUIDE`, `TOOL_REQUIRED`: fix the arguments and retry.

## 4. Rules that hold whatever the guide says

- Look up IDs with `qbMasterData` and check for duplicates with
  `qbFetchTransactions` before any write. Never use an ID the server did not
  return in this conversation.
- Write only when the user explicitly requested or confirmed the exact
  transaction, a reviewer approved it through `tasks`, or QuickBooks history
  clearly supports the categorization. Otherwise create a review task with
  specific `aiReasoning` instead of writing.
- Journal entries must balance. Fetch and verify a transaction before voiding
  it; voids cannot be undone.
- Report confirmed outcomes and open tasks. A tool call alone is not proof
  that the transaction, attachment or close state was saved; read back an
  uncertain write before retrying.
- This skill does not expand the user's request, the approval given, or the
  credential's permissions.
