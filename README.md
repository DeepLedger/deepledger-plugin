# DeepLedger

**AI staff accountant for QuickBooks**

DeepLedger connects QuickBooks to Grok, ChatGPT, Claude and other AI agents, allowing businesses to record bills, invoices, payments, deposits, journal entries and other transactions, run financial reports, manage receivables and payables, and track month-end close using natural language.

This repository packages DeepLedger as a plugin for Claude Code, Cursor, Grok Build and Grok Bot. Install it, sign in with your DeepLedger account, and ask for bookkeeping in plain language. The agent works your QuickBooks company the way a staff accountant would: it looks up the payee, checks for duplicates, picks the right transaction type, records it, and hands anything it is not sure about to a human reviewer instead of guessing.

QuickBooks Online stays the ledger of record. DeepLedger holds the QuickBooks connection your company authorized through Intuit's own OAuth flow, so the plugin never sees Intuit credentials. One sign-in reaches every company you can open in the DeepLedger portal.

## Who it is for

- **Small business owners and founders** who keep their own books in QuickBooks Online and want the routine entries, categorization and reports done for them.
- **Bookkeepers and CPA firms** managing many client companies. One connection switches between companies, every write follows the same protocol, and uncertain items land in a review queue with the agent's reasoning attached.
- **Finance teams** who want the month-end close, reconciliation prep and management reports drafted for review rather than built from scratch.

## What it does

| Area | What the agent does |
|------|---------------------|
| Bank feed | Pulls unrecorded bank and card transactions, categorizes them from the payee's QuickBooks history, records the obvious ones and creates review tasks for the rest |
| Recording | Records payments, bills, invoices, customer payments, deposits, transfers, refunds, credits and journal entries with an ID lookup and a duplicate check before every write |
| Payables and receivables | Enters bills and pays them against outstanding balances, invoices customers and applies their payments, applies credits, reports aging |
| Reconciliation | Matches a bank or card statement to the ledger and prepares the reconciliation workbook and open items |
| Month-end close | Works the close checklist, drafts adjusting entries and the Close Sheet, and hands the package to a reviewer for sign-off |
| Reports and analysis | Runs P&L, balance sheet, aging and custom reports, compares periods and explains what moved |
| Audit preparation | Builds supporting schedules, an evidence index and a missing-document list |
| Memory | Keeps durable per-company knowledge (policies, recurring patterns, context) and applies reviewer corrections going forward |
| Human review | Anything uncertain becomes a task with the agent's reasoning; reviewer decisions are applied verbatim |

## How to use it

1. **Connect QuickBooks** at [deepledger.ai](https://deepledger.ai): Settings, QuickBooks, Connect. Repeat for each company you manage.
2. **Install the plugin** in your host (below) and sign in when the browser opens.
3. **Ask in plain language.** The `bookkeeping` skill activates on any accounting request:

   ```text
   Which QuickBooks company is active?
   Process the bank feed.
   Record: paid $500 to Office Depot for office supplies with the company credit card.
   Enter the Acme invoice for $2,400, net 30.
   Reconcile the operating account against the August statement.
   Generate a P&L for last month and compare it to the prior month.
   Close the books for June.
   ```

4. **Review what it escalated.** Open Tasks in the DeepLedger portal to approve, recategorize or dismiss items the agent was not sure about. Approved tasks are recorded on the next run.

The plugin itself is thin: a connector to DeepLedger's hosted MCP server and one skill. The skill does not carry procedures; it tells the agent which procedure to pull from the server's `getGuide` tool for each kind of request, so bookkeeping guidance updates on the server without a plugin release. There are no slash commands, agents, hooks or scripts.

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

DeepLedger is listed in the [xAI plugin marketplace](https://github.com/xai-org/plugin-marketplace) as a remote source pinned to a commit of this repository. Inside Grok Build open the extensions modal with `/plugins` (or browse with `/marketplace`), select DeepLedger and install it. Grok Build reads the `.grok-plugin/plugin.json` manifest, the skill and `.mcp.json` directly. Enable and trust the plugin so Grok can connect to the bundled MCP server; the first DeepLedger tool call opens the browser sign-in, and `/mcps` shows the connection.

To test before marketplace approval, install a pinned revision straight from GitHub:

```bash
grok plugin install DeepLedger/deepledger-plugin@<full-commit-sha> --trust
```

Local development: `grok --plugin-dir ./deepledger-plugin`.

### Grok Bot

Settings, Plugins, search for DeepLedger, Add, then complete the sign-in in your browser. Attach the plugin to a task with `@` and ask which QuickBooks company is active before requesting accounting work.

### Sign-in

On first use the host discovers the server's OAuth 2.1 endpoints (authorization code with PKCE, dynamic client registration) and opens a browser sign-in. Sign in with your DeepLedger account and the connection is authorized. DeepLedger personal API keys (`dl_live_...`) are also accepted as Bearer tokens for programmatic clients. The transport is Streamable HTTP at `https://mcp.deepledger.ai/mcp`.

### Network access and credentials

The plugin ships no scripts, binaries, hooks or shell commands. Everything it does goes through the hosted MCP server. The one runtime exception is the attachment upload below, where the server returns a shell-quoted `curl` command for the agent to run. The only network endpoints it reaches are:

| Endpoint | Purpose |
|----------|---------|
| `https://mcp.deepledger.ai/mcp` | MCP server (Streamable HTTP). All QuickBooks reads and writes, guides, tasks, memory, documents and reports. |
| `https://mcp.deepledger.ai/.well-known/oauth-authorization-server`, `/oauth/register`, `/oauth/authorize`, `/oauth/token`, `/oauth/revoke` | OAuth 2.1 discovery, dynamic client registration, authorization code with PKCE, token refresh and revocation. |
| `https://deepledger.ai` | Browser sign-in page opened by the host during authorization. |
| `https://mcp.deepledger.ai/upload-to-qb` | File attachment upload. `qbAttachFile` returns a `curl` command the agent runs to send one user-chosen local file (10 MB limit) to this path with a one-time token; the server forwards it to QuickBooks. It downloads and executes nothing. |

Credentials: a DeepLedger account (OAuth sign-in in the browser, scope `quickbooks`). The host stores the resulting token; the plugin never sees, stores or transmits Intuit credentials, and it reads no local files, environment variables or secrets. QuickBooks access is scoped to the companies the signed-in user can already open in the DeepLedger portal.

## How the skill works

The `bookkeeping` skill activates on any accounting request. It confirms the active QuickBooks company, then calls the server's `getGuide` tool with the guide type that matches the request and follows the returned steps, safety checklist and common mistakes:

| Request | `getGuide` type |
|---------|-----------------|
| Record a payment, bill, invoice, customer payment, refund, credit, deposit, transfer or journal entry; categorize bank feed items; AP and AR writes | `transaction_recording` |
| Close a month, adjusting entries, Close Sheet | `month_end_closing` |
| Reconcile a statement, explain a reconciliation difference | `reconciliation` |
| Report comparisons, ratios, trends, budget variances | `financial_analysis` |
| Audit or review support, evidence index, missing documents | `audit_preparation` |
| A tool failure the agent cannot resolve from the response | `error_recovery` (per tool and error code) |

Master data, agent memory, review tasks, documents and custom reports follow their tool descriptions; any QuickBooks write still goes through the `transaction_recording` protocol.

## What the connector provides

The `.mcp.json` connector points at DeepLedger's official hosted MCP server. It installs no local binary. The server exposes 27 tools; every one operates on the active QuickBooks company and names it in its result.

| Tool | Capability |
|------|------------|
| `qbCompanyProfile` | Show, list or switch the active QuickBooks company |
| `qbMasterData` | Look up, create or update accounts, vendors, customers, items, classes and tax codes |
| `qbFetchTransactions` | Fetch transactions for duplicate checks, payee history and outstanding bills or invoices |
| `qbExpense` | Record or update a purchase paid now (card, ACH, check, cash) |
| `qbBill` | Record or update a vendor bill to pay later |
| `qbBillPayment` | Pay outstanding bills, applying vendor credits |
| `qbInvoice` | Create or update a customer invoice |
| `qbReceivePayment` | Apply a customer payment to outstanding invoices |
| `qbSalesReceipt` | Record a sale paid on the spot |
| `qbDeposit` | Record a bank deposit, batching held payments or direct income lines |
| `qbTransfer` | Move money between the company's own accounts |
| `qbRefundReceipt` | Refund a customer |
| `qbCredit` | Record a vendor or customer credit |
| `qbJournalEntry` | Record a balanced journal entry |
| `qbEstimate` | Create, update or convert an estimate |
| `qbRecurringTransaction` | Manage recurring transaction templates |
| `qbVoidTransaction` | Void an invoice, sales receipt, refund receipt, payment or bill payment |
| `qbAttachFile` | Attach a receipt or document to a QuickBooks record |
| `qbSendEmail` | Email a sales document to the customer through QuickBooks |
| `qbReports` | Run profit and loss, balance sheet, aging and other QuickBooks reports |
| `getGuide` | Load the current bookkeeping procedure or a per-tool error playbook |
| `tasks` | Review tasks shared between the agent and the human reviewer |
| `agentMemory` | Read and maintain durable per-company knowledge |
| `bankFeed` | Fetch unrecorded bank transactions and stamp them once recorded |
| `documents` | Read documents from DeepLedger storage or QuickBooks attachments |
| `customReports` | Run saved report definitions by name |
| `closeRun` | Track the month-end close and its Close Sheet |

## Safety model

Every QuickBooks write follows the server's protocol, carried by the `transaction_recording` guide and the tool descriptions:

1. **Lookup**: `qbMasterData` resolves vendor, customer and account IDs.
2. **Duplicate check**: `qbFetchTransactions` verifies no duplicate exists.
3. **Decide**: the agent records when the payee, category, tool and amount are obvious from the evidence and it is confident; it creates a review task with specific reasoning only when something is genuinely uncertain. Reviewer decisions on tasks are applied verbatim.

Correctness guards do not yield to confidence: an outstanding bill or invoice dictates the tool, journal entries must balance, voids require a fetch-and-verify first, and there is no batch tool: every transaction is recorded individually so each write passes through the full protocol. Vendor and customer categorization is inferred from QuickBooks history at decision time, not from stored mappings, so a recategorization in QuickBooks takes effect on the next transaction.

## Architecture

```
Plugin (this repo)
  skills/bookkeeping/SKILL.md  One skill: confirm company, pull the matching getGuide procedure, verify
  .grok-plugin/plugin.json     Grok Build manifest
  .claude-plugin/plugin.json   Claude Code manifest
  .mcp.json                    MCP connector for Claude Code and Grok Build
  .cursor-plugin/plugin.json   Cursor and Grok Bot manifest
  mcp.json                     MCP connector for Cursor
    | Streamable HTTP + OAuth 2.1
MCP server (hosted at https://mcp.deepledger.ai/mcp)
  27 tools (20 QuickBooks + 7 platform)
  getGuide procedures, tasks, memory, documents, bank feed, custom reports, close runs
```

## Connecting QuickBooks

1. Log in to [deepledger.ai](https://deepledger.ai)
2. Open Settings, QuickBooks
3. Click Connect QuickBooks and authorize access
4. Once connected, the plugin can read and write to that company

## Data handling and support

- Data returned by DeepLedger and QuickBooks is processed in your host session and is subject to the [DeepLedger Terms](https://deepledger.ai/terms) and [Privacy Policy](https://deepledger.ai/privacy).
- Writes go to a live QuickBooks Online company. Review the agent's proposed transaction before approving anything consequential; voids cannot be undone.
- Support: [support@deepledger.ai](mailto:support@deepledger.ai). Plugin issues: [GitHub issues](https://github.com/DeepLedger/deepledger-plugin/issues).

## Version and license

See [CHANGELOG.md](CHANGELOG.md) for release history. MIT License, see [LICENSE](LICENSE).
