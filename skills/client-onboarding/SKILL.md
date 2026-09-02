---
name: client-onboarding
description: Onboard a newly connected client — assess the books, seed policies/patterns/general memory from QuickBooks history with CPA review, and record bootstrap status. Use when the user mentions bootstrapping, onboarding, setting up a new client, or learning from history.
---

# Client Onboarding Skill

Assess a newly connected client's books and seed durable memory (`policies`, `patterns`, `general`) from QuickBooks history. Onboarding is an accelerant, not a prerequisite: categorization is always inferred in realtime from QB history via the consistency rule — never from stored mappings — so the agent is accurate with or without it. What onboarding adds is the context history can't express on its own: client rules, confirmed recurring streams, and durable facts.

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

Activate this skill when the user wants to:
- Onboard or bootstrap a new client
- Set up a newly connected QuickBooks account
- Learn from existing QuickBooks history
- Re-run the onboarding analysis

## Workflow: Client Onboarding (Run Once)

### 1. Pre-check
`agentMemory(operation="read", type="system", title="bootstrap_status")` — skip if already onboarded, unless the user asks to re-run. (Batch writes skip existing org+tag+title rows, so a re-run never duplicates.)

### 2. Assess the books
- `qbMasterData` — chart of accounts, vendors, customers, items; note entity structure, account numbering, class/location usage
- `qbReports` — P&L (current + prior period) and Balance Sheet for a baseline; note uncategorized/Ask-My-Accountant balances and anything unreconciled
- `qbFetchTransactions` — report scans over the last 6 months (matches the consistency-rule window; extend to 12 on request for quarterly/annual patterns)

### 3. Identify memory candidates
From the scans, propose entries — analysis only, no writes yet:

| Type | What qualifies | Example |
|------|----------------|---------|
| `patterns` | Recurring charge/income confirmed **2+ times**: vendor/source, amount range, frequency, QB account | "AWS monthly invoice ~$800 → 6030 Cloud Hosting" |
| `policies` | Consistent treatment backed by QBO evidence (agent-observed), or a rule the CPA states during onboarding | "Owner codes all meals to 6200 Meals & Entertainment (~$400/mo across 6 months)" |
| `general` | Durable client context not in QB or derivable from it | "Fiscal year ends March 31"; "3 locations: Austin, Dallas, Houston" |

The write bar for every candidate: **"Would a CPA want this note 6 months from now?"** Do NOT propose: vendor→account mappings (realtime QB history covers those), one-off transactions, estimates or guesses, standard accounting rules.

### 4. Present to CPA
Show the proposed entries in a table — type, title, evidence — before writing anything. **Wait for CPA confirmation.** The CPA may also dictate explicit rules here (capitalization threshold, standing categorizations); capture them as `policies`.

### 5. Seed memory (batch write)
- Agent-observed entries in one call: `agentMemory(operation="write", entries=[{type, title, content}, ...])` — up to 50 per call; existing rows are skipped and returned in `skipped[]`
- Agent-observed `policies` content must cite the QBO evidence and end with: `✦ AI generated from QBO data — <date>`
- Reviewer-dictated policies go in a **separate** write with `createdBy="reviewer"` (createdBy applies per call, not per entry)

### 6. Mark complete
`agentMemory(operation="write", title="bootstrap_status", content=...)` — auto-routed to the hidden `system` tag. Content: date, entry counts by type, CPA reviewer.

### 7. Handoff summary
Report what was written and skipped, plus books-assessment flags worth immediate attention: uncategorized balances, unreconciled accounts, aged AR/AP concentrations.

## Safety Checklist

- [ ] `bootstrap_status` checked before running — no duplicate onboarding
- [ ] Every proposed entry passes the write bar (CPA-useful in 6 months)
- [ ] No vendor→account mappings written — categorization stays realtime via the consistency rule
- [ ] CPA reviewed and confirmed the proposed entries before the batch write
- [ ] Agent-observed policies cite QBO evidence and carry the AI-generated footer
- [ ] Reviewer-dictated policies written with `createdBy="reviewer"`
- [ ] `bootstrap_status` written on completion

## Common Mistakes to Avoid

- Seeding vendor→account mappings "for accuracy" → the server infers accounts in realtime from QB history; stored mappings go stale and drift from the ledger
- Writing a pattern from a single occurrence → patterns require 2+ confirmed sightings
- Skipping CPA review → unvetted entries pollute the CPA-facing Memory page
- Using legacy memory types (`vendor`, `customer`, `client`, `worklog`) → invalid; the only tags are `patterns`, `policies`, `general`, `system`
- Mixing reviewer-dictated and agent-observed policies in one batch call → `createdBy` is per call; they need separate writes
- Treating onboarding as required before bank-feed processing → the feed categorizes from live history either way; onboarding just adds policy/pattern context
