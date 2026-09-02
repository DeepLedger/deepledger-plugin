---
name: financial-analysis
description: Generate financial reports, analyze business health, compute ratios, identify trends, and provide actionable CFO-level insights. Use when the user mentions P&L, balance sheet, cash flow, financial health, ratios, trends, or budget vs actuals.
---

# Financial Analysis Skill

Expertise in generating financial reports, analyzing business health, computing ratios, identifying trends, and providing actionable CFO-level insights.

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
- Generate a P&L, Balance Sheet, Cash Flow, or aging report
- Analyze profitability, margins, or revenue trends
- Calculate financial ratios (current ratio, DSO, debt-to-equity, etc.)
- Assess cash runway or cash flow health
- Compare performance across periods
- Check customer concentration or vendor spending patterns
- Run a financial health check

## Report Types Available

| Report | Tool Call | Purpose |
|--------|-----------|---------|
| Profit & Loss | `qbReports(reportType="ProfitAndLoss")` | Revenue, expenses, net income |
| P&L by Class | `qbReports(reportType="ProfitAndLossByClass")` | P&L with one column per class |
| P&L Detail | `qbReports(reportType="ProfitAndLossDetail")` | Transaction-level drill-down |
| Balance Sheet | `qbReports(reportType="BalanceSheet")` | Assets, liabilities, equity |
| Cash Flow | `qbReports(reportType="CashFlow")` | Operating, investing, financing flows |
| Aged Receivables | `qbReports(reportType="AgedReceivables")` | Who owes you, how overdue |
| Aged Receivables Detail | `qbReports(reportType="AgedReceivablesDetail")` | Invoice-level AR aging |
| Aged Payables | `qbReports(reportType="AgedPayables")` | What you owe, when due |
| Aged Payables Detail | `qbReports(reportType="AgedPayablesDetail")` | Bill-level AP aging |
| Trial Balance | `qbReports(reportType="TrialBalance")` | All accounts, debits = credits |
| Sales by Customer | `qbReports(reportType="SalesByCustomer")` | Revenue by customer |
| Customer Income | `qbReports(reportType="CustomerIncome")` | Customer profitability |
| Customer Balance | `qbReports(reportType="CustomerBalance")` | Total AR owed per customer |
| Sales by Product | `qbReports(reportType="SalesByProduct")` | Product/service revenue |
| Vendor Expenses | `qbReports(reportType="VendorExpenses")` | Spending by vendor |
| Vendor Balance | `qbReports(reportType="VendorBalance")` | Total AP owed per vendor |
| Transaction List | `qbReports(reportType="TransactionList")` | All transactions in a period |
| General Ledger | `qbReports(reportType="GeneralLedger")` | Per-account transaction history |
| Budget vs Actuals | `qbReports(reportType="BudgetVsActuals")` | Actual vs budget vs variance per account |

## Workflow: Full Financial Analysis

Follow the `getGuide(guideType="financial_analysis")` workflow:

### Step 1: Pull Core Financial Reports
```
qbReports → P&L (current month, QTD, YTD with prior period comparison)
qbReports → Balance Sheet (current date with prior month comparison)
qbReports → Cash Flow (current month and YTD)
```

Use `summarizeBy: "Month"` for trend visibility.
Keep `accountingMethod` consistent across all reports.

### Step 2: Analyze Profitability
Calculate from P&L data:
- **Gross Margin** = (Revenue - COGS) / Revenue
- **Operating Margin** = Operating Income / Revenue
- **Net Margin** = Net Income / Revenue

Drill deeper with:
- `SalesByCustomer` → revenue concentration risk
- `SalesByProduct` → which offerings are most profitable
- `VendorExpenses` → spending pattern analysis

### Step 3: Review Cash Position
From Cash Flow statement:
- Separate operating, investing, and financing flows
- **Cash Runway** = Cash on Hand / Average Monthly Cash Burn
- Review AR aging for incoming cash timing
- Review AP aging for upcoming obligations

Red flags:
- Operating cash flow negative while P&L shows profit → investigate AR
- Cash runway < 3 months → critical alert

### Step 4: Compute Key Ratios

**Liquidity:**
- Current Ratio = Current Assets / Current Liabilities (target > 1.5)
- Quick Ratio = (Current Assets - Inventory) / Current Liabilities

**Efficiency:**
- DSO = AR / (Annual Revenue / 365) — rising DSO = collection problems
- DPO = AP / (Annual COGS / 365)

**Leverage:**
- Debt-to-Equity = Total Liabilities / Equity

### Step 5: Identify Trends & Anomalies

Flag these conditions:
- Revenue declining 3+ consecutive months
- Expense categories growing faster than revenue
- Margin compression vs prior periods
- Top customer > 30% of revenue (concentration risk)
- DSO increasing month-over-month
- Cash runway < 3 months

Use `qbReports(reportType="GeneralLedger")` or `qbFetchTransactions` on the relevant accounts to spot statistical outliers (amounts far outside the account's normal range).

### Step 6: Synthesize & Present

Structure the output:
1. **Headline** — One sentence: overall health
2. **Key Metrics Table** — Numbers with period-over-period comparison
3. **Trends** — Improving / declining / stable with percentages
4. **Risks** — Issues requiring attention
5. **Recommendations** — Specific, actionable next steps

Save key insights to `agentMemory` for longitudinal tracking.

## Presentation Rules

- Never present a number without comparison context
- Lead with the most impactful finding
- Use percentages for changes, not just absolute numbers
- Need 3+ data points to call something a "trend"
- Account for seasonality before flagging declines
- Never mix Accrual and Cash basis between reports
- Never compare a partial month to a full month

## Health Check Quick Mode

For a fast health check:
1. Scan all bank + CC accounts with `qbFetchTransactions` (duplicates, uncategorized entries) and `qbReports(reportType="GeneralLedger")` (outliers)
2. `qbReports(reportType="ProfitAndLoss")` for current month
3. `qbReports(reportType="AgedReceivables")` for overdue AR
4. `qbReports(reportType="AgedPayables")` for upcoming AP

Present: health scores, headline P&L, overdue amounts, and any flags.
