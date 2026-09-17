# DeepLedger Plugin

AI bookkeeping for QuickBooks Online, packaged as one plugin for Claude Code, Cursor, Grok Build and Grok Bot. It bundles eleven bookkeeping skills, a connector to DeepLedger's hosted MCP server and, in Claude Code, prompt hooks that guard every QuickBooks write. There are no slash commands and no custom agents: describe what you need and the matching skill activates.

QuickBooks Online stays the ledger of record. DeepLedger holds the QuickBooks connection your company authorized through Intuit's own OAuth flow, so the plugin never sees Intuit credentials. It signs in to DeepLedger as you and reaches every company you can open in the DeepLedger portal.

## Prerequisites

- A DeepLedger account with at least one QuickBooks Online company connected at [deepledger.ai](https://deepledger.ai) (Settings, QuickBooks, Connect)
- One of the hosts below

## Install

Marketplace availability depends on each marketplace's review. Until a listing appears, use the local install for that host.

### Claude Code

```bash
claude plugin marketplace add DeepLedger/deepledger-plugin
claude plugin install deepledger@deepledger-marketplace
```

Local development: `claude --plugin-dir ./deepledger-plugin`.

### Cursor

Customize, Marketplace, search for DeepLedger, Add. Local install: clone this repository and symlink it into Cursor's local plugins folder, then reload Cursor.

```bash
git clone https://github.com/DeepLedger/deepledger-plugin.git
mkdir -p ~/.cursor/plugins/local
ln -s "$(pwd)/deepledger-plugin" ~/.cursor/plugins/local/deepledger
```

### Grok Build

Open the plugin marketplace inside Grok Build and add DeepLedger. The catalog entry points at this repository.

### Grok Bot

Settings, Plugins, search for DeepLedger, Add, then complete the sign-in in your browser. Attach the plugin to a task with `@` and ask which QuickBooks company is active before requesting accounting work.

### Sign-in

On first use the host discovers the server's OAuth 2.1 endpoints (authorization code with PKCE, dynamic client registration) and opens a browser sign-in. Sign in with your DeepLedger account and the connection is authorized. DeepLedger personal API keys (`dl_live_...`) are also accepted as Bearer tokens for programmatic clients. The transport is Streamable HTTP at `https://mcp.deepledger.ai/mcp`.

## Quick start

```
# 1. Onboard a new client (run once per client)
"Bootstrap this client from their QuickBooks history"

# 2. Process the bank feed
"Process the bank feed"

# 3. Record a transaction
"Record: paid $500 to Office Depot for office supplies with the company credit card"

# 4. Reports and analysis
"Generate a P&L for last month and compare it to the prior month"

# 5. Month-end close
"Close the books for June"
```

> **First time?** Run the client-onboarding skill once per client. It assesses the books and seeds durable memory (client policies, confirmed recurring patterns, lasting context) for human review. Categorization works without it, because accounts are inferred in real time from QuickBooks history, but onboarding gives the agent the rules and context that history cannot express.

## Skills

| Skill | What it covers |
|-------|----------------|
| `client-onboarding` | Onboard a new client: assess the books, seed policies, patterns and general memory for human review |
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

## Safety model

Every QuickBooks write follows a three-step protocol:

1. **Lookup**: `qbMasterData` resolves vendor, customer and account IDs.
2. **Duplicate check**: `qbFetchTransactions` verifies no duplicate exists.
3. **Decide**: proceed only if the user requested it, a reviewer approved it, or QuickBooks history supports it (the consistency rule); otherwise escalate as a review task. User confirmation is reserved for interrupts: duplicates, amount anomalies, wrong-type guards, voids.

In Claude Code the protocol is enforced by prompt-type PreToolUse hooks in `.claude-plugin/hooks.json`. Other hosts follow the same protocol through the skills and the server's own tool guidance. Additional guards:

- Transaction-type guards catch wrong-tool writes (Expense vs BillPayment, Deposit vs ReceivePayment, Invoice vs SalesReceipt)
- Vendor and account IDs are cross-referenced against the latest `qbMasterData` results to catch invented IDs
- Journal entries are blocked unless debits equal credits
- Void operations require fetching and verifying the transaction first
- Review tasks (`tasks` create) must include specific `aiReasoning`

There is no batch tool. Every transaction is recorded individually with the appropriate tool (`qbExpense`, `qbBill`, and so on), so each write passes through the full protocol.

## Agent memory

The plugin uses `agentMemory` for durable per-client knowledge, never for vendor-to-account mappings, which are inferred in real time from QuickBooks history:

| Type | Purpose | Example |
|------|---------|---------|
| `patterns` | Confirmed recurring charges or income seen 2+ times (vendor, amount range, frequency, account) | "AWS monthly invoice about $800 to 6030 Cloud Hosting" |
| `policies` | Accounting rules: explicit reviewer or client instructions, or agent-observed policies citing QuickBooks evidence | "Capitalize fixed assets over $2,500 (client policy)" |
| `general` | Lasting client context not available in QuickBooks | "Fiscal year ends March 31" |
| `system` | Machine-written app state rendered by the portal (hidden from the memory page) | `bootstrap_status`, `latest_forecast` |

### Categorization: real time, not stored

Vendor and customer categorization comes from QuickBooks history at decision time, not from stored mappings. The decide gate proceeds only when one of these holds:

1. The user explicitly requested or confirmed the exact transaction
2. A reviewer approved it via task (`effectiveCategory` used verbatim)
3. The **consistency rule** passes on the entity's 6-month history: at least 3 transactions, dominant account at least 70%, no runner-up at 20% or more, amount within 5x the median

Anything else is escalated as a review task; the agent never guesses an account. Because inference is real time, a recategorization made in QuickBooks takes effect on the very next transaction. There is no stale mapping to correct.

## Architecture

```
Plugin (this repo)
  skills/                      11 bookkeeping and review preparation skills (all hosts)
  .claude-plugin/plugin.json   Claude Code and Grok Build manifest
  .claude-plugin/hooks.json    Prompt hooks guarding QuickBooks writes (Claude Code)
  .mcp.json                    MCP connector for Claude Code and Grok Build
  .cursor-plugin/plugin.json   Cursor and Grok Bot manifest
  mcp.json                     MCP connector for Cursor
    | Streamable HTTP + OAuth 2.1
MCP server (hosted at https://mcp.deepledger.ai/mcp)
  27 tools (20 QuickBooks + 7 platform)
  Tasks, memory, documents, bank feed, custom reports, close runs, workflow guides
```

## Connecting QuickBooks

1. Log in to [deepledger.ai](https://deepledger.ai)
2. Open Settings, QuickBooks
3. Click Connect QuickBooks and authorize access
4. Once connected, the plugin can read and write to that company

## Version and license

See [CHANGELOG.md](CHANGELOG.md) for release history. MIT License, see [LICENSE](LICENSE).
