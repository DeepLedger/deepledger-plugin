# DeepLedger Plugin

AI bookkeeping for QuickBooks Online, packaged as a thin plugin for Claude Code, Cursor, Grok Build and Grok Bot. It contains two things: a connector to DeepLedger's hosted MCP server and one `bookkeeping` skill. The skill does not carry procedures itself; it tells the agent which procedure to pull from the server's `getGuide` tool for each kind of request, so bookkeeping guidance updates on the server without a plugin release. There are no slash commands, agents, hooks or scripts.

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

DeepLedger is listed in the [xAI plugin marketplace](https://github.com/xai-org/plugin-marketplace) as a remote source pinned to a commit of this repository. Inside Grok Build open the extensions modal with `/plugins` (or browse with `/marketplace`), select DeepLedger and install it. Grok Build reads the Claude Code manifest, skill and `.mcp.json` directly. Local development: `grok --plugin-dir ./deepledger-plugin`.

### Grok Bot

Settings, Plugins, search for DeepLedger, Add, then complete the sign-in in your browser. Attach the plugin to a task with `@` and ask which QuickBooks company is active before requesting accounting work.

### Sign-in

On first use the host discovers the server's OAuth 2.1 endpoints (authorization code with PKCE, dynamic client registration) and opens a browser sign-in. Sign in with your DeepLedger account and the connection is authorized. DeepLedger personal API keys (`dl_live_...`) are also accepted as Bearer tokens for programmatic clients. The transport is Streamable HTTP at `https://mcp.deepledger.ai/mcp`.

### Network access and credentials

The plugin ships no scripts, binaries, hooks or shell commands. Everything it does goes through the hosted MCP server. The only network endpoints it reaches are:

| Endpoint | Purpose |
|----------|---------|
| `https://mcp.deepledger.ai/mcp` | MCP server (Streamable HTTP). All QuickBooks reads and writes, guides, tasks, memory, documents and reports. |
| `https://mcp.deepledger.ai/.well-known/oauth-authorization-server`, `/oauth/register`, `/oauth/authorize`, `/oauth/token`, `/oauth/revoke` | OAuth 2.1 discovery, dynamic client registration, authorization code with PKCE, token refresh and revocation. |
| `https://deepledger.ai` | Browser sign-in page opened by the host during authorization. |

Credentials: a DeepLedger account (OAuth sign-in in the browser, scope `quickbooks`). The host stores the resulting token; the plugin never sees, stores or transmits Intuit credentials, and it reads no local files, environment variables or secrets. QuickBooks access is scoped to the companies the signed-in user can already open in the DeepLedger portal.

## Quick start

```
"Which QuickBooks company is active?"
"Record: paid $500 to Office Depot for office supplies with the company credit card"
"Process the bank feed"
"Reconcile the operating account against the August statement"
"Generate a P&L for last month and compare it to the prior month"
"Close the books for June"
```

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
  .claude-plugin/plugin.json   Claude Code and Grok Build manifest
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

## Version and license

See [CHANGELOG.md](CHANGELOG.md) for release history. MIT License, see [LICENSE](LICENSE).
