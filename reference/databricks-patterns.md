# Reusable Databricks Patterns

The building blocks every blueprint composes. Blueprints reference these by name (e.g., "Pattern B:
Parse → Chunk → Index → RAG") instead of re-explaining them. Think of this as the shared vocabulary.

---

## Pattern A - Medallion ingest (bronze → silver → gold)

**When:** any use case that starts from source-system extracts.
**How:** Auto Loader or batch read → bronze (raw, append) → silver (typed, deduped, conformed,
joined) → gold (business-ready aggregates / feature tables). Built with **Lakeflow Declarative
Pipelines** (streaming tables + materialized views) on serverless.
**Why it matters at SLHS:** this is the exact pattern that replaces the Health Catalyst pipelines
and the manual Power BI/Excel reconciliation. Show it once, reuse everywhere.

---

## Pattern B - Parse → Chunk → Index → RAG (unstructured knowledge)

**When:** answering questions over documents/policies/transcripts (use cases 2, 4, 9, 11; CKD notes).
**How:** `ai_parse_document` (PDF/Word/images → structured text) → chunk → embed into **Vector
Search** index → retrieve top-k → ground an LLM answer with citations. Package as an **Agent Bricks
Knowledge Assistant** for the fast path, or a custom RAG agent when you need custom retrieval logic.
**Governance:** answers cite source docs; UC controls which repositories a user can retrieve from.

---

## Pattern C - SQL-native AI enrichment (`ai_query` / `ai_extract` / `ai_classify`)

**When:** you need LLM logic *inside* a pipeline over structured/semi-structured rows at scale
(extract CKD stage from a note, classify an email theme, summarize an investigation).
**How:** call the AI function directly in a Lakeflow pipeline or SQL job. No endpoint to manage;
runs where the data lives; results land in a Delta column. Deterministic parts stay as SQL rules;
only the fuzzy step calls the model.
**Why:** cheapest, most governable way to add AI to a batch pipeline. Great "aha" for a Fabric shop.

---

## Pattern D - Genie Space (governed natural-language querying)

**When:** business users ask NL questions over trusted tables (HTM "what anesthesia machines need
replacement in 2026?", diversion peer benchmarks, nursing staffing).
**How:** curate a gold-layer schema + **Metric Views** for governed KPIs → build a **Genie Space**
with instructions, sample SQL, and certified metrics → embed via Genie Code / Databricks App.
**Why vs. Copilot:** Genie answers over *governed* UC tables with lineage and row/column security,
not a black box. This is the headline differentiator for #14 (the Day-1 demo anchor).

---

## Pattern E - Forecasting (`ai_forecast` / MLflow models)

**When:** projecting future state from time series (nursing staffing shortages/surpluses by pay
period, HTM replacement volume by year).
**How:** gold feature table → `ai_forecast` for the quick path (SQL, no model management), or a
trained model logged to **MLflow** + batch/serving for custom horizons and covariates.
**Output:** forecast tables feeding a dashboard/app + scenario knobs.

---

## Pattern F - Constrained optimization + human-in-loop

**When:** generating a schedule/plan under hard + soft constraints (hospitalist scheduling).
**How:** encode rules (coverage, PTO, pay compliance, preferences) → solver (OR-Tools / PuLP in a
notebook or job) produces a *draft* → LLM agent explains changes in plain language and answers
"why" → **human reviews and approves** in a Databricks App. The optimizer does the math; the agent
does the communication; the scheduler stays the decision-maker.
**Why:** this is where Copilot-in-Excel failed - it can't hold the constraints. Databricks can.

---

## Pattern G - Anomaly detection + narrative report generation

**When:** flag suspicious patterns and explain them (diversion support).
**How:** feature engineering (peer cohorts, z-scores, timing/dosage rules) → statistical/ML anomaly
flags (deterministic rules first, then ML) → `ai_query` generates a **plain-language risk
narrative** → assemble a structured, human-readable investigation report (replaces the manual Excel
pivot workflow). MLflow logs every generation for audit.
**Governance:** de-identified/synthetic data, RBAC, full audit trail, human investigator owns the
conclusion.

---

## Pattern H - Multi-Agent Supervisor (MAS) orchestration

**When:** a use case needs several specialized skills (retrieve policy + query structured data +
draft a message). E.g., a full "agent" that both looks up a benefit policy (RAG) and checks an
employee's leave balance (SQL).
**How:** **Agent Bricks Multi-Agent Supervisor** routes across a Knowledge Assistant + a Genie Space
+ custom tools. Each sub-agent is independently governed and evaluable.

---

## Pattern I - Serve it: Databricks App + Model Serving

**When:** any solution that needs a UI beyond a notebook (all five hackathon solutions end here).
**How:** deploy the agent/pipeline behind **Model Serving**; build a **Databricks App**
(React/FastAPI) as the front end; use **OBO (on-behalf-of) auth** so the app respects each user's UC
permissions (never make the app SP an admin). Lakebase for any app-side transactional state.
**Why:** turns a POC into something a nurse leader or investigator actually uses - the POC→prod
step Molly's OKR wants.

---

## Pattern J - Evaluate & monitor (MLflow GenAI eval)

**When:** every agent/LLM solution, before anyone trusts it.
**How:** build an eval dataset (synthetic Q&A + expected behavior) → **MLflow `genai.evaluate()`**
with built-in + custom scorers (correctness, groundedness, safety, guideline adherence) → trace in
prod → align LLM-judges to SME feedback. This is the credibility layer for clinical use cases and
the honest answer to "how do you know it's right?"

---

## How blueprints compose these

A typical full solution = **A** (ingest) → **C or B** (AI enrichment / retrieval) → **E/F/G**
(the core analytical engine) → **D** (Genie for exploration) → **I** (serve) → **J** (evaluate).
Not every use case uses every pattern; each blueprint names the exact chain.
