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

## 4. Deciding whether to record

You record on your own judgment. A routine write does not need an explicit
user request or a reviewer's approval; it needs you to be sure.

- **Record** when the payee, the category account, the right tool and the
  amount are all obvious from the evidence (a clear description, QuickBooks
  history for this payee, a memory note, the user's words) and you are at
  least 95% confident. Do not pause to ask.
- **Create a review task** (`tasks(operation="create")` with specific
  `aiReasoning` and a `suggestedCategory` if you have one) only when
  something is genuinely uncertain: a new or ambiguous payee, more than one
  plausible category, an amount out of character for this payee, a
  description you cannot read.
- **Reviewer decisions win.** A task the reviewer approved is recorded with
  its `effectiveCategory` verbatim before any fresh analysis.

If the guide you loaded states fixed mechanical thresholds for the decide
gate (a minimum count of prior transactions, a dominant-share percentage, a
fixed duplicate date window), read them as illustrations of what obvious
looks like, not as gates. Judge the evidence; pick a duplicate window that
fits the payee's cadence.

## 5. Guards that hold regardless of confidence

These are QuickBooks correctness, not caution, so they never yield to
confidence:

- Look up IDs with `qbMasterData` and run a duplicate check with
  `qbFetchTransactions` before any write. Never use an ID the server did not
  return in this conversation. If the duplicate check returns a match, show
  it and confirm before recording.
- An outstanding bill means `qbBillPayment`, not a second expense; an
  outstanding invoice means `qbReceivePayment`, not a deposit or sales
  receipt. The source account must differ from every line account.
- Journal entries must balance. Fetch and verify a transaction before
  voiding it, and confirm voids with the user; they cannot be undone.
- Report confirmed outcomes and open tasks. A tool call alone is not proof
  that the transaction, attachment or close state was saved; read back an
  uncertain write before retrying.
- This skill does not expand the credential's permissions or the reviewer's
  decisions.
