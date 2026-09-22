---
name: bookkeeping
description: Bookkeeping for QuickBooks Online through DeepLedger MCP. Use for payments, expenses, bills, invoices, deposits, transfers, journal entries, bank feeds, reconciliation, month-end close, financial analysis, audit support, and managing vendors, customers or accounts. Loads current procedures with getGuide before acting.
---

# Bookkeeping

Load current procedures from DeepLedger with `getGuide`, follow them, and verify the result. Use tool schemas for accepted arguments. QuickBooks is live: every write changes real financial records.

## 1. Select the company

Call `qbCompanyProfile` first and tell the user the active company. The connection can access multiple companies:

- If it returns `NO_ACTIVE_COMPANY`, use `operation: "list"`, then `operation: "switch"` with the intended `organizationId`.
- Switch if the user named a different company. Ask if the intended company is unclear; never guess between similar names.
- Check each result's `company`. A `warning` means the active company changed outside this session; stop and reconfirm before writing.

## 2. Load the procedure

Call `getGuide(guideType="...")` for the task below. Fetch the current guide each time; do not rely on a remembered copy. Follow its `steps`, `safetyChecklist` and `commonMistakes`, with the routine-recording policy in section 3.

| Task | `guideType` |
|------|-------------|
| Record transactions, categorize bank feeds, or write to accounts payable or receivable | `transaction_recording` |
| Close a month, prepare adjustments, or draft/update the Close Sheet | `month_end_closing` |
| Reconcile bank/credit-card statements, investigate differences, or prepare a reconciliation workbook | `reconciliation` |
| Compare reports, explain financial changes, or analyze ratios, trends and budgets | `financial_analysis` |
| Prepare audit/review schedules, evidence indexes or missing-document lists | `audit_preparation` |

For master data, memory, review tasks, documents, custom reports and onboarding, follow the tool description. Load `transaction_recording` before any QuickBooks write, including writes within another workflow.

For an unresolved tool failure, call `getGuide(guideType="error_recovery", tool="<failed tool>", errorCode="<returned code>")`. If no code was returned, pass the refusal message as `errorCode`. Read `content` and `recovery`, including verification and stop conditions; use `codes` or `topics` to narrow further requests.

If `getGuide` returns `success: false`, use `errorCode`:

- `GUIDE_FETCH_FAILED`: retry once. If it persists, handle as `GUIDE_NOT_AVAILABLE`.
- `GUIDE_NOT_AVAILABLE`: explain that the procedure is unavailable and hand off dependent work; do not invent a replacement.
- `NO_GUIDE_FOR_TOOL`: consult the response's list of supported tools.
- `NO_ENTRY_FOR_CODE`: inspect `codes` and request a relevant entry.
- `NO_TOPIC_FOR_GUIDE` or `TOOL_REQUIRED`: correct the arguments and retry.

## 3. Decide whether to record

- **Record routine transactions within the user's requested scope** when the payee, category account, tool and amount are clear from the evidence and you are at least 95% confident. Evidence can include the description, QuickBooks history, memory or the user's instructions.
- **Create a review task** with `tasks(operation="create")` when something is uncertain, such as a new or ambiguous payee, multiple plausible categories, an unusual amount or an unreadable description. Include specific `aiReasoning` and a `suggestedCategory` when available.
- **Apply reviewer decisions first.** Record approved tasks using `effectiveCategory` verbatim before fresh analysis.

For routine recording, treat guide thresholds for history counts, category share and duplicate date windows as examples, not mandatory gates. Judge the evidence and use a duplicate window suited to the payee's cadence. The checks below still apply regardless of confidence.

## 4. Check and verify every write

- Resolve IDs with `qbMasterData` and check duplicates with `qbFetchTransactions` before writing. Use only IDs returned by the server in this conversation. Show any duplicate match and obtain confirmation before recording.
- Pay an outstanding bill with `qbBillPayment`; collect an outstanding invoice with `qbReceivePayment`. Do not create a second expense, deposit or sales receipt instead. The source account must differ from every line account.
- Balance journal entries. Before voiding a transaction, fetch it, verify it and obtain user confirmation; voids cannot be undone.
- Verify saved outcomes. Read back an uncertain write before retrying to avoid duplicates. Report confirmed results and open tasks, including whether transactions, attachments and close states were saved.
- Stay within credential permissions and reviewer decisions.
