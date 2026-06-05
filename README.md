# byollm-finance-harness

**Trust infrastructure for financial LLM outputs.**

A workflow-centric harness for FP&A variance analysis. Bring your own LLM. Your data stays in your environment. Every number is validated against domain rules before it reaches the CFO.

---

## The Problem

Finance teams are already using LLMs to run variance analysis, build EBITDA bridges, and write management commentary. The output looks right. The formatting is clean. And occasionally the revenue is off by $376 million.

Nobody catches it until a human checks. The human has to check everything, every time, because there is no systematic layer between the LLM and the report.

This harness is that layer.

It encodes the financial logic your LLM doesn't know — GL-to-budget mapping, materiality thresholds, FX fallback conventions, sign rules, EBITDA reconciliation — and enforces them as gates, not warnings. Output that doesn't pass doesn't emit. The report doesn't render until the numbers are correct.

---

## Architecture

```
Finance User (Claude Desktop / custom UI / scheduled agent)
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│                    WORKFLOW SURFACE                        │
│  variance-analyst · flash-report · budget-vs-actual ·     │
│  fx-exposure · [bring your own workflow]                   │
└────────────────────────┬──────────────────────────────────┘
                         │
┌────────────────────────▼──────────────────────────────────┐
│                   WORKFLOW ENGINE                          │
│                                                           │
│  WorkflowContext ── assembled once, immutable in flight   │
│                                                           │
│  FSM Step Sequencer                                       │
│  ├── validates step is allowed for user role              │
│  ├── passes WorkflowContext to each skill                 │
│  ├── retries on transient failure                         │
│  └── blocks progression if trust gate fails               │
│                                                           │
│  Skill Registry ── 9 MCP skills, independently callable  │
└────────┬──────────────────────────────┬───────────────────┘
         │                              │
┌────────▼──────────┐       ┌───────────▼──────────────────┐
│   TRUST LAYER     │       │        MODEL LAYER            │
│                   │       │                               │
│  Domain Rules     │       │  LiteLLM (BYOLLM)             │
│  ├── fx_rules     │       │  ├── Azure OpenAI             │
│  ├── materiality  │       │  ├── Anthropic                │
│  ├── gl_mapping   │       │  ├── AWS Bedrock              │
│  └── sign_convs   │       │  ├── Google Vertex            │
│                   │       │  └── Ollama (local)           │
│  Validators       │       │                               │
│  ├── reconcile    │       │  Structured output enforced   │
│  ├── completeness │       │  before re-entering sequencer │
│  └── fx_coverage  │       └───────────────────────────────┘
│                   │
│  Output Gate      │  ← blocks emit, does not warn
└────────┬──────────┘
         │
┌────────▼──────────────────────────────────────────────────┐
│                     DATA LAYER                            │
│  Normalizer: GL / Budget / COGS / FX / OpEx / Sales      │
│  Connectors: Excel · CSV · ERP stubs · live FX feeds     │
│  Session state: in-memory, scoped to one workflow run     │
└───────────────────────────────────────────────────────────┘
```

---

## Design Principles

**1. Workflow-first.** The 9 skills are the product surface. The engine, trust layer, and model layer serve the workflows — not the other way around.

**2. Single-tenant, user-scoped.** One deployment per organisation. Runs inside your VPC on your LLM endpoint. User roles (`analyst` / `controller` / `cfo`) scope which workflows and steps each user can execute.

**3. WorkflowContext is immutable in flight.** Assembled once at the start of a run. Passed unchanged through every step. No step mutates what another step sees.

**4. Trust is a gate, not a flag.** A reconciliation failure stops the workflow. The report does not render with a warning attached — it does not render at all until the numbers close.

**5. The FSM is invisible.** The step sequencer is deterministic and auditable internally. The finance user sees a workflow.

---

## The Nine Skills

Each skill is an independently callable MCP tool. They can be called individually or chained by a workflow.

| Step | Skill | What it does |
|------|-------|-------------|
| 1 | `validate_financial_data` | Data quality checks, FX fallback application, dimension reconciliation |
| 2 | `compute_variances` | Actual vs Budget in USD, F/U flags, period filtering |
| 3 | `rank_variance_drivers` | Top drivers by materiality across Revenue, COGS, OpEx |
| 4 | `generate_ebitda_bridge` | Waterfall chart: Budget EBITDA → variances → Actual EBITDA |
| 5 | `traffic_light_dashboard` | Country × P&L line heatmap, green/amber/red by threshold |
| 6 | `generate_commentary` | CFO-facing narrative, What / So What / Now What per P&L line |
| 7 | `executive_summary` | Board-ready paragraph, 30-second readout format |
| 8 | `export_report` | Self-contained HTML, 5-tab layout, base64-embedded charts |

All skills receive a `WorkflowContext`. All skills return a `StepResult`. Skills do not call each other.

---

## Quick Start

### Prerequisites

- Python 3.12+
- [`uv`](https://docs.astral.sh/uv/) for package management
- A GitHub account and `gh` CLI (for repo operations)
- An LLM provider endpoint (Azure OpenAI, Anthropic, Bedrock, Vertex, or local Ollama)

### Install

```bash
git clone https://github.com/TaiPanda96/byollm-finance-harness
cd byollm-finance-harness
uv sync
```

### Configure your LLM provider

Edit `config/llm_providers.yaml`:

```yaml
# Azure OpenAI
provider: azure
model: azure/gpt-4o
api_base: https://your-org.openai.azure.com/
api_key: ${AZURE_OPENAI_API_KEY}
api_version: "2024-08-01-preview"

# --- or Anthropic ---
# provider: anthropic
# model: claude-sonnet-4-5
# api_key: ${ANTHROPIC_API_KEY}

# --- or local Ollama ---
# provider: ollama
# model: ollama/llama3
# api_base: http://localhost:11434
```

One config line to switch providers. No code changes.

### Configure user roles

Edit `config/roles.yaml`:

```yaml
roles:
  analyst:
    allowed_skills:
      - validate_financial_data
      - compute_variances
      - rank_variance_drivers

  controller:
    allowed_skills: all

  cfo:
    allowed_workflows:
      - flash_report
```

### Start the MCP server

```bash
uv run python server/main.py
```

The server registers all 9 skills as MCP tools. Add it to your Claude Desktop config:

```json
{
  "mcpServers": {
    "finance-harness": {
      "command": "uv",
      "args": ["run", "python", "/path/to/byollm-finance-harness/server/main.py"]
    }
  }
}
```

---

## Running a Workflow

### From Claude Desktop

Drop your 6 financial files into the conversation and type:

```
Run the variance analyst for January.
```

The harness works through 9 steps, surfaces data quality notes, validates reconciliation, and returns a downloadable HTML report with a confidence score.

### Programmatically

```python
from engine.context import WorkflowContext, UserRole
from engine.sequencer import run_workflow
from data.connectors.excel import load_financial_files

# Load and normalise raw Excel files
ctx = load_financial_files(
    gl="data/FMCG_GL_Actuals_Jan2026.xlsx",
    budget="data/FMCG_FY2026_Budget_USD.xlsx",
    cogs="data/FMCG_COGS_Manufacturing_Jan2026.xlsx",
    fx="data/FMCG_FX_Rates_Jan2026.xlsx",
    opex="data/FMCG_Operating_Expenses_Jan2026.xlsx",
    sales="data/FMCG_Sales_by_SKU_Jan2026.xlsx",
    period="2026-01",
    user_role=UserRole.CONTROLLER,
)

# Run the workflow — FSM sequences the 9 skills, gate validates output
result = await run_workflow("variance_analyst", ctx)

print(result.confidence_score)   # e.g. 0.94
print(result.gate_log)           # what the validators checked
print(result.report_path)        # path to the rendered HTML
```

### Calling a single skill

```python
from skills.compute_variances import compute_variances

step_result = await compute_variances(ctx)
print(step_result.data)          # variance DataFrame
print(step_result.warnings)      # non-blocking notes
```

---

## The Trust Gate

The output gate in `trust/gate.py` enforces five checks on every run. If any check fails, the workflow stops and the reason is surfaced. The report does not render.

| Check | Rule |
|-------|------|
| EBITDA reconciliation | `Revenue − COGS − OpEx = EBITDA` within rounding tolerance |
| FX coverage | Every currency in the dataset has a rate (file or fallback) |
| Completeness | All required workflow sections are present |
| Sign convention | F/U flags are consistent with P&L line direction |
| Materiality | Only material variances are surfaced in driver tables |

```
✗  Step 5 of 9 — Reconciliation failed. Report blocked.

   Revenue ($142.3M) − COGS ($89.1M) − OpEx ($31.4M) = $21.8M
   Reported EBITDA: $18.2M
   Gap: $3.6M — exceeds materiality threshold ($0.5M)

   Likely cause: GL file missing logistics entries for MX and IN.
   Suggested fix: Re-export GL actuals including cost centre 7xxx entries.
```

---

## Domain Rules

All financial rules live in `config/` as YAML. Finance teams edit these directly — no Python required.

```yaml
# config/materiality.yaml
thresholds:
  revenue:
    absolute_usd: 500_000
    percentage: 0.5
    severity: block
  cogs:
    absolute_usd: 250_000
    percentage: 1.0
    severity: block
  opex:
    absolute_usd: 100_000
    percentage: 2.0
    severity: warn
```

```yaml
# config/fx_rules.yaml
fallback_rates:
  BRL: 0.17
  INR: 0.0118
  MXN: 0.049
source_priority:
  - daily_file
  - monthly_average
  - hardcoded_fallback
```

```yaml
# config/gl_mapping.yaml
mappings:
  - gl_pattern: "Revenue*"
    pl_line: revenue
  - gl_pattern: "COGS*"
    pl_line: cogs
  - gl_pattern: ["Logistics*", "Marketing*", "SGA*"]
    pl_line: opex
```

---

## Evaluation Harness

The `eval/` directory contains the pytest suite. The 6 Excel files from the source dataset are live fixtures. Every validator has a corresponding test.

```bash
# Run the full eval suite
uv run pytest eval/

# Run a specific case
uv run pytest eval/cases/test_001_376m_error.py -v

# Run provider benchmarks
uv run pytest eval/benchmarks/ --provider=gpt4o
```

`test_001_376m_error.py` is the regression test for the known failure mode — a real $376M revenue discrepancy caused by incomplete GL file coverage. It runs on every push to `main`.

```python
# eval/cases/test_001_376m_error.py
def test_incomplete_gl_triggers_gate(fmcg_jan2026_incomplete_gl):
    """
    GL file missing logistics entries for MX and IN.
    Harness must detect the EBITDA gap and block the report.
    Revenue discrepancy in the original dataset: $376.7M.
    """
    result = run_workflow_sync("variance_analyst", fmcg_jan2026_incomplete_gl)

    assert result.status == WorkflowStatus.BLOCKED
    assert result.gate_log.reconciliation_failed is True
    assert result.report_path is None
```

---

## Repo Structure

```
byollm-finance-harness/
├── workflows/        User-facing workflow definitions
├── engine/           FSM sequencer, WorkflowContext, skill registry, roles
├── skills/           Nine callable MCP skills
├── trust/            Domain rules, validators, output gate
├── model/            LiteLLM BYOLLM client (thin)
├── data/             Financial data normaliser and connectors
├── server/           MCP server entrypoint
├── templates/        Jinja2 report templates
├── config/           YAML rule definitions (editable by finance teams)
└── eval/             pytest evaluation harness with real Excel fixtures
```

Full structure with annotations: [`repo_structure.md`](repo_structure.md)

---

## Supported Workflows

| Workflow | Description | Skills used |
|----------|-------------|-------------|
| `variance_analyst` | Full 9-step monthly variance report | All 9 |
| `flash_report` | CFO scorecard + 30-second readout | 1, 2, 7, 8 |
| `budget_vs_actual` | Month-end close package with close checklist | All 9 + close checks |
| `fx_exposure` | Currency risk snapshot across all active currencies | 1, 2, 3 |

Custom workflows are defined in `workflows/` by specifying a step sequence and role requirements.

---

## Scheduled Agents

The `managed-agent-cookbooks/` directory contains configured agents for automated runs.

```yaml
# managed-agent-cookbooks/monthly-close/config.yaml
workflow: variance_analyst
schedule: "0 7 * * 1-5"          # 7am weekdays
trigger: 3rd-business-day-of-month
source:
  type: sharepoint
  path: /Finance/Monthly Close/
notify:
  slack: "#finance-leadership"
  on_gate_failure: alert @albert.lee
```

---

## Stack

| Concern | Choice |
|---------|--------|
| Language | Python 3.12+ |
| Package management | `uv` |
| LLM abstraction | LiteLLM |
| Schema validation | Pydantic v2 |
| Report templating | Jinja2 |
| MCP interface | Anthropic MCP Python SDK |
| Linting / formatting | `ruff` |
| Test harness | pytest |
| Observability | Logfire |

---

## Further Reading

- [`CLAUDE.md`](CLAUDE.md) — architecture principles, layer ownership, finance domain vocabulary, coding conventions, and anti-patterns. Read this before contributing.
- [`USE_CASES.md`](USE_CASES.md) — seven end-user scenarios across all four personas: analyst, controller, CFO, and enterprise procurement.
- [`product_insight.md`](product_insight.md) — product positioning, the trust problem, and why this is an MCP ensemble rather than a CLI tool.

---

## Status

Early-stage. Architecture is locked. Scaffolding in progress.

The evaluation harness and domain rules engine are the first build targets — intentionally before LLM integration. If you cannot validate a correct answer without an LLM, you cannot evaluate an LLM's answer with one.

Contributions welcome. Read [`CLAUDE.md`](CLAUDE.md) first.
