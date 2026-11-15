# DeepLedger

**The accounting harness for AI**

DeepLedger connects QuickBooks Online, QuickBooks Desktop (beta), Xero, Bill.com and Stripe (for reporting) to Claude, ChatGPT, Cursor, Grok and other AI agents through one MCP address. Record bills, invoices, payments, journal entries and other transactions, run reports, reconcile and close the month across every company you manage, with your firm's rules, review tasks and audit history built in.

This repository packages DeepLedger as a plugin for Claude Code, Cursor and Grok Build. Install it, sign in with your DeepLedger account, and ask for bookkeeping in plain language. The AI agent works the active company the way a staff accountant would: it looks up the payee, checks for duplicates, picks the right transaction type, records it, and hands anything it is not sure about to a human reviewer instead of guessing.

The ledger stays the record. DeepLedger holds the connections your company authorized (QuickBooks through Intuit's own OAuth flow, Xero through Xero's own sign-in, QuickBooks Desktop through the QuickBooks Web Connector on your own machine, Bill.com with your own BILL credentials, Stripe through Stripe's official MCP), so the plugin never sees those credentials. One sign-in reaches every company you can open in the DeepLedger portal.

## What it connects

| App | What the AI agent can do |
|-----|--------------------------|
| QuickBooks Online | Read and write: transactions, master data, reports, projects, custom fields, recurring templates, bank feed, attachments |
| QuickBooks Desktop (beta) | QuickBooks Desktop Enterprise (US edition) company files, through the QuickBooks Web Connector on the client's Windows machine. Read and write: lists, transactions and reports; create or update list records, bills, vendor credits, purchase orders, item receipts, checks, card charges and credits, invoices, sales receipts, credit memos, estimates, sales orders, statement charges, received payments, bill payments, deposits, journal entries, inventory adjustments, builds and stock transfers, time and mileage, and custom fields; void or delete a transaction; delete, merge or inactivate a list record. The AI agent is instructed to run each write as a dry run and ask you to confirm before it is sent. Transactions, reports and every write need that machine on and connected |
| Xero | Read and write: invoices, bills, credit notes, payments, spend and receive money, transfers, manual journals, quotes, repeating invoices, tracking categories, reports. New invoices, bills, credit notes and journals are saved as drafts by default |
| Bill.com | Read bills, invoices, vendor credits, credit memos, payments, aging, vendors, customers and accounts; save drafts and edit vendors, customers, bills and open invoices. It never pays, approves or sends anything in BILL |
| Stripe (for reporting) | Read only: balance, payouts, charges, refunds, disputes, invoices, customers and subscriptions. It never refunds, pays or changes anything in Stripe |

A company's books are on QuickBooks Online, QuickBooks Desktop or Xero. Bill.com and Stripe connect next to the ledger, on the company's Connections tab in the DeepLedger portal.

## Who it is for

- **Accounting firms and bookkeepers** managing many client companies. One connection switches between companies on any ledger, every write follows the same checks, and uncertain items land in Tasks with the agent's reasoning attached.
- **In-house finance teams** running several companies or entities, who want the month-end close, reconciliation prep and management reports drafted for review.
- **Small business owners** who keep their own books in QuickBooks Online, QuickBooks Desktop or Xero and want the routine entries, categorization and reports done for them.

## What it does

| Area | What the AI agent does |
|------|------------------------|
| Bank feed | Pulls unrecorded bank and card transactions, categorizes them from the company's rules and history, records the clear ones and creates review tasks for the rest |
| Recording | Records expenses, bills, invoices, payments, deposits, transfers, refunds, credits and journal entries with an ID lookup and a duplicate check before every write |
| Payables and receivables | Enters bills and pays them against outstanding balances in the ledger, invoices customers and applies their payments, applies credits, reports aging |
| Reconciliation | Matches a bank or card statement to the ledger and prepares the reconciliation for your review |
| Month-end close | Works the close checklist, proposes adjusting entries, and hands the close to a reviewer for sign-off |
| Reports and analysis | Runs P&L, balance sheet, aging and saved reports, compares periods and explains what moved; builds dashboards across companies |
| Audit preparation | Builds supporting schedules, an evidence index and a missing-document list |
| AI Brain | Reads and maintains your firm's and each client's rules and policies; where they differ, the client's rule wins |
| Human review | Anything uncertain becomes a task with the agent's reasoning; a reviewer signs off |

## How to use it

1. **Connect your companies** at [deepledger.ai](https://deepledger.ai): each company's QuickBooks Online, QuickBooks Desktop or Xero books, then Bill.com or Stripe on its Connections tab if you use them.
2. **Install the plugin** in your host (below) and sign in when the browser opens.
3. **Ask in plain language.** The `bookkeeping` skill activates on any accounting request:

   ```text
   Which company is active?
   Switch to Harbour Cafe.
   Process last week's bank feed.
   Record: paid $500 to Office Depot for office supplies with the company credit card.
   Enter the Acme invoice for $2,400, net 30.
   Reconcile the operating account against the October statement.
   Compare the P&L for October against September.
   Close the books for October.
   ```

4. **Review what it escalated.** Open Tasks in the DeepLedger portal to approve, correct or dismiss items the agent was not sure about. The agent records approved work when you next ask it to.

The plugin itself is thin: a connector to DeepLedger's hosted MCP server and one skill. The skill does not carry procedures; it tells the agent to read the company's own playbook from the server's `skills` tool for each kind of request, so bookkeeping guidance updates on the server, and per company, without a plugin release. There are no slash commands, agents, hooks or scripts.

## Prerequisites

- A DeepLedger account with at least one company connected at [deepledger.ai](https://deepledger.ai), on QuickBooks Online, QuickBooks Desktop or Xero
- One of the hosts below

## Install

Marketplace availability depends on each marketplace's review. Until a listing appears, use the local install for that host. Setup guides for every AI app: [deepledger.ai/integrations](https://deepledger.ai/integrations).

### Claude Code

```bash
claude plugin marketplace add DeepLedger/deepledger-plugin
claude plugin install deepledger@deepledger-marketplace
```

Local development: `claude --plugin-dir ./deepledger-plugin`. Guide: [Connect Claude Code](https://deepledger.ai/blog/connect-quickbooks-to-claude-code).

### Cursor

Clone this repository and symlink it into Cursor's local plugins folder, then reload Cursor.

```bash
git clone https://github.com/DeepLedger/deepledger-plugin.git
mkdir -p ~/.cursor/plugins/local
ln -s "$(pwd)/deepledger-plugin" ~/.cursor/plugins/local/deepledger
```

Guide: [Connect Cursor](https://deepledger.ai/blog/connect-quickbooks-to-cursor).

### Grok Build

DeepLedger is submitted to the [xAI plugin marketplace](https://github.com/xai-org/plugin-marketplace) as a remote source pinned to a commit of this repository. Once it is listed, open the extensions modal inside Grok Build with `/plugins` (or browse with `/marketplace`), select DeepLedger and install it. Grok Build reads the `.grok-plugin/plugin.json` manifest, the skill and `.mcp.json` directly. Enable and trust the plugin so Grok can connect to the bundled MCP server; the first DeepLedger tool call opens the browser sign-in, and `/mcps` shows the connection.

To install before the listing, pin a revision straight from GitHub:

```bash
grok plugin install DeepLedger/deepledger-plugin@<full-commit-sha> --trust
```

Local development: `grok --plugin-dir ./deepledger-plugin`.

### Grok chat and Grok Bot

These do not use this plugin; they connect to the same MCP address directly. Steps for both: [Connect Grok and Grok Bot](https://deepledger.ai/blog/connect-grok-to-quickbooks).

- **Grok chat:** open grok.com/connectors, choose New Connector, then Custom, paste `https://mcp.deepledger.ai/mcp` and sign in with your DeepLedger account.
- **Grok Bot:** DeepLedger is not in the Grok Bot plugin catalog. In a Grok Bot chat, ask it to add DeepLedger as a custom MCP server at `https://mcp.deepledger.ai/mcp`, then complete the sign-in.

### Sign-in

On first use the host discovers the server's OAuth 2.1 endpoints (authorization code with PKCE, dynamic client registration) and opens a browser sign-in. Sign in with your DeepLedger account and the connection is authorized. DeepLedger personal API keys (`dl_live_...`) are also accepted as Bearer tokens for programmatic clients. The transport is Streamable HTTP at `https://mcp.deepledger.ai/mcp`.

### Network access and credentials

The plugin ships no scripts, binaries, hooks or shell commands. Everything it does goes through the hosted MCP server. The one runtime exception is the attachment upload below, where the server returns a shell-quoted `curl` command for the agent to run. The only network endpoints it reaches are:

| Endpoint | Purpose |
|----------|---------|
| `https://mcp.deepledger.ai/mcp` | MCP server (Streamable HTTP). All ledger and connected-app reads and writes, skills, tasks, AI Brain, documents and reports. |
| `https://mcp.deepledger.ai/.well-known/oauth-authorization-server`, `/.well-known/oauth-protected-resource`, `/oauth/register`, `/oauth/authorize`, `/oauth/token`, `/oauth/revoke` | OAuth 2.1 discovery, dynamic client registration, authorization code with PKCE, token refresh and revocation. |
| `https://deepledger.ai` | Browser sign-in page opened by the host during authorization. |
| `https://mcp.deepledger.ai/upload-to-qb` | QuickBooks file attachment. `documents` (operation `upload`) returns a `curl` command the agent runs to send one user-chosen local file (10 MB limit) to this path with a one-time token; the server forwards it to QuickBooks. It downloads and executes nothing. |
| `https://xero.deepledger.ai/upload-to-xero` | The same attachment upload for a Xero company (10 MB limit). |

Credentials: a DeepLedger account (OAuth sign-in in the browser; the host asks for the scopes the server advertises). The host stores the resulting token; the plugin never sees, stores or transmits QuickBooks, Xero, BILL or Stripe credentials, and it reads no local files, environment variables or secrets. Access is scoped to the companies the signed-in user can already open in the DeepLedger portal, at that user's access level (read only, or read and write).

## How the skill works

The `bookkeeping` skill activates on any accounting request. It confirms the active company with `companies`, then reads the matching skill with `skills` and follows its steps and safety rules:

| Request | Skill (default name) |
|---------|----------------------|
| Record a payment, bill, invoice, refund, credit, deposit, transfer or journal entry; categorize bank feed items; payables and receivables writes | `record-transactions` |
| Close a month, adjusting entries | `month-end-close` |
| Reconcile a statement, explain a reconciliation difference | `reconcile-bank` |
| Audit or review support, evidence index, missing documents | `audit-preparation` |
| First-time setup of a company | `get-started` |
| A tool failure the agent cannot resolve from the response | `skills` operation `errors` (per tool and error code) |

Each company starts from DeepLedger's default skills and can keep its own preferences. Master data, the AI Brain, review tasks, documents and saved reports follow their tool descriptions; any write to the books still goes through the recording skill.

## What the connector provides

The `.mcp.json` connector points at DeepLedger's official hosted MCP server. It installs no local binary. Signed-in users get the DeepLedger tools (a user whose companies are all on QuickBooks Desktop gets a shorter set: `companies`, `connections`, `describeTool`, `callReadTool`, `callWriteTool` and `tasks`); each app's own tools are listed for users whose companies have that app connected. Every tool that touches the books operates on the active company and names it in its result.

**DeepLedger (every user)**

| Tool | Capability |
|------|------------|
| `companies` | Show, list or switch the active company on any ledger; send feedback to DeepLedger |
| `connections` | Which apps each company has connected, which need a reconnect, and each app's tools |
| `dashboard` | Read-only dashboard numbers across companies: firm summary, company overview, close board |
| `describeTool` | Full description and input schema of any connected app's tool |
| `callReadTool` | Run a read with any connected app's tool, including tools not in your list yet |
| `callWriteTool` | Run a write with any connected app's tool, with the same checks as calling it directly |
| `tasks` | Review tasks shared between the agent and the human reviewer |
| `agentMemory` | Read and maintain the company's AI Brain: procedures, policies and vendor and customer rules |
| `documents` | Read documents from DeepLedger storage or ledger attachments, and upload a file to a ledger record |
| `closeRun` | Track the month-end close |
| `closeTemplate` | Propose the company's close checklist, approved in the portal |
| `reconciliation` | Prepare a bank or card reconciliation, finished in the ledger |
| `customReports` | Run saved report definitions by name |
| `skills` | The company's own playbook for each job, and error recovery per tool |
| `getGuide` | Earlier name for skill reads, kept for compatibility |

**QuickBooks Online**

| Tool | Capability |
|------|------------|
| `qbCompanyProfile` | Show, list or switch the active QuickBooks company |
| `qbMasterData` | Look up, create or update accounts, vendors, customers, items, classes and tax codes |
| `qbFetchTransactions` | Fetch transactions for duplicate checks, payee history and outstanding bills or invoices |
| `qbExpense` | Record or update a purchase paid now (card, ACH, check, cash) |
| `qbBill` | Record or update a vendor bill to pay later |
| `qbPayment` | Apply a customer payment to outstanding invoices, or pay outstanding bills applying vendor credits |
| `qbInvoice` | Create or update a customer invoice |
| `qbSalesReceipt` | Record a sale paid on the spot, or refund a customer |
| `qbDeposit` | Record a bank deposit, batching held payments or direct income lines |
| `qbTransfer` | Move money between the company's own accounts in the ledger |
| `qbCredit` | Record a vendor or customer credit |
| `qbJournalEntry` | Record a balanced journal entry |
| `qbEstimate` | Create, update or convert an estimate |
| `qbRecurringTransaction` | Manage recurring transaction templates |
| `qbProject` | List, create, update and close QuickBooks projects |
| `qbCustomization` | Manage custom field definitions and transaction dimensions |
| `qbVoidTransaction` | Void or delete a transaction, after confirmation |
| `qbSendEmail` | Email a sales document to the customer through QuickBooks |
| `qbReports` | Run profit and loss, balance sheet, aging and other QuickBooks reports |
| `bankFeed` | Fetch unrecorded bank transactions and mark them once recorded |

**QuickBooks Desktop (beta)**

Listed for users whose companies are all on QuickBooks Desktop; next to QuickBooks Online or Xero companies, reach them through `describeTool` and `callReadTool` / `callWriteTool`. The AI agent is instructed to run each write as a dry run (`dryRun`) and ask you to confirm before it is sent.

| Tool | Capability |
|------|------------|
| `qbdesktopCompany` | The connected company file: name, fiscal year, features switched on |
| `qbdesktopMasterData` | Look up every list (accounts, customers and jobs, vendors, items, classes and the reference lists); create or update list records |
| `qbdesktopFetchTransactions` | Read every kind of transaction live from QuickBooks |
| `qbdesktopReports` | Profit and loss, balance sheet, trial balance, general ledger, aging and other QuickBooks Desktop reports |
| `qbdesktopSync` | How current the copy of the lists is; refresh it |
| `qbdesktopPurchases` | Bills, vendor credits, purchase orders, item receipts, checks and card charges |
| `qbdesktopSales` | Invoices, sales receipts, credit memos, estimates, sales orders and statement charges |
| `qbdesktopPayments` | Received payments, bill payments and deposits (records that money moved; moves none) |
| `qbdesktopJournalEntry` | General journal entries; debits must equal credits |
| `qbdesktopInventory` | Adjust stock, build assemblies, transfer stock between sites |
| `qbdesktopTime` | Time entries and mileage |
| `qbdesktopCustomization` | Custom field values and definitions |
| `qbdesktopVoidOrDelete` | Void or delete a transaction; delete, merge or inactivate a list record |

**Xero**

| Tool | Capability |
|------|------------|
| `xeroOrganisation` | The active Xero organisation's details |
| `xeroMasterData` | Look up accounts, contacts, items, tax rates, tracking categories and currencies; create or update accounts, contacts and items |
| `xeroFetchTransactions` | Search transactions for duplicate checks, history and outstanding documents |
| `xeroInvoice` | Sales invoices |
| `xeroBill` | Bills |
| `xeroCreditNote` | Credit notes |
| `xeroPayment` | Payments against invoices, bills and credit notes |
| `xeroBankTransaction` | Spend and receive money |
| `xeroBankTransfer` | Transfers between the organisation's bank accounts |
| `xeroManualJournal` | Manual journals |
| `xeroQuote` | Quotes |
| `xeroRepeatingInvoice` | Repeating invoices |
| `xeroTrackingCategory` | Tracking categories |
| `xeroReports` | Profit and loss, balance sheet, aging and other Xero reports |
| `xeroBanking` | Read-only bank view: bank accounts, balances and unreconciled ledger items |

**Bill.com** (your own BILL credentials; never pays, approves or sends)

| Tool | Capability |
|------|------------|
| `billcomMasterData` | Read vendors, customers, accounts and classifications; create, update or archive vendors and customers |
| `billcomFetchTransactions` | Read bills, vendor credits, payments made and received, invoices, credit memos and AP and AR aging |
| `billcomBill` | Save bills unapproved and unpaid, edit their dates, numbers and descriptions, and record vendor credits |
| `billcomInvoice` | Save invoices without sending them, update them, and record credit memos |

**Stripe** (for reporting, read only)

| Tool | Capability |
|------|------------|
| `stripemcpRead` | Read the connected Stripe account through Stripe's official MCP: balance, payouts, charges, refunds, disputes, invoices, customers and subscriptions |

## Safety model

Every write to the books follows the server's checks, carried by the recording skill and the tool descriptions:

1. **Lookup**: the ledger's master-data tool resolves vendor, customer and account IDs.
2. **Duplicate check**: the ledger's fetch tool verifies no duplicate exists.
3. **Decide**: the agent records when the payee, category, tool and amount are clear from the evidence and it is confident; it creates a review task with specific reasoning when something is uncertain. Reviewer decisions on tasks are applied as decided.

Correctness guards do not yield to confidence: an outstanding bill or invoice dictates the tool, journal entries must balance, and voids and deletes need a fetch, a check and the user's confirmation first. Each write must name the company it is meant for, so a company switch in another chat cannot send a write to the wrong books. Every tool call is recorded in the company's Audit history. Read-only users cannot change the books.

DeepLedger has no tool that sends a payment or moves money from a bank account: in Bill.com it reads, drafts and edits, and never pays, approves or sends; Stripe is read only.

## Architecture

```
Plugin (this repo)
  skills/bookkeeping/SKILL.md  One skill: confirm the company, read the matching company skill, verify
  .grok-plugin/plugin.json     Grok Build manifest
  .claude-plugin/plugin.json   Claude Code manifest
  .claude-plugin/marketplace.json  Claude Code marketplace entry
  .mcp.json                    MCP connector for Claude Code and Grok Build
  .cursor-plugin/plugin.json   Cursor manifest
  mcp.json                     MCP connector for Cursor
    | Streamable HTTP + OAuth 2.1
MCP server (hosted at https://mcp.deepledger.ai/mcp)
  DeepLedger tools for every user, plus the tools of each connected app:
  QuickBooks Online, QuickBooks Desktop (beta), Xero, Bill.com, Stripe (for reporting)
```

## Data handling and support

- Data returned by DeepLedger and your connected apps is processed in your host session and is subject to the [DeepLedger Terms](https://deepledger.ai/terms) and [Privacy Policy](https://deepledger.ai/privacy). Security: [deepledger.ai/security](https://deepledger.ai/security).
- Writes go to a live ledger. Review the agent's proposed transaction before approving anything consequential; voids and deletes cannot be undone.
- Support: [hello@deepledger.ai](mailto:hello@deepledger.ai). Plugin issues: [GitHub issues](https://github.com/DeepLedger/deepledger-plugin/issues).

## Version and license

See [CHANGELOG.md](CHANGELOG.md) for release history. MIT License, see [LICENSE](LICENSE).
