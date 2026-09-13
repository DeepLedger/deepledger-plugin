# Changelog

## [2.2.0] - 2026-09-12

### Added

- Audit preparation skill loads the shared evidence-packet procedure.

### Changed

- Reconciliation and financial analysis now load current Supabase-backed workflows through `getGuide`.
- Reconciliation distinguishes workbook preparation from QuickBooks UI matching and finalization, including authorized browser/computer-use assistance and saved-report verification.

## [2.1.1] - 2026-09-12

### Changed

- Recording and month-end close skills load their current procedures through `getGuide` instead of maintaining separate workflow copies.
- Both skills explain guide-read failures, targeted error recovery, verification, company checks and credential limits. Close adjustments remain proposed until approved, with human final sign-off.

## [2.0.0] - 2026-07-18

### Removed
- **All 18 slash commands** and **both agents** (Accountant, CFO) — the plugin is now skills + hooks + the DeepLedger MCP connector only. Natural-language requests trigger the matching skill directly; the commands duplicated what the skills already cover.

### Fixed
- **financial-analysis skill**: added the missing YAML frontmatter (name + description) so the skill is actually discoverable.

### Changed
- README rewritten around skills-first usage; architecture section updated to the current server tool set (23 tools).

## [1.3.0] - 2026-04-01

### Changed
- **Hooks completely redesigned** — from 5 basic guards to 10 comprehensive validators that catch real failure modes

### Added — New Hooks
- **`duplicate-result-guard`**: Verifies qbFetchTransactions returned ZERO matches before allowing writes. Prior hook only checked the call happened — this checks the result was clean. On violation: stops and shows potential duplicates to user.
- **`expense-type-guard`**: Catches expense-side type errors — Expense when outstanding Bill exists (should be BillPayment), Deposit when outstanding Invoice exists (should be ReceivePayment).
- **`income-type-guard`**: Catches income-side type errors — SalesReceipt when outstanding Invoice exists (should be ReceivePayment), Invoice when customer already paid (should be SalesReceipt), ReceivePayment when no Invoice exists (should be Deposit or SalesReceipt). Full AR cycle validation.
- **`vendor-resolution-guard`**: Cross-references vendorId/customerId AND accountIds against qbMasterData results. Catches hallucinated IDs, copy-paste errors, and wrong-vendor selection.
- **`amount-anomaly-guard`**: Checks transaction amount against learned vendor amount range (from bootstrap or history). Flags if 3x outside average or below 1/3 of minimum.
- **`source-category-collision-guard`**: Blocks when source account (bank/CC) equals a line item account — a zero-net transaction that breaks reconciliation.

### Upgraded — Existing Hooks
- **`journal-entry-balance-enforcer`** (was `journal-entry-balance-check`): Upgraded from advisory reminder to hard block. Sums debits and credits to 2 decimal places and blocks if unequal.
- **`void-transaction-guard`**: Added cross-reference check — transaction ID being voided must appear in the most recent qbFetchTransactions results.
- **`batch-safety-guard`**: Added type homogeneity check (no mixed types in one batch) and duplicate check requirement.
- **`bank-feed-flag-quality`**: Added aiReasoning quality enforcement — rejects generic flags ("not sure", "needs review") and requires specific context with examples.

## [1.2.0] - 2026-04-01

### Added
- **`/bootstrap` command**: First-time client onboarding — reads 12 months of QB history, extracts vendor/customer/account mappings, presents summary to CPA for review, seeds agent memory with upvote cap of 5. Supports review, status, and reset modes.
- **Bootstrap workflow in bookkeeping skill**: Full extract → analyze → present → seed → mark workflow
- **Bootstrap detection in `/loop` and `/bank-feed`**: Auto-warns if client hasn't been bootstrapped, recommends running `/bootstrap` first
- **Amount range anomaly detection**: Bootstrap stores min/max/avg per vendor — flags transactions 3x outside learned range
- **Memory lifecycle documentation**: Bootstrap (cap 5) → real-time upvotes (+1) → CPA corrections override

### Changed
- **Accountant agent**: Added bootstrap as core responsibility #1, expanded memory section with lifecycle and bootstrap vs real-time distinction
- **README**: Added bootstrap to quick-start flow, command table, and agent memory section

## [1.1.0] - 2026-04-01

### Changed
- **Transport**: Switched from SSE to HTTP Streamable (`/sse` → `/mcp`) — the Anthropic-recommended transport for remote MCP servers

### Added
- **README**: Setup guide, quick-start, command reference, architecture overview, memory schema docs
- **`/budget` command**: Budget vs actuals comparison with variance analysis and budget creation
- **`/recurring` command**: List, create, pause, resume, and delete recurring transactions
- **Error recovery in `/loop`**: Crash resume via `lastCompletedStep` tracking, per-item failure isolation (log and continue), retry limits (3 attempts before flagging), worklog schema documentation
- **Batch operation workflow**: Step-by-step batch recording guide in bookkeeping skill with example payload and error handling
- **Corrections & reversals**: Void-and-rerecord vs reversing JE guidance with decision criteria in accountant agent and bookkeeping skill
- **Recurring transaction management**: Guidance in accountant agent for automated vs reminder types
- **Agent memory schema**: Documented types (vendor, customer, client, worklog), naming conventions, and upvote lifecycle in README

### Fixed
- Plugin was pointing at removed `/sse` endpoint — now uses the live `/mcp` endpoint

## [1.0.0] - 2026-03-21

### Added
- Initial release
- Accountant and CFO agents
- Bookkeeping and Financial Analysis skills
- 11 commands: /record, /bank-feed, /find, /transfer, /pnl, /balance-sheet, /cash-flow, /aging, /close-books, /health-check, /loop
- 5 safety hooks: write-safety-guard, void-transaction-guard, batch-safety-guard, bank-feed-flag-quality, journal-entry-balance-check
