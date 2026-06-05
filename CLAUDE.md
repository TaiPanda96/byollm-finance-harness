# CLAUDE.md — byollm-finance-harness

This file primes every coding agent working on this repo. Read it before writing a single line.

---

## What This Is

A **workflow-centric harness** for financial LLM outputs. Not an AI finance tool. Not a report generator. Trust infrastructure — the layer that sits between an LLM and a CFO, enforcing domain rules, validating numerical correctness, and blocking output that hasn't been verified.

The product promise to the finance professional: *"You get a result you can put your name on."*

The product promise to the enterprise: *"Runs in your VPC. Your LLM contract. Your data never leaves your environment."*

---

## Five Design Principles. Non-Negotiable.

**1. Workflow-first.**
The 9 skills are the product surface. The engine, trust layer, and model layer exist to serve the workflows — not the other way around. If you're building something that isn't directly serving a workflow, ask why.

**2. Single-tenant, user-scoped.**
One deployment per organization. User roles (analyst / controller / cfo) control which workflows and steps a user can execute. There is no multi-tenant routing. There is no shared infrastructure between organizations. BYOLLM means their endpoint, their key, their VPC.

**3. WorkflowContext is immutable in flight.**
Assembled once at the start of a run. Passed unchanged through every step. No step mutates what another step sees. If you find yourself modifying context mid-workflow, you're doing it wrong — create a new derived context with explicit lineage.

**4. Trust is a gate, not a flag.**
The output does not emit until validators clear it. A reconciliation failure at step 3 blocks steps 4–9 from running. Do not add warning fields to outputs. Do not let bad numbers reach the report layer. Block. Surface the reason. Stop.

**5. The FSM is invisible.**
The step sequencer is an FSM internally — deterministic, auditable, recoverable. The finance user never sees it. They see a workflow. Never expose FSM state names, transition logic, or retry counts in user-facing output.

---

## Architecture Layers

Each folder in this repo owns exactly one layer. Do not mix concerns across layers.

```
workflows/    SURFACE     User-facing workflow definitions. Step sequences.
                          Role requirements. Workflow-specific config overrides.

engine/       ENGINE      FSM step sequencer. WorkflowContext. Skill registry.
                          User role capability scoping. Retry logic.
                          The sequencer validates each step is allowed for the
                          current user role before calling the skill.

skills/       PRIMITIVES  Nine callable MCP skills. Each independently testable.
                          Skills receive a WorkflowContext. They do not call
                          each other. They do not know about the sequencer.

trust/        TRUST       Domain rules engine (Pydantic v2). Validators.
                          Output gate. This is the moat. Rules live in
                          config/YAML — not hardcoded in Python.

model/        MODEL       LiteLLM client. Provider config. Structured output
                          enforcement. The model must return valid structured
                          JSON before the result re-enters the sequencer.
                          This layer is intentionally thin.

data/         DATA        Financial data normalizer. Raw files → canonical
                          WorkflowContext fields. Connectors for Excel, CSV,
                          ERP stubs. Session state is in-memory, scoped to
                          one workflow run.

server/       INTERFACE   MCP server. Registers all 9 skills. One file.
                          One entrypoint. No business logic here.
```

---

## The Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Python 3.12+ | Finance data is pandas-native. Don't fight it. |
| Package management | uv | Fast, deterministic, lockfile-first. |
| LLM abstraction | LiteLLM | BYOLLM — one line to switch providers. Don't build your own. |
| Schema validation | Pydantic v2 | Rules as typed models. Enforced at runtime, readable by controllers. |
| Report templating | Jinja2 | Replaces hardcoded HTML string generation. Testable, versionable. |
| MCP interface | Anthropic MCP Python SDK | Skills are MCP tools. Distribution via `uvx`. |
| Test harness | pytest | Eval cases are pytest tests. Real Excel fixtures in `eval/fixtures/`. |
| Observability | Logfire | Full trace per run. LLM calls, rule evaluations, gate decisions. |

**Do not introduce:**
- LangChain — too many abstraction layers, breaks unpredictably
- Vector databases — this is not a RAG problem
- A frontend framework — MCP is the interface for now
- Temporal — FSM-lite handles retry and recovery for v1

---

## Finance Domain Vocabulary

Every agent working on this repo must understand these terms precisely.

| Term | Definition |
|---|---|
| Variance | Actual minus Budget. Positive = Favourable (F). Negative = Unfavourable (U). |
| Materiality | A variance is material if it exceeds the threshold defined in `config/materiality.yaml` for that P&L line. Immaterial variances are not surfaced. |
| EBITDA bridge | Waterfall chart showing Budget EBITDA → revenue variance → COGS variance → OpEx variance → Actual EBITDA. Every bar must sum correctly. |
| GL | General Ledger. Raw journal entries in local currency. Source of actuals. |
| FX fallback | When a currency rate is missing from the FX file, use the hardcoded rate in `config/fx_rules.yaml`. BRL, INR, MXN are known missing currencies. |
| F/U flag | Favourable/Unfavourable. Applied to every variance line. Revenue: positive variance = F. COGS: negative variance = F (cost saving). |
| Traffic light | Country × P&L line heatmap. Green = within threshold. Amber = approaching threshold. Red = exceeds threshold. |
| What/So What/Now What | The narrative structure for every management commentary section. What happened → Why it matters → What we do next. |
| Month-end close | The process of finalizing financial results for a month. Budget vs Actual workflow is the primary harness for this. |

---

## The Trust Contract

The output gate in `trust/gate.py` enforces this contract on every run:

```
EBITDA reconciliation    Revenue − COGS − OpEx = EBITDA within rounding tolerance
FX coverage              Every currency in the dataset has a rate (file or fallback)
Completeness             All required sections for the workflow are present
Sign convention          F/U flags are consistent with P&L line direction
Materiality              Only material variances are surfaced in driver tables
```

If any check fails: the workflow stops, the reason is surfaced to the user, the report does not render. No partial reports. No reports with warnings attached.

---

## WorkflowContext Shape

Every skill receives a `WorkflowContext`. Understand this object.

```python
@dataclass(frozen=True)
class WorkflowContext:
    run_id: str                      # UUID for this workflow run
    workflow: str                    # e.g. "variance_analyst"
    user_role: UserRole              # analyst | controller | cfo
    period: str                      # e.g. "2026-01"
    tenant_id: str                   # org identifier

    # Data — populated by the normalizer before sequencer starts
    gl_actuals: pd.DataFrame
    budget: pd.DataFrame
    cogs: pd.DataFrame
    fx_rates: pd.DataFrame
    opex: pd.DataFrame
    sales: pd.DataFrame

    # Accumulates as steps complete — each step adds its result
    results: dict[str, StepResult]
```

`frozen=True` is not optional. If a skill needs to derive something from context, it returns a `StepResult`. It does not modify context.

---

## Coding Conventions

- **Follow latest Python idioms.** Use `match` for pattern matching, `X | Y` union syntax over `Union[X, Y]`, `pathlib.Path` over `os.path`, `ruff` for linting and formatting, and `@dataclass(frozen=True)` for immutable data. Prefer `pyproject.toml` for all project configuration. Target Python 3.12+ features — do not write code that is compatible with older versions.
- **Type everything.** No `Any`. No untyped dicts passed between layers.
- **Rules in YAML, not code.** Materiality thresholds, FX fallbacks, GL mappings — all in `config/`. If a finance team needs to change a threshold, they edit YAML, not Python.
- **Every skill is independently testable.** Skills receive a `WorkflowContext` and return a `StepResult`. No side effects. No external calls except through `model/client.py`.
- **Eval fixtures are real data.** The 6 Excel files in `eval/fixtures/fmcg_jan2026/` are the ground truth. Every new validator must have a passing test against these files.
- **Name test cases by what they catch.** `test_001_376m_error.py` not `test_variance.py`. The name should tell a reviewer exactly what failure mode is being prevented.

---

## Anti-Patterns

```
✗  Modifying WorkflowContext inside a skill
✗  A skill calling another skill
✗  Emitting a report before the gate runs
✗  Hardcoding FX rates or thresholds in Python (use config/YAML)
✗  Returning warnings in output instead of blocking
✗  Exposing FSM state to the user
✗  Adding a new dependency without updating pyproject.toml and documenting why
✗  Writing a validator without a corresponding pytest case
```
