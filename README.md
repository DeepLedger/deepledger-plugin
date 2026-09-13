# DeepLedger Plugin

Claude Code plugin for AI bookkeeping and financial analysis powered by QuickBooks Online.

The plugin is intentionally lean: **skills** (bookkeeping expertise that triggers on natural language), **hooks** (safety guards on every QuickBooks write), and the **DeepLedger MCP connector**. No slash commands or custom agents — just describe what you need.

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- A DeepLedger account with QuickBooks connected via the [portal](https://deepledger.ai)
- MCP server running at `https://mcp.deepledger.ai/mcp`

## Setup

```bash
# Run Claude Code with the plugin
claude --plugin-dir ./deepledger-plugin
```

The plugin connects to the DeepLedger MCP server using Streamable HTTP transport. On first use, Claude Code discovers the server's OAuth 2.1 (PKCE) endpoints and opens a browser login — sign in with your DeepLedger account and the connection is authorized automatically. The server also accepts DeepLedger API keys (`dl_live_...`) and Supabase session JWTs as Bearer tokens for programmatic access.

## Quick Start

Just ask in plain language — the matching skill activates automatically:

```
# 1. Onboard a new client — learn from existing QB history (run once per client)
"Bootstrap this client from their QuickBooks history"

# 2. Process the bank feed — agent already knows your vendors
"Process the bank feed"

# 3. Record a transaction
"Record: paid $500 to Office Depot for office supplies with the company credit card"

# 4. Reports and analysis
"Generate a P&L for last month and compare it to the prior month"

# 5. Month-end close
"Close the books for June"
```

> **First time?** Run the client-onboarding skill once per client. It assesses the books and seeds durable memory — client policies, confirmed recurring patterns, lasting context — with CPA review. Categorization works without it (accounts are inferred in realtime from QB history), but onboarding gives the agent the rules and context history can't express.

## Skills

| Skill | What it covers |
|-------|----------------|
| `client-onboarding` | Onboard a new client — assess the books, seed policies/patterns/general memory with CPA review |
| `bank-feed-processing` | Categorize, match, record, or flag bank feed transactions |
| `bank-reconciliation` | Prepare reconciliation workbooks and complete the QuickBooks UI handoff |
| `record-transactions` | Shared recording procedure: tool selection, approval, verification |
| `accounts-payable` | Bills, vendor payments, vendor credits, AP aging |
| `accounts-receivable` | Invoices, customer payments, credits, AR aging, collections |
| `journal-entries` | Journal entries, adjusting entries, transfers, corrections |
| `master-data` | Chart of accounts, vendors, customers, items, classes, tax rates |
| `month-end-close` | Shared close workflow: drafts the Close Sheet (anchored 16-point checks, line-level statements, proposed entries) for human sign-off in the portal |
| `financial-analysis` | Comparable reports, ratios, trends and evidence-backed explanations |
| `audit-preparation` | Supporting schedules, evidence index and missing-document requests |

## Safety Model

Every QuickBooks write operation is protected by a 3-step protocol enforced via hooks:

1. **Lookup** — `qbMasterData` to resolve vendor/customer and account IDs
2. **Duplicate check** — `qbFetchTransactions` to verify no duplicate exists
3. **Decide** — proceed only if user-requested, CPA-approved, or history-consistent (consistency rule); otherwise escalate as a CPA task. User confirmation is reserved for interrupts: duplicates, amount anomalies, wrong-type guards, voids

Additional guards:
- Transaction-type guards catch wrong-tool writes (Expense vs BillPayment, Deposit vs ReceivePayment, Invoice vs SalesReceipt)
- Vendor/account IDs are cross-referenced against the latest `qbMasterData` results to catch hallucinated IDs
- Journal entries are hard-blocked unless debits equal credits
- Void operations require fetching and verifying transaction details first
- CPA escalations (`tasks` create) must include specific `aiReasoning`

There is no batch tool — every transaction is recorded individually with the appropriate tool (`qbExpense`, `qbBill`, etc.), so each write passes through the full safety protocol.

## Agent Memory

The plugin uses `agentMemory` for durable per-client knowledge — never for vendor→account mappings, which are inferred in realtime from QB history:

| Type | Purpose | Example |
|------|---------|---------|
| `patterns` | Confirmed recurring charges/income seen 2+ times (vendor, amount range, frequency, account) | "AWS monthly invoice ~$800 → 6030 Cloud Hosting" |
| `policies` | Accounting rules — explicit CPA/client instructions, or agent-observed policies citing QBO evidence | "Capitalize fixed assets >$2,500 — client policy" |
| `general` | Lasting client context not available in QuickBooks | "Fiscal year ends March 31" |
| `system` | Machine-written app state rendered by the portal (hidden from the CPA memory page) | `bootstrap_status`, `latest_forecast` |

### Categorization: realtime, not stored

Vendor/customer categorization comes from QB history at decision time, not from stored mappings. The decide gate proceeds only when one of these holds:

1. The user explicitly requested or confirmed the exact transaction
2. A CPA approved it via task (`effectiveCategory` used verbatim)
3. The **consistency rule** passes on the entity's 6-month history: ≥3 transactions, dominant account ≥70%, no runner-up ≥20%, amount within 5× median

Anything else is escalated as a CPA task — the agent never guesses an account. Because inference is realtime, a CPA recategorization in QuickBooks takes effect on the very next transaction; there is no stale mapping to correct.

## Architecture

```
Plugin (this repo)
  ├── skills/     → 11 bookkeeping and review preparation skills
  ├── hooks/      → Safety validation guards on every QB write
  └── .mcp.json   → DeepLedger MCP connector
  ↓ Streamable HTTP
MCP Server (deepledger-mcp, hosted on Render at https://mcp.deepledger.ai/mcp)
  ├── 23 tools (17 QuickBooks + 6 platform)
  ├── Agent infrastructure (tasks, memory, documents, bank feed, close runs)
  └── Supabase PostgreSQL backend
```

## Connecting QuickBooks

1. Log in to [deepledger.ai](https://deepledger.ai)
2. Navigate to Settings > QuickBooks
3. Click "Connect QuickBooks" and authorize access
4. Once connected, the plugin can read and write to your QB company

## Version

See [CHANGELOG.md](CHANGELOG.md) for release history.
