# byollm-finance-harness — Canonical Repo Structure

Architecture decisions locked:
- Workflow-centric surface, not infrastructure-centric
- Single-tenant, user-scoped within org deployment
- WorkflowContext assembled once, immutable in flight
- Trust is a gate (blocks emit), not a flag (warns)
- FSM as the step sequencer — invisible to the user

Folder names map directly to the agreed architecture layers.

---

```
byollm-finance-harness/
│
├── .github/
│   └── workflows/
│       ├── ci.yml                      ← lint, type-check, unit tests
│       └── eval.yml                    ← full eval harness on push to main
│
├── .claude-plugin                      ← MCP marketplace manifest
├── .mcp.json                           ← MCP server registration
├── CLAUDE.md                           ← Finance profile every skill reads from
├── QUICKSTART.md
├── README.md
├── pyproject.toml                      ← uv-managed, single source of deps
├── uv.lock
├── Dockerfile
│
│
├── workflows/                          ── SURFACE LAYER ──────────────────────
│   │                                   User-facing. Each workflow owns its
│   │                                   step sequence and role requirements.
│   │
│   ├── variance_analyst/               ← Full 9-step monthly variance report
│   │   ├── definition.py               ← Step sequence + role requirements
│   │   └── config.yaml                 ← Workflow-specific overrides
│   │
│   ├── flash_report/                   ← CFO scorecard + summary only
│   │   ├── definition.py
│   │   └── config.yaml
│   │
│   ├── budget_vs_actual/               ← Month-end close package
│   │   ├── definition.py
│   │   └── config.yaml
│   │
│   └── fx_exposure/                    ← Currency risk snapshot
│       ├── definition.py
│       └── config.yaml
│
│
├── engine/                             ── WORKFLOW ENGINE ─────────────────────
│   │                                   FSM sequencer, context, registry, roles.
│   │                                   Invisible to users. Serves the workflows.
│   │
│   ├── context.py                      ← WorkflowContext (immutable dataclass)
│   │                                     assembled once, passed through all steps
│   ├── sequencer.py                    ← FSM step sequencer
│   │                                     validates step allowed for user role
│   │                                     retries on transient failure
│   │                                     blocks if trust gate fails
│   ├── registry.py                     ← Skill registry — maps step names to skills
│   └── roles.py                        ← User role definitions + capability scoping
│                                         analyst / controller / cfo
│
│
├── skills/                             ── SKILL PRIMITIVES ────────────────────
│   │                                   Nine callable MCP skills.
│   │                                   Each is independently testable.
│   │
│   ├── validate_financial_data.py      ← Section 2
│   ├── compute_variances.py            ← Section 3
│   ├── rank_variance_drivers.py        ← Section 4
│   ├── generate_ebitda_bridge.py       ← Section 5
│   ├── traffic_light_dashboard.py      ← Section 6
│   ├── generate_commentary.py          ← Section 7
│   ├── executive_summary.py            ← Section 8
│   └── export_report.py               ← Section 9
│
│
├── trust/                              ── TRUST LAYER ─────────────────────────
│   │                                   The moat. Rules, validators, output gate.
│   │                                   Output does not emit until this passes.
│   │
│   ├── rules/                          ← Domain rules engine (Pydantic v2)
│   │   ├── fx.py                       ← Fallback rates, monthly avg logic
│   │   ├── materiality.py              ← Thresholds by P&L line
│   │   ├── gl_mapping.py               ← GL accounts → Budget P&L lines
│   │   └── sign.py                     ← F/U conventions, Actual-Budget sign
│   │
│   ├── validators/                     ← Numerical correctness checks
│   │   ├── reconciliation.py           ← Revenue − COGS − OpEx = EBITDA
│   │   ├── completeness.py             ← All required sections present
│   │   ├── fx_coverage.py              ← Every currency has a rate
│   │   └── variance.py                 ← Math, sign, materiality flags
│   │
│   └── gate.py                         ← Output gate
│                                         RunResult → scored → emit or hold
│                                         blocks progression, not just warns
│
│
├── model/                              ── MODEL LAYER ─────────────────────────
│   │                                   Thin. BYOLLM via LiteLLM.
│   │                                   Provider is config, not code.
│   │
│   ├── client.py                       ← LiteLLM unified client
│   ├── config.py                       ← Azure / Anthropic / Bedrock / Ollama
│   └── structured.py                   ← Structured output enforcement
│                                         model must return valid JSON
│                                         before result re-enters sequencer
│
│
├── data/                               ── DATA LAYER ──────────────────────────
│   │                                   Normalizer + connectors.
│   │                                   Raw files → canonical WorkflowContext.
│   │
│   ├── normalizer/
│   │   ├── gl.py
│   │   ├── budget.py
│   │   ├── cogs.py
│   │   ├── fx.py
│   │   ├── opex.py
│   │   └── sales.py
│   │
│   └── connectors/
│       ├── excel.py                    ← openpyxl — primary connector today
│       ├── csv.py
│       └── erp/                        ← ERP stubs (SAP, Oracle, NetSuite)
│           ├── sap.py
│           ├── oracle.py
│           └── netsuite.py
│
│
├── server/                             ── MCP SERVER ──────────────────────────
│   └── main.py                         ← Entrypoint. Registers all 9 skills.
│                                         One uvx command to install.
│
│
├── templates/                          ← Jinja2 report templates
│   ├── report.html.j2                  ← 5-tab HTML (Glance/Exec/Deep/Commentary/DQ)
│   ├── commentary.md.j2                ← What / So What / Now What structure
│   └── executive_summary.md.j2         ← 30-second readout format
│
│
├── config/                             ← Rule definitions — editable YAML
│   │                                   Finance teams edit these, not the code.
│   │
│   ├── fx_rules.yaml                   ← Fallback rates + source priority
│   ├── materiality.yaml                ← Thresholds by P&L line + severity
│   ├── gl_mapping.yaml                 ← GL account → P&L line mapping
│   ├── roles.yaml                      ← analyst / controller / cfo capabilities
│   └── llm_providers.yaml              ← Active provider + endpoint config
│
│
├── eval/                               ── EVALUATION HARNESS ──────────────────
│   │                                   pytest suite. This is what you show
│   │                                   enterprise buyers. Not the report —
│   │                                   the test suite.
│   │
│   ├── conftest.py
│   ├── fixtures/
│   │   ├── fmcg_jan2026/               ← The 6 Excel files from source repo
│   │   │   ├── FMCG_GL_Actuals_Jan2026.xlsx
│   │   │   ├── FMCG_COGS_Manufacturing_Jan2026.xlsx
│   │   │   ├── FMCG_FX_Rates_Jan2026.xlsx
│   │   │   ├── FMCG_FY2026_Budget_USD.xlsx
│   │   │   ├── FMCG_Operating_Expenses_Jan2026.xlsx
│   │   │   └── FMCG_Sales_by_SKU_Jan2026.xlsx
│   │   └── synthetic/                  ← Generated edge-case datasets
│   │
│   ├── cases/
│   │   ├── test_001_376m_error.py      ← The known failure. Regression test #1.
│   │   ├── test_fx_fallback.py         ← BRL/INR/MXN hardcoded enforcement
│   │   ├── test_reconciliation.py      ← EBITDA cross-validation
│   │   ├── test_output_gate.py         ← Gate blocks bad output, not just flags
│   │   ├── test_materiality.py         ← Threshold flagging by P&L line
│   │   ├── test_role_scoping.py        ← Analyst cannot run CFO-gated steps
│   │   └── test_narrative_quality.py   ← What/So What/Now What coverage
│   │
│   └── benchmarks/                     ← Score LLM providers on same cases
│       ├── gpt4o.py
│       ├── claude_sonnet.py
│       └── llama3.py
│
│
└── scripts/
    ├── install.sh
    ├── run_eval.sh
    └── seed_fixtures.py                ← Copies Jan2026 files into eval/fixtures/
```
