---
name: financial-analysis
description: Analyze financial performance, cash movements, report comparisons or budget variances using comparable QuickBooks figures and traceable evidence. Use when the user asks for financial analysis, trends, ratios, budget comparisons or explanations of profit and cash changes.
---

# Financial Analysis

Answer the business question with comparable figures, supported drivers and explicit assumptions. Keep analysis read-only unless a specific follow-up action is authorized.

## Select the company

Call `qbCompanyProfile` and identify the active company. If no company is
selected, list accessible companies and switch to the one the user means.
Ask when the choice is ambiguous. If the active company changes during the
workflow, confirm the intended company before a write. Check company context
on tool responses throughout the work.

## Load the current procedure

Before beginning the workflow, call:

```text
getGuide(guideType="financial_analysis")
```

Follow its `steps`, `safetyChecklist` and `commonMistakes`. The server's
published guide is the shared procedure for this skill and other clients;
do not substitute an older remembered copy. Tool schemas remain the authority
for accepted arguments. This skill does not expand the user's request,
transaction approval or the credential's permissions. Unattended credentials
do not gain QuickBooks write access from a guide.

If `success:false`, use the returned `errorCode`. A missing or draft guide
(`GUIDE_NOT_AVAILABLE`) is not an empty checklist. A failed or invalid read
(`GUIDE_FETCH_FAILED`) may be retried; if it persists, explain that the
procedure is unavailable and hand off dependent work. Do not invent the
missing procedure or loop on the same failure.

## Recover and verify

For a tool failure you cannot resolve from its response, call
`getGuide(guideType="error_recovery", tool="<failed tool>", errorCode="<returned code>")`.
If no code was returned, use its refusal message. Follow the matching action,
verification and stop condition. Message-based matches are candidates; check
which condition actually applies. Read back an uncertain write before retrying.

Report confirmed outcomes and unresolved tasks. A tool call alone is not
proof that the intended transaction, attachment or close state was saved.
