# USE_CASES.md — byollm-finance-harness

How finance professionals use this harness. Written from the outside in — what the user experiences, not how the system works.

---

## The Primary Persona

**Albert Lee — FP&A Manager, mid-market FMCG company**

Albert runs monthly variance analysis for a company with operations in 8 countries and 3 business units. He's fluent in Excel and PowerPoint. He's used Claude and ChatGPT. He's gotten impressive outputs and embarrassing outputs in equal measure. He doesn't debug prompt chains. He doesn't write Python. He needs numbers he can put his name on before Monday's CFO call.

Albert is the person this harness was built for.

---

## User Roles

Three roles. Each scopes what a user can run.

| Role | Can Run | Cannot Run |
|---|---|---|
| `analyst` | `validate_financial_data`, `compute_variances`, `rank_variance_drivers` | EBITDA bridge, commentary, executive summary, export |
| `controller` | All 9 skills | — |
| `cfo` | `flash_report` workflow only | Deep-dive steps (by preference, not restriction) |

Role is set in `config/roles.yaml` per user. The sequencer enforces it. The user doesn't see a permission error — they see only the workflows and steps available to their role.

---

## Use Case 1: Monthly Variance Analysis

**Who:** Albert Lee (controller role)
**When:** 3rd business day of the month, after GL close
**Workflow:** `variance_analyst`

### What Albert does

He opens Claude Desktop. He drops 6 files into the conversation:

```
FMCG_GL_Actuals_Jan2026.xlsx
FMCG_COGS_Manufacturing_Jan2026.xlsx
FMCG_FX_Rates_Jan2026.xlsx
FMCG_FY2026_Budget_USD.xlsx
FMCG_Operating_Expenses_Jan2026.xlsx
FMCG_Sales_by_SKU_Jan2026.xlsx
```

He types: **"Run the variance analyst for January."**

### What he sees

The harness works through 9 steps. He watches progress in real time:

```
✓  Step 1 of 9 — Data validated. 2 warnings found.
   → BRL and INR rates missing from FX file. Fallback rates applied per config.
   → COGS file missing Country dimension. Allocated by GL proportions.

✓  Step 2 of 9 — Variances computed. Period: Jan 2026.

✓  Step 3 of 9 — Top variance drivers ranked by materiality.

✓  Step 4 of 9 — EBITDA bridge generated.

✓  Step 5 of 9 — Traffic light dashboard complete. 3 red cells.

✓  Step 6 of 9 — Management commentary written.

✓  Step 7 of 9 — Executive summary written.

✓  Step 8 of 9 — Reconciliation passed. Revenue − COGS − OpEx = EBITDA ✓

   Confidence: 94%  |  2 methodology notes  |  0 blocking issues

✓  Step 9 of 9 — Report exported.
```

He downloads `variance_report_jan2026.html`. Five tabs: Glance, Exec, Deep Dive, Commentary, Data Quality. He sends it to the CFO.

### What the harness did that Albert didn't have to

- Applied FX fallback rates for BRL and INR automatically, logged it
- Allocated COGS to countries using GL proportions
- Checked that Revenue − COGS − OpEx = EBITDA before the report rendered
- Structured every commentary section as What / So What / Now What
- Blocked the report from rendering until reconciliation passed

---

## Use Case 2: The Gate Catches a Blocking Error

**Who:** Albert Lee (controller role)
**Scenario:** The GL file is incomplete. EBITDA doesn't reconcile.

### What Albert sees

```
✓  Step 1 of 9 — Data validated. 1 warning found.
✓  Step 2 of 9 — Variances computed.
✓  Step 3 of 9 — Top variance drivers ranked.
✓  Step 4 of 9 — EBITDA bridge generated.

✗  Step 5 of 9 — Reconciliation failed. Report blocked.

   Revenue ($142.3M) − COGS ($89.1M) − OpEx ($31.4M) = $21.8M
   Reported EBITDA: $18.2M
   Gap: $3.6M — exceeds materiality threshold ($0.5M)

   Likely cause: GL file missing logistics entries for MX and IN.
   Suggested fix: Re-export GL actuals including cost centre 7xxx entries.
```

The report does not render. Albert gets a specific, actionable error — not a warning on a document he's already sent.

---

## Use Case 3: CFO Flash Report

**Who:** Sarah Chen (CFO role)
**When:** Monday morning, 10 minutes before board pre-read
**Workflow:** `flash_report`

### What Sarah does

She opens Claude Desktop. She types: **"Flash report for January."**

She doesn't upload files. The `variance-watcher` agent already ran at 6am and cached the validated dataset for the current period.

### What she sees

30 seconds later:

```
January 2026 — Flash Report

Revenue       $142.3M   vs $138.1M budget   +$4.2M  F   +3.0%
Gross Margin   $53.1M   vs $51.2M budget     +$1.9M  F   +3.7%
GM Rate         37.3%   vs   37.1% budget    +0.2pp  F
OpEx           $31.4M   vs $29.8M budget     −$1.6M  U   −5.4%
EBITDA         $21.8M   vs $21.4M budget     +$0.4M  F   +1.9%

30-Second Readout:
January came in slightly ahead on revenue, led by strong UK and US performance
in Personal Care (+$3.1M combined). Gross margin held. The story is OpEx —
MX logistics costs ran $1.4M over budget due to Q4 freight contract renewals
hitting in January. EBITDA is marginally favourable but OpEx is a watch item
for February.
```

Two clicks to export to PDF. She walks into the boardroom.

---

## Use Case 4: Follow-Up Questions in Conversation

**Who:** Albert Lee (controller role)
**When:** After the variance report has been generated

Albert stays in the same Claude conversation. The WorkflowContext is live. He asks:

> *"Which country is driving the COGS variance?"*

The harness calls `rank_variance_drivers` for COGS only and returns:

```
Top COGS Variance Drivers — January 2026

1. MX   −$2.1M  U   Logistics cost increases, freight contract renewal
2. IN   −$1.4M  U   Raw material inflation, palm oil +12% YoY
3. UK   +$0.8M  F   Manufacturing efficiency gains, line 3 upgrade
4. CN   −$0.6M  U   Energy surcharges
   Others          −$0.3M  net U
```

He asks: *"Rewrite the executive summary — the CFO has 20 seconds, not 30."*

The harness calls `executive_summary` with a tighter constraint. Returns a 3-sentence version. The report is not re-generated — only the summary updates.

He asks: *"The India number looks off. Re-run variance for IN only using the file FX rate, not the fallback."*

The harness re-runs `compute_variances` scoped to IN, overriding the FX source. Returns the delta. Logs the override in the Data Quality tab.

---

## Use Case 5: Scheduled Monthly Report

**Who:** Albert Lee's team
**Setup:** One-time configuration

Albert runs the setup wizard once:

```
Where do your files land each month?        → SharePoint: /Finance/Monthly Close/
What day should the report run?             → 3rd business day of the month, 7am
Who gets the report?                        → #finance-leadership (Slack)
Which workflow?                             → variance_analyst
Fallback if files are missing?              → Notify Albert, don't run
```

After that, the `variance-watcher` managed agent runs automatically. On the 3rd business day of every month, before Albert's Monday standup, the validated report is in the Slack channel. Albert reviews exceptions, not the whole report.

He stopped spending 4 hours on the variance report. He now spends 20 minutes reviewing what the harness flagged.

---

## Use Case 6: Budget vs Actual — Month-End Close

**Who:** Finance controller team
**When:** Month-end close, multiple reviewers
**Workflow:** `budget_vs_actual`

The controller uploads the period files. The workflow runs the full variance package plus a close checklist:

```
✓  All 8 country GL files received and validated
✓  Budget extract filtered to correct period
✓  FX rates sourced (4 from file, 3 from fallback — logged)
✓  EBITDA reconciliation passed
✓  Prior period comparatives included
✓  Intercompany eliminations flagged for manual review

Close package ready. 12 items in Data Quality log for review.
```

The close package exports as a single HTML with a Data Quality tab that the controller can reference during the close review meeting. The 12 flagged items are the agenda.

---

## Use Case 7: Analyst Running a Partial Workflow

**Who:** Junior analyst (analyst role)
**Scenario:** Runs validation and variance computation for a first review

The analyst uploads files and runs `compute_variances`. The sequencer allows this — it's within the analyst role.

She tries to call `generate_commentary`. The sequencer checks her role.

```
This step requires controller or above.
Your computed variances have been saved to this session.
Ask your controller to continue from here, or request a role upgrade.
```

She's not blocked from her work. The variances she computed are preserved in the session. Her controller picks up the same run, reviews her numbers, and continues from step 6.

---

## What Success Looks Like for Each Persona

| Persona | Before | After |
|---|---|---|
| FP&A Manager (Albert) | 4 hours building the variance report in Excel, then checking LLM output manually | 30 minutes reviewing what the harness flagged |
| CFO (Sarah) | Waiting for Albert's report, sometimes getting it wrong | Flash report ready before her 8am, 94% confidence score |
| Controller | Manually checking every number before sign-off | Reviewing the gate log — the harness caught the issues |
| Junior Analyst | Afraid to share LLM output because she can't verify it | Confidently shares validated variance computation with her controller |
| Enterprise Procurement | "We can't route financial data through a third-party LLM" | "It runs in our VPC on our Azure OpenAI endpoint" — approved |
