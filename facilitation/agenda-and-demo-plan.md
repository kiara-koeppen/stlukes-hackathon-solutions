# St. Luke's AI Workshop: Agenda, Demo Plan, and Build Plan

Facilitator playbook for the SLHS AI Dev Collaborative Workshop. Internal (do not share with SLHS).
Companion to `hackathon-guide.md` (group coaching, competitive backdrop) and the per-use-case
blueprints in `blueprints/hackathon/`.

- **When:** Wednesday Sept 16, 2026 (full day) + Thursday Sept 17, 2026 (half day). Boise, ID.
- **Shape:** Databricks drives the first half of Day 1 (demos). Teams build from Day 1 afternoon
  through Day 2, with an 11:00 leadership read-out on Day 2.
- **Databricks team:** Kiara (lead), Michael Comerford (AE), Mobeen Vaid (SA).
- **Attendees:** 12 SLHS participants in 3 mixed teams (Azure devs, data scientists, cyber
  engineers, a domain architect, an intern).

---

## TL;DR

We show the modern Databricks agent stack, then teams build one healthcare use case each on it. The
stack changed since this repo was first drafted: **Agent Bricks' interactive pieces (Knowledge
Assistant, Multi-Agent Supervisor) and the Supervisor API are winding down**, and the work has moved
into the **Genie family** plus **composable custom agents**, all governed by **Unity AI Gateway**.
So we demo and build on: **Genie Agents**, **Genie One**, **Genie Code**, and **Unity AI Gateway**,
backed by AI Functions, Vector Search, Model Serving, Databricks Apps, and MLflow eval.

Important: the Agent Bricks pivot is **internal only, not customer-communicable**. We never tell
SLHS "Agent Bricks is deprecated." We simply build on the current GA stack.

The Day-1 demo is a **hybrid**: one anchor use case end to end (HTM, non-PHI, safe to project),
then a capability tour mapped to the five use cases, with **Nursing Position Control (#10)** carried
as the north-star story because leadership most wants it in production.

---

## The workshop stack (verified Sept 2026)

Sourced from internal Slack (PM/product channels) and Databricks release notes, not training data.

| Layer | Product | What it does here | Maturity |
|---|---|---|---|
| **Build assistant** | **Genie Code** | The AI coding assistant teams use to build pipelines, semantic layers, ERDs, and agents. This is *how they build* during the hackathon. | **GA** (150 DBU/user/mo free; 25% off through Jan 31 2027) |
| **Governed analytics** | **Genie Agents** (fka Genie Spaces) | Curated, governed natural-language Q&A over UC tables + Metric Views. The "talk to your data" surface each team builds. | **GA** |
| **Unstructured in Genie** | Genie Agents: **Analyze Files in Volumes** | Genie reasons over documents/notes in a UC Volume alongside tables. Key for CKD chart notes. | **Beta** |
| **Business/exec surface** | **Genie One** | Unified front door where a non-technical leader asks questions and sees dashboards. The read-out consumption layer. | **GA** (free through Jan 31 2027) |
| **AI governance** | **Unity AI Gateway** (fka Mosaic AI Gateway) | One control plane over every model/agent call: budgets, PHI guardrails, model routing, full audit. The governance story Fabric/Copilot can't tell. | **GA**; **Smart Routing** = Beta |
| **SQL-native AI** | AI Functions (`ai_query`, `ai_extract`, `ai_classify`, `ai_parse_document`, `ai_forecast`) | LLM logic inside pipelines at scale, no endpoint to manage. | **GA** |
| **Retrieval** | Vector Search | RAG index for custom retrieval beyond Genie's file analysis. | **GA** |
| **Custom agents** | Composable agents (ResponsesAgent/ChatAgent) on **Model Serving** / **Databricks Apps**, tools via **MCP** | Orchestrate Genie Agent + retrieval + tools yourself. Replaces the retired Supervisor API. | **GA** |
| **Serving surface** | **Databricks Apps** (React/FastAPI, OBO auth) | The reviewer/leader UI. Turns a POC into something a nurse leader actually uses. | **GA** |
| **Quality** | **MLflow** `genai.evaluate()` | The "how do you know it's right?" credibility layer. | **GA** |
| **Data foundation** | Unity Catalog, Lakeflow Declarative Pipelines, Metric Views, Delta, Serverless | Medallion ingest + governed KPIs everything sits on. | **GA** |

Pattern reference (`reference/databricks-patterns.md`) maps each of these into the reusable
building blocks A through K that the blueprints compose.

---

## Day 1: Wednesday Sept 16 (full day)

| Time (MT) | Block | Owner |
|---|---|---|
| 9:00 - 9:20 | Welcome, goals, why-we're-here (tie to Molly's AI OKR: POC to prod) | Kiara + Yutong |
| 9:20 - 10:15 | **Anchor demo: HTM end to end** (see script below) | Kiara |
| 10:15 - 10:30 | Break | |
| 10:30 - 11:30 | **Capability tour** mapped to the 5 use cases (see script below) | Kiara + Mobeen |
| 11:30 - 12:00 | **Environment onboarding**: sandbox login, `00_setup_catalog`, load your use case's synthetic data, verify Genie Code + Model Serving access | Mobeen (Kiara floats) |
| 12:00 - 1:00 | Lunch | |
| 1:00 - 4:15 | **Team build sprint 1.** Goal by EOD: data loaded, medallion started, the core "engine" of the use case scoped (rules/constraints/features/Genie Agent) | Teams; Kiara + Mobeen float |
| 4:15 - 4:30 | Stand-up: each team states tomorrow's demoable slice | All |

## Day 2: Thursday Sept 17 (half day)

| Time (MT) | Block | Owner |
|---|---|---|
| 8:30 - 11:00 | **Build sprint 2** to a demoable slice. Push for *something* end to end over one polished piece. | Teams; Kiara + Mobeen float |
| 11:00 - 12:00 | **Leadership read-out** (Molly, Rex + leaders). Each group demos. Kiara frames the through-line: what got built in ~1.5 days and the concrete path to production for the 1-2 strongest. | Teams + Kiara |

---

## Day-1 demo script

### Part A: Anchor deep-dive, HTM (#14), ~55 min

Non-PHI, safe to project, and the cleanest Genie story. Walk it live, end to end:

1. **Ingest to medallion** (Pattern A): TMS asset extract, bronze to silver to gold with Lakeflow
   Declarative Pipelines. Point out this is what replaces Health Catalyst pipelines + manual Power
   BI/Excel reconciliation.
2. **Metric Views** (Pattern D): governed KPIs (% past end-of-service, replacement spend by year).
   Show that the metric is defined once and trusted everywhere.
3. **The money shot, Genie Agent** (Pattern D): ask "What anesthesia machines need replacement in
   2026?" in plain English over governed tables. Then ask a follow-up. This is the Genie-vs-Copilot
   moment: governed UC tables with lineage and row/column security, not a black box.
4. **Genie One** (Pattern D): open the same content as a *leader* would, in the unified Genie One
   surface. This is what Molly/Rex would actually touch. It sets up the Day-2 read-out.
5. **Unity AI Gateway** (Pattern K): show that every model call above ran through the Gateway,
   budgeted, guardrailed, and audited. This is the "will security and legal let this ship?" answer.

### Part B: Capability tour, ~60 min (concrete, each tied to a use case)

Keep each to a tight beat. The point is "here is your toolkit for the next day and a half."

| Capability | The beat | Ties to |
|---|---|---|
| **Genie Code** | Build a small pipeline / semantic layer live by prompting. This is *how you'll build.* | All teams |
| **Genie Agents + Analyze Files in Volumes** | Genie answering over a document in a Volume alongside tables. | CKD notes (#1) |
| **AI Functions in SQL** | `ai_extract` pulling a field from a note; `ai_forecast` projecting a series. | CKD (#1), Scheduling (#6), Nursing (#10) |
| **Anomaly + narrative** | Rules-first flags, then `ai_query` writing the plain-language risk narrative. | Diversion (#12) |
| **Optimization + human-in-loop** | Solver drafts a schedule, agent explains "why", human approves in an App. | Scheduling (#6) |
| **Composable agent on Model Serving / Apps** | A custom agent calling a Genie Agent + a tool. Note: you orchestrate it (no Supervisor black box). | Scheduling (#6), stretch on any |
| **Databricks Apps** | The reviewer/leader cockpit with OBO auth. | All (the POC-to-prod surface) |
| **MLflow eval** | `genai.evaluate()` scoring an answer for correctness/groundedness. | CKD (#1), Diversion (#12) |
| **Unity AI Gateway** | Budgets + guardrails + audit across all of the above. | All (the governance close) |

### Part C: Environment + starter repo, ~30 min

Show the `stlukes-hackathon-starter` repo, run `00_setup_catalog`, load synthetic data, and confirm
each person has Genie Code + Model Serving access. Everyone leaves lunch ready to build.

**North-star thread:** throughout Parts A and B, keep pointing back to **Nursing Position Control
(#10)** as the use case leadership most wants in production, so the Day-2 read-out narrative
connects to Molly's OKR even though the safe-to-project anchor is HTM.

---

## What each team builds

Five use cases, three teams. Assignments below reflect the last planning email; **#10's final home
is still open and is a prep-call item** (see open items). Full architectures live in
`blueprints/hackathon/`; each blueprint's "What a team can realistically build" section is the MVP
scope to hold teams to.

| # | Use case | Group | Problem archetype | Pattern chain | MVP demoable slice |
|---|---|---|---|---|---|
| **12** | **Diversion Support** | **Group 1** (Yutong, Gabe, Jerome, Shawn) | Anomaly detection + narrative report | A -> C/G -> D -> J, governed by K | Planted-pattern flags + an `ai_query` risk narrative + a Genie Agent for peer benchmarks. RBAC + audit front and center. |
| **6** | **Hospitalist Scheduling** | **Group 2** (Drake, Clara, Justin, Terry) | Constrained optimization + human-in-loop | A -> F -> D/I, governed by K | Solver produces a draft schedule; a composable agent explains changes; a thin App or Genie Agent shows coverage/equity. |
| **1** | **CKD Risk Flagging** | **Group 3** (Tyler, Chris, Max, Carter) | Clinical extraction + rules-based flagging | A -> C/B -> D -> J, governed by K | KDIGO staging in SQL, one `ai_extract`/Analyze-Files touch on notes, a Genie Agent finding lab-evidence patients with no N18.x code (the care gap). |
| **14** | **NextGen HTM** | **Group 3** (secondary) or Day-1 anchor | Asset analytics + NL query + forecasting | A -> E -> D -> I, governed by K | Gold asset table + Metric Views + a Genie Agent answering the replacement-planning questions. Easiest to make demo-ready. |
| **10** | **Nursing Position Control** | **Highest priority; assignment open** | Multi-source reconciliation + forecasting | A -> C -> E -> D -> I, governed by K | Reconcile roster/TAS/HR (synthetic data has planted mismatches), `ai_forecast` net FTE by pay period, a Genie Agent or net-FTE heatmap App for leaders. |

**Recommendation on #10:** it is the one leadership most wants in production, but it is
reconciliation-heavy and less safe to project than HTM. Keep it as the **read-out north star** and
either (a) fold it into Group 2 as a stretch (scheduling-adjacent), or (b) run it as the cross-group
"if you finish early" target. Decide on the prep call. Do not let it go invisible.

---

## Feature to use-case matrix

Which product each team will actually touch. G = governed by Unity AI Gateway (all of them).

| Feature | CKD #1 | Scheduling #6 | Nursing #10 | Diversion #12 | HTM #14 |
|---|:--:|:--:|:--:|:--:|:--:|
| Lakeflow Declarative Pipelines | Y | Y | Y | Y | Y |
| Metric Views | Y | Y | Y | Y | Y |
| Genie Code (build) | Y | Y | Y | Y | Y |
| Genie Agent (analytics) | Y | Y | Y | Y | Y |
| Genie One (leader surface) | opt | opt | **Y** | opt | Y |
| Analyze Files in Volumes | **Y** | | | opt | |
| AI Functions (`ai_extract`/`ai_forecast`/`ai_query`) | Y | Y | Y | Y | Y |
| Vector Search / RAG | opt | | | opt | |
| Solver (OR-Tools/PuLP) | | **Y** | | | |
| Composable agent (Model Serving/App) | opt | Y | opt | opt | opt |
| Databricks Apps | opt | Y | Y | opt | opt |
| MLflow eval | **Y** | opt | opt | **Y** | opt |
| Unity AI Gateway | G | G | G | G | G |

---

## Why Databricks, not Copilot/Fabric (thread this all event)

SLHS is deeply Microsoft-invested (Azure, Power BI, Epic, Copilot Studio via Avanade); **Fabric is
the real competitive risk**. Every "aha" should implicitly answer *why Databricks*:

1. **Governed Genie Agents over real UC tables** (lineage, row/column security), not a black-box
   answer of unknown provenance.
2. **Unity AI Gateway as one control plane** over every model call: budgets, PHI guardrails, audit.
   A Copilot/Fabric stack cannot show *who* called *which* model at *what* cost with *what* filters.
3. **Constraint optimization** (Scheduling #6) that Copilot-in-Excel structurally cannot hold.
4. **MLflow eval** as the honest answer to "how do you know it's right?" for clinical use cases.
5. **A clean POC-to-prod path** (DABs, Apps, Model Serving) that ties directly to Molly's AI OKR of
   moving solutions to production.

---

## Prep checklist and open items

- [ ] **Confirm final use-case-to-group assignments**, especially where **#10 Nursing** lands.
- [ ] **Real metadata** (`catalog / schema / table / column / type / count`) for all 5 use cases
      (requested from Yutong 7/28). Feed corrections into the starter-repo schemas so synthetic
      shapes match reality.
- [ ] **Sandbox provisioning verified**: AD group `R-AiDevCollab_GG_AP` populated; per-group
      catalogs/schemas created; **Genie Code + Model Serving/FMAPI + Unity AI Gateway enabled**;
      participants auto-onboarded.
- [ ] **Synthetic data generated and loaded** for all 5 use cases (starter repo generators run).
- [ ] Confirm the hospitalist 3rd-party tool (Lightning Bolt?).
- [ ] Confirm room, timing, and which leaders make the 11:00 Day-2 read-out.
- [ ] Rehearse the HTM anchor demo end to end on the actual sandbox before Day 1.
- [ ] Propose a prep call ~1 week out (week of Sept 8, MT).

---

*Product facts verified Sept 2026 via internal Slack (PM/product channels) and Databricks release
notes. Agent Bricks pivot details are facilitator-only and not customer-communicable.*
