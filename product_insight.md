# Financial Model Evaluation Harness — Product Insight

## The Product Case

The wedge is specific: **finance professionals are already doing agentic LLM work — badly.**

Albert Lee's repo is proof. He got the numbers wrong by $376M, iterated to v4, and buried all domain expertise in ephemeral prompts that nobody else can use or audit. This is happening at every mid-market and enterprise FP&A team right now. The problem isn't awareness of AI — it's the absence of infrastructure that makes AI outputs *trustworthy enough to put in front of a CFO*.

**The product: trust infrastructure for financial LLM outputs.** Not another AI finance tool. The harness.

---

## CLI vs. Adaptive Interface: No Contest

Don't build for CLI as the primary surface.

The CLI-first path optimizes for the wrong persona. The person who would use a CLI in a finance context — a quant analyst, a financial engineer — already has Python and pandas. They don't need the harness; they'll roll their own. The person who *needs* this product is Albert Lee: an FP&A manager who understands finance deeply but doesn't want to debug prompt chains at midnight because EBITDA is off by $107M.

Albert Lee does not live in a terminal. He lives in Excel, PowerPoint, Teams, and a BI dashboard.

**The adaptive interface via MCP is the right architecture.** Here's why:

1. **MCP maps perfectly to the 9-section workflow.** Each section is a natural skill boundary — independently callable, independently evaluatable, composable.
2. **BYOLLM is a forcing function for MCP.** Finance teams have enterprise agreements with OpenAI, Azure OpenAI, or Anthropic. They're not routing data through your cloud. An MCP server inside their environment calling *their* LLM endpoint is the only architecture that clears enterprise procurement.
3. **The interface adapts to where finance users already are.** Claude Desktop today, custom dashboard tomorrow, Excel or Teams the day after. You're not building a destination product — you're building the substrate.
4. **The evaluation harness is the differentiator — and it's invisible in a CLI.** The real value isn't report generation. It's the domain rules: FX fallback logic, GL-to-budget mapping, materiality thresholds, sign conventions, the check that catches a $376M error before the CFO sees it.

---

## Skill Ensemble (mapped to the 9-section workflow)

```
validate_financial_data      → Section 2 (data quality + QC)
compute_variances            → Section 3 (Actual vs Budget, F/U flags)
rank_variance_drivers        → Section 4 (top 5 by materiality)
generate_ebitda_bridge       → Section 5 (waterfall chart)
traffic_light_dashboard      → Section 6 (country × P&L heatmap)
generate_commentary          → Section 7 (What/So What/Now What)
executive_summary            → Section 8 (board-ready paragraph)
export_report                → Section 9 (self-contained HTML)
```

---

## Full Architecture

```
Finance User (Excel / Teams / Claude Desktop / Custom UI)
        │
        ▼
   MCP Server  ←── BYOLLM endpoint (their OpenAI / Anthropic / Azure)
        │
   Skill Ensemble
   ├── validate_financial_data
   ├── compute_variances
   ├── rank_variance_drivers
   ├── generate_ebitda_bridge
   ├── traffic_light_dashboard
   ├── generate_commentary
   ├── executive_summary
   └── export_report
        │
   Domain Rules Engine  ←── THE MOAT
   ├── GAAP P&L line mapping
   ├── FX fallback rates + monthly average logic
   ├── Materiality thresholds (what's a real variance vs noise)
   ├── GL allocation rules
   ├── Sign conventions (F/U flags)
   └── Narrative quality rubric (What/So What/Now What)
        │
   Evaluation Harness
   ├── Numerical accuracy checks
   ├── Completeness checks
   ├── Cross-validation (Sales + COGS + OpEx → EBITDA reconciliation)
   └── Output scoring → confidence signal to user
```

---

## Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│              LLM Abstraction Layer                   │
│         (BYOLLM: OpenAI / Claude / Gemini / OSS)     │
├─────────────────────────────────────────────────────┤
│              Prompt Registry                         │
│    Versioned, parameterized templates                │
│    (the 5 prompts here, made reusable)               │
├─────────────────────────────────────────────────────┤
│           Execution Orchestrator                     │
│    Multi-step pipeline (9 sections)                  │
│    Retry logic, correction loops                     │
├─────────────────────────────────────────────────────┤
│         Domain Evaluation Suite                      │
│  ┌─────────────────┬────────────────────────────┐   │
│  │ Numerical checks│ Narrative quality checks   │   │
│  │ • Math accuracy │ • What/So What/Now What    │   │
│  │ • FX conversion │ • Materiality language     │   │
│  │ • Sign / F/U   │ • Board-ready tone         │   │
│  │ • Completeness  │ • Section coverage         │   │
│  └─────────────────┴────────────────────────────┘   │
├─────────────────────────────────────────────────────┤
│            Financial Data Normalizer                 │
│   GL / Budget / COGS / FX / OpEx / Sales → schema   │
├─────────────────────────────────────────────────────┤
│             Finance Domain Rules                     │
│   GAAP mapping, materiality, FX fallbacks,           │
│   allocation logic, P&L line conventions             │
└─────────────────────────────────────────────────────┘
```

---

## The Moat

The domain rules layer is where defensibility lives. General eval frameworks (Braintrust, LangSmith, Promptfoo) are LLM-agnostic but finance-ignorant. Bloomberg GPT is finance-aware but closed. Nobody has built the open, composable substrate that encodes *why* a variance analysis is correct — not just whether it ran.

The prompt chain in this repo is the training data for that evaluator. Prompt 3 (the $376M correction) is a gold-standard eval case: the failure mode is known, the correct output is known, the rule that catches it can be codified. The "v4" iteration loop is exactly what a harness should eliminate.

The longer a finance team uses the harness, the more their domain-specific rules get encoded. That's the retention loop.

---

## Bottom Line

Finance professionals will never adopt a tool that requires them to become engineers. But they will adopt a skill that shows up inside the tool they're already in, gives them a trustworthy answer in 30 seconds, and tells them *why* the number is right.

That's an MCP ensemble, not a CLI.
