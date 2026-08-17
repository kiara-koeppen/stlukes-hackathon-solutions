# Reusable Databricks Patterns

The building blocks every blueprint composes. Blueprints reference these by name (e.g., "Pattern B:
Parse → Chunk → Index → RAG") instead of re-explaining them. Think of this as the shared vocabulary.

> **Product-naming note (Sept 2026, facilitator-only - do NOT say this to SLHS).** The agent tier
> pivoted. The **Supervisor API is retired**, and the interactive **Agent Bricks** pieces
> (Knowledge Assistant, Multi-Agent Supervisor) are winding down - their functionality is folding
> into the **Genie** family (Genie Agents / agentic analytics) and **composable custom agents**.
> There is no customer-communicable EOL date, so we do **not** tell St. Luke's "Agent Bricks is
> deprecated." We just build on the current, GA stack: **Genie Agents**, **Genie One**,
> **Genie Code**, and **Unity AI Gateway**. Patterns below use the current names.
>
> Maturity as of Sept 2026: Genie Agents, Genie One, Genie Code, Unity AI Gateway = **GA** (safe to
> build on). Preview pieces to use with eyes open: Genie "Analyze Files in Volumes" = **Beta**,
> Genie Agent-mode APIs = **Private Preview**, Unity AI Gateway **Smart Routing** = **Beta**,
> Genie Ontology = **Gated Public Preview**.

---

## Pattern A - Medallion ingest (bronze → silver → gold)

**When:** any use case that starts from source-system extracts.
**How:** Auto Loader or batch read → bronze (raw, append) → silver (typed, deduped, conformed,
joined) → gold (business-ready aggregates / feature tables). Built with **Lakeflow Declarative
Pipelines** (streaming tables + materialized views) on serverless.
**Why it matters at SLHS:** this is the exact pattern that replaces the Health Catalyst pipelines
and the manual Power BI/Excel reconciliation. Show it once, reuse everywhere.

---

## Pattern B - Parse → retrieve → ground (unstructured knowledge)

**When:** answering questions over documents/policies/notes/transcripts (CKD notes; use cases 2, 4,
9, 11).
**How:** `ai_parse_document` (PDF/Word/images → structured text) → chunk → embed into **Vector
Search** index → retrieve top-k → ground an LLM answer with citations.
**Two ways to package it:**
- **Fast path - Genie Agent with "Analyze Files in Volumes" (Beta).** Point a Genie Agent at files
  in a UC Volume so it reasons over documents *alongside* your governed UC tables. No separate RAG
  app to build; great for the hackathon timebox.
- **Custom RAG agent** (ResponsesAgent/ChatAgent on **Model Serving**) when you need bespoke
  retrieval logic, re-ranking, or tool use beyond what the Genie Agent gives you.
**Governance:** answers cite source docs; UC controls which repositories/volumes a user can
retrieve from; LLM traffic governed by Unity AI Gateway (Pattern K).

---

## Pattern C - SQL-native AI enrichment (`ai_query` / `ai_extract` / `ai_classify`)

**When:** you need LLM logic *inside* a pipeline over structured/semi-structured rows at scale
(extract CKD stage from a note, classify an email theme, summarize an investigation).
**How:** call the AI function directly in a Lakeflow pipeline or SQL job. No endpoint to manage;
runs where the data lives; results land in a Delta column. Deterministic parts stay as SQL rules;
only the fuzzy step calls the model.
**Why:** cheapest, most governable way to add AI to a batch pipeline. Great "aha" for a Fabric shop.

---

## Pattern D - Genie Agent + Genie One (governed natural-language querying)

**When:** business users ask NL questions over trusted tables (HTM "what anesthesia machines need
replacement in 2026?", diversion peer benchmarks, nursing staffing).
**How:** curate a gold-layer schema + **Metric Views** for governed KPIs → build a **Genie Agent**
(fka Genie Space) with instructions, sample SQL, and certified metrics → develop and embed it with
**Genie Code** or a **Databricks App**. Surface it to non-technical leaders through **Genie One**,
the unified business front door (the read-out / exec-consumption layer).
**Three Genies, three jobs:** *Genie Agents* = the governed, curated domain experience the data team
builds; *Genie One* = the business/exec surface where leaders ask questions and see dashboards;
*Genie Code* = the developer assistant that builds the pipeline, semantic layer, and agent.
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
notebook or job) produces a *draft* → an LLM agent explains changes in plain language and answers
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

## Pattern H - Composable multi-skill agent (orchestration)

**When:** a use case needs several specialized skills (retrieve a policy + query structured data +
draft a message). E.g., a "talk to my data" agent that both looks up a benefit policy (RAG) and
checks an employee's leave balance (SQL).
**How:** a **custom agent** (ResponsesAgent/ChatAgent) on **Model Serving** or a **Databricks App**
orchestrates the skills - it calls a **Genie Agent** for governed structured lookups, a retrieval
step / Genie "Analyze Files in Volumes" for unstructured knowledge (Pattern B), and custom tools via
**MCP**. Each skill is independently governed and evaluable (Pattern J), and all model calls route
through **Unity AI Gateway** (Pattern K).
**Direction of travel (facilitator context, not for SLHS):** the retired Supervisor API is replaced
by this composable / API-first / no-lock-in approach - build the orchestration yourself in an App
so there's no black-box abstraction between you and production.

---

## Pattern I - Serve it: Databricks App + Model Serving

**When:** any solution that needs a UI beyond a notebook (all five hackathon solutions can end here).
**How:** deploy the agent/pipeline behind **Model Serving**; build a **Databricks App**
(React/FastAPI) as the front end; use **OBO (on-behalf-of) auth** so the app respects each user's UC
permissions (never make the app SP an admin). Lakebase for any app-side transactional state. Route
the app's LLM/agent calls through **Unity AI Gateway** (Pattern K) for budgets, guardrails, and
audit.
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

## Pattern K - Govern & route AI traffic (Unity AI Gateway)

**When:** any solution that calls an LLM or agent - which is all five. This is the "Control" layer.
**How:** put **Unity AI Gateway** (GA; the rebrand of Mosaic AI Gateway, built on Unity Catalog) in
front of model/agent traffic. It gives you: **budgets** (per-user/group spend caps), **guardrails**
(content + identity service policies - the PHI safety net), **observability** (system tables for
requests, tokens, latency, cost attribution), **access control** (models, external providers, MCP
tools registered as UC securables), and **model routing** - including **Smart Routing (Beta)**, which
auto-picks the right model by task/complexity/permissions/budget and can cut task cost 30%+.
**Why it matters at SLHS:** this is the governance story a Copilot/Fabric stack can't tell - one
control plane for *who* can call *which* model, at *what* cost, with *what* content filters, all
audited. For a health system worried about PHI and AI spend, it's the difference between a demo and
something legal/security will let go to production.

---

## How blueprints compose these

A typical full solution = **A** (ingest) → **C or B** (AI enrichment / retrieval) → **E/F/G**
(the core analytical engine) → **D** (Genie Agent for exploration, Genie One for leaders) → **I**
(serve) → **J** (evaluate), with **K** (Unity AI Gateway) governing every model call throughout.
Not every use case uses every pattern; each blueprint names the exact chain.
