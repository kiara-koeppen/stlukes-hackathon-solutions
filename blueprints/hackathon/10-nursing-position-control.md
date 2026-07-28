# 10 Nursing Position Control & Workforce Forecasting Automation

**Group:** cross-group (highest-priority, most eyes on it) | **Requester:** Tim Dougherty, Aireal Horn, Caleb Benedict | **Problem archetype:** Multi-source entity reconciliation → forecasting | **Priority:** !! (highest in the candidate doc)

## 1. Problem & desired outcome

Nursing leaders maintain their workforce forecast in a heavily customized **spreadsheet** that has to be hand-reconciled every pay period against three moving sources: Power BI roster exports (HR), TAS scheduling/PTO validation, and the leaders' own future-staffing assumptions typed into Excel. Every reconciliation is manual, error-prone, and stale the moment it is saved. It cannot cleanly account for PTO, leaves of absence, shift changes, resignations, or planned hires, so a nurse manager like Aireal never has a trustworthy "do I have enough RNs on nights next pay period" number.

The target experience: roster, schedule, and HR-event data land in Databricks automatically; employee identity is **reconciled across systems** into one gold "position control" table at the department/shift/pay-period grain; a forecast projects available FTE against demand and flags shortages and surpluses per pay period; and nursing leaders get a Genie chat + a dashboard app instead of a spreadsheet.

## 2. Solution in one picture

```
 SOURCE SYSTEMS (synthetic)              DATABRICKS (Lakeflow + UC)                 DELIVERED EXPERIENCE
 ┌──────────────────────┐
 │ Power BI / HR roster │──┐
 │ hr_roster            │  │        ┌──────── BRONZE (raw append) ────────┐
 └──────────────────────┘  │        │ raw copies of all 5 sources          │
 ┌──────────────────────┐  │        └──────────────────┬───────────────────┘
 │ TAS schedule + PTO   │──┤                            ▼
 │ tas_schedule         │  │        ┌──────── SILVER (typed + conformed) ──┐
 └──────────────────────┘  ├──▶ A ──│  *** RECONCILIATION (the crux) ***   │
 ┌──────────────────────┐  │        │  fuzzy identity match across roster/ │
 │ PTO / LOA / resign   │──┤        │  TAS/HR → emp_crosswalk (surrogate   │
 │ pto_loa              │  │        │  employee key + confidence + review) │
 └──────────────────────┘  │        └──────────────────┬───────────────────┘
 ┌──────────────────────┐  │                            ▼
 │ Demand-based metrics │──┤        ┌──────── GOLD ────────────────────────┐
 │ demand_metrics       │  │        │ position_control_gold                │       ┌────────────────────┐
 └──────────────────────┘  │        │ available FTE vs required FTE by     │──▶ D ─│ Genie Space        │
 ┌──────────────────────┐  │        │ dept × shift × pay_period (current   │       │ "RN gap on nights  │
 │ Budgeted positions   │──┘        │ + future)                            │       │  next pay period?" │
 │ position_control     │           └──────────────────┬───────────────────┘       └────────────────────┘
 └──────────────────────┘                              ▼                            ┌────────────────────┐
                                     ┌──────── FORECAST (E) ────────────────┐──▶ I ─│ Databricks App     │
                                     │ ai_forecast supply & demand →        │       │ position-control   │
                                     │ staffing_forecast_gold: shortage /   │       │ dashboard (replaces│
                                     │ surplus FTE per future pay_period    │       │ the spreadsheet)   │
                                     └──────────────────────────────────────┘       └────────────────────┘
```

## 3. The chained architecture (step by step)

The core chain is **A → (reconciliation) → E → D → I**, with **C** doing the fuzzy-match work inside silver and **J** as the credibility layer. Reconciliation is the technical crux; the pay-period forecast is the business value.

**Stage 1 - Ingest all five sources into bronze. (Pattern A)**
*What happens:* the five synthetic source tables (`hr_roster`, `tas_schedule`, `pto_loa`, `demand_metrics`, `position_control`) land in bronze as raw appends. In production these are Auto Loader / Lakeflow Connect / Power BI export drops on a schedule.
*Feature:* **Lakeflow Declarative Pipelines** streaming tables on serverless.
*Why this vs. alternatives:* one declarative pipeline replaces the manual "export from Power BI, paste into Excel" step entirely, and gives lineage + incremental refresh for free. This is the same medallion motion that replaces Health Catalyst everywhere else in the hackathon, so it is worth showing once cleanly.

**Stage 2 - Conform each source in silver.**
*What happens:* type-cast, dedupe, standardize department/unit/role codes, normalize names (upper/trim, split first/last), parse dates. Each source becomes a clean typed silver table. Crucially the sources do **not** share a clean key: HR uses `employee_id` like `E04821`, TAS uses a `tas_worker_id` like `TAS-4821` plus a free-text name, and some workers appear in only one system.
*Feature:* Lakeflow materialized views (Pattern A silver layer).
*Why:* keeps the messy reconciliation logic isolated in one place (Stage 3) instead of smeared across every downstream query.

**Stage 3 - Reconcile employee identity across systems. (THE CRUX - Pattern A silver + Pattern C)**
*What happens:* build an `emp_crosswalk` table that maps every source identifier to a single surrogate `employee_key`. Deterministic matches first (normalized ID, exact normalized name + department); then fuzzy matching for the rest - Jaro-Winkler / Levenshtein on name, nickname resolution (Bob↔Robert, Liz↔Elizabeth), plus department/role as a tie-breaker. Each match gets a **confidence score** and a **match_method**; low-confidence and one-sided records are flagged `needs_review` rather than silently dropped or wrong-joined.
*Feature:* SQL string functions + Python UDF (rapidfuzz/jellyfish) for the deterministic + fuzzy tiers, and **`ai_query` / `ai_classify` (Pattern C)** for the genuinely ambiguous residue - e.g. "are 'Kathy Nguyen RN, Med-Surg' and 'Katherine Nguyen, 4West' the same person?" as a last-resort LLM adjudication with a reason string.
*Why this vs. alternatives:* a plain join on `employee_id` produces **confidently wrong** staffing numbers the moment IDs disagree - which is exactly the spreadsheet failure mode today. A tiered matcher (cheap deterministic → fuzzy → LLM only on the hard residue) is both accurate and cheap, and the confidence/review flag is what lets a nurse leader trust the total. This is the single most important thing a team can nail.

**Stage 4 - Build the gold position-control table.**
*What happens:* using `emp_crosswalk`, join reconciled roster + schedule + PTO/LOA + budgeted positions to compute **available FTE** by `department × shift × pay_period` - starting from budgeted/rostered FTE, then subtracting PTO/LOA/terminations effective in that pay period and adding scheduled future hires and known shift changes. Result: `position_control_gold`, one row per dept/shift/pay-period with `budgeted_fte`, `available_fte`, `required_fte` (from demand), and `net_fte = available − required`.
*Feature:* Lakeflow gold materialized view + a **Metric View** over it for governed KPIs (available FTE, coverage ratio, net gap).
*Why:* the pay-period grain is the whole point - leaders plan by pay period, not by day. Encoding it once in gold means Genie, the forecast, and the app all agree on the number.

**Stage 5 - Forecast supply and demand into future pay periods. (Pattern E)**
*What happens:* project `available_fte` and `required_fte` forward across the next several pay periods, then compute forecasted `shortage_fte` / `surplus_fte` per dept/shift/pay-period. Attrition trend (from historical terminations) and known future events (scheduled hires, dated LOAs) feed the supply side.
*Feature:* **`ai_forecast`** for the quick path (pure SQL over the gold time series, no model to manage), graduating to an **MLflow-logged model** when leaders want covariates (seasonality, census-driven demand) and scenario knobs ("what if two RNs resign?").
*Why this vs. alternatives:* `ai_forecast` gets a credible shortage/surplus curve in one SQL statement during the hackathon; MLflow is the honest production upgrade path. Either way the output is a `staffing_forecast_gold` table the app reads.

**Stage 6 - Let nursing leaders ask questions. (Pattern D)**
*What happens:* a **Genie Space** over `position_control_gold` + `staffing_forecast_gold` + the Metric View, with curated instructions and sample SQL, so Aireal can ask "which units are short RNs on nights next pay period?" or "show me surplus CNAs in Med-Surg this quarter" in plain language.
*Feature:* **Genie Space** (Agent Bricks), embedded via Genie Code.
*Why vs. Copilot:* Genie answers over *governed* UC tables with lineage and row/column security and the certified Metric View definitions - not a spreadsheet formula nobody can audit, and not a Fabric black box.

**Stage 7 - Serve it as the spreadsheet replacement. (Pattern I)**
*What happens:* a **Databricks App** (React/FastAPI) dashboard: pay-period selector, department/shift heatmap of net FTE (red shortage / green surplus), drill-down to the roster behind a number, an embedded Genie panel, and a **reconciliation review queue** where a workforce analyst resolves the `needs_review` matches from Stage 3.
*Feature:* **Databricks App + Model Serving**, **OBO auth** so each leader sees only their departments, Lakebase for app-side state (review decisions, saved scenarios).
*Why:* this is the POC→prod step - an actual UI a nurse leader uses instead of maintaining the Excel file.

**Stage 8 - Evaluate & monitor. (Pattern J)**
*What happens:* an eval set of known employee pairs (true matches / true non-matches) scores reconciliation precision/recall; forecast accuracy is back-tested against held-out pay periods (MAPE). MLflow traces every LLM adjudication for audit.
*Feature:* **MLflow GenAI evaluation** + scorers.
*Why:* "how do you know the staffing number is right?" is the question leadership will ask; reconciliation precision + forecast MAPE is the answer.

## 4. Data model

All synthetic, generated by `synthetic_data/generators/gen_10_nursing.py`, written to `hackathon.shared.nursing_*`. Full DDL + column dictionary in `synthetic_data/schemas/10_nursing_schema.md`.

| Table | Grain | Key columns | Reconciliation role |
|---|---|---|---|
| `nursing_hr_roster` | one row per employee | `employee_id` (`E#####`), `first_name`, `last_name`, `department`, `unit`, `role` (RN/LPN/CNA/…), `fte`, `hire_date`, `term_date`, `status` | **Source of truth for identity.** Clean IDs. |
| `nursing_tas_schedule` | one row per employee × shift × pay_period | `tas_worker_id` (`TAS-####`), `worker_name` (free-text, **name variants**), `department`, `shift` (Day/Eve/Night), `pay_period`, `scheduled_hours`, `pto_hours` | **Different ID format + name variants** → must be fuzzy-matched to roster. Some workers here are missing from roster and vice-versa. |
| `nursing_pto_loa` | one row per leave event | `employee_id`, `leave_type` (PTO/LOA/RESIGNATION), `start_date`, `end_date`, `hours` | Effective-dated reductions to available FTE. |
| `nursing_demand_metrics` | one row per department × shift × pay_period | `department`, `shift`, `pay_period`, `required_fte`, `census`, `demand_method` | Enterprise demand-based staffing target (the "required" side). |
| `nursing_position_control` | one row per department × shift | `department`, `shift`, `budgeted_positions`, `budgeted_fte`, `role_mix` | Budgeted baseline the spreadsheet tracks against. |

**Engineered reconciliation challenges** (deliberate, so the crux is real): TAS uses `TAS-####` not `E#####`; ~30% of TAS names are variants of the roster name (nicknames, maiden/married, middle-initial, typos, "RN" suffix); a handful of employees exist in HR but not TAS (new hires not yet scheduled) and in TAS but not HR (contract/agency staff); a few duplicate near-identical names in different departments to force the department tie-breaker.

**Gold tables the teams build** (in their own group schema, not shared): `emp_crosswalk`, `position_control_gold`, `staffing_forecast_gold`, plus a Metric View.

**Governance shape:** roster/HR data is sensitive but synthetic. Department-level row filtering via `is_account_group_member` so a unit manager sees only their departments; name/ID columns are the join keys, not display fields, in the leader-facing views.

## 5. Governance & safety

- **No PHI, no real HR data** - this is synthetic employee data mirroring the shape of Power BI/TAS/HR. That is both the compliance requirement and the design constraint: reconciliation must work on de-identified shapes.
- **RBAC:** Unity Catalog row filters restrict each nurse leader to their own departments (`is_account_group_member`); the reconciliation review queue and full crosswalk are restricted to workforce analysts.
- **Human-in-the-loop on reconciliation:** low-confidence matches are **flagged for a human to confirm**, never auto-merged into a staffing number a leader will act on. The crosswalk carries `confidence` and `match_method` so every merged identity is auditable.
- **Human-in-the-loop on the forecast:** shortage/surplus numbers are *decision support* for a nurse leader, not automated staffing actions. No hire/termination is ever triggered by the system.
- **Audit:** UC system tables + MLflow traces log every LLM adjudication and every forecast run, so a number on the dashboard can be traced back to the records and the match decisions behind it.
- **HR-legal sensitivity:** resignations and LOAs are handled as effective-dated events, and the app never exposes an individual's leave reason - only its FTE impact.

## 6. What a team can realistically build in the hackathon (~1.5 days)

**Scope IN (the MVP that tells the whole story):**
1. Run the generator; land all five sources in bronze/silver via one Lakeflow pipeline.
2. **Build `emp_crosswalk` with a tiered matcher** - deterministic (normalized ID + name), then fuzzy (rapidfuzz on name + department tie-breaker), with a confidence score and a `needs_review` flag. *This is the part to nail; it is the differentiator.*
3. Build `position_control_gold` at dept × shift × pay_period (available vs required FTE).
4. One `ai_forecast` call to project shortage/surplus for the next 2–3 pay periods → `staffing_forecast_gold`.
5. A Genie Space over gold answering 4–5 leader questions, **or** a minimal Databricks App with the net-FTE heatmap. (Pick one surface to finish, not both.)

**Scope OUT (coach them off these):** the full MLflow forecasting model with covariates/scenarios; LLM adjudication for every ambiguous pair (do the top-of-funnel deterministic + fuzzy tiers first, LLM only if time remains); the reconciliation *review UI* (a table + flag is enough for the demo); OBO/row-level security polish; more than ~3 future pay periods.

**Suggested split:** one pair owns ingest + reconciliation (the crux), one owns gold + forecast, one owns Genie/app. Reconciliation is the critical path - start it first.

## 7. Path to production

- **Reconciliation hardening:** move the matcher into a Lakeflow pipeline step with the LLM-adjudication tier enabled; stand up the review queue in the app; measure precision/recall against a labeled set (Pattern J). This is the piece that determines whether leaders trust the tool, so it graduates first.
- **Real ingestion:** replace synthetic loads with Lakeflow Connect / Auto Loader against the real Power BI roster export, TAS extract, and HR-events feed; confirm refresh cadence lines up with the pay-period close.
- **Forecast upgrade:** `ai_forecast` → MLflow-logged model with census/seasonality covariates and scenario inputs; back-test MAPE and gate on it.
- **App:** OBO auth + UC row filters for per-department access; Lakebase for saved scenarios and review decisions; DABs for CI/CD across dev→prod.
- **Eval gates + monitoring:** reconciliation precision and forecast MAPE as release gates; MLflow trace monitoring in prod. Realistic story: a focused team ships reconciliation + gold + forecast + dashboard in a few weeks once real feeds are wired, which is exactly the kind of POC→prod win Molly's OKR wants.
- **Scope question to resolve:** confirm whether this is one nursing org or the broader ~120-department workforce-planning problem, and how the **future scheduling-platform migration** (TAS successor) changes the ingestion contract. Design the crosswalk to be source-agnostic so a new scheduling system is just another identifier column.

## 8. Competitive angle

Copilot-in-Excel and Fabric cannot **reconcile employee identity across three systems with a confidence-scored, auditable crosswalk** and then forecast on the reconciled result - that is exactly why the current process is a fragile hand-maintained spreadsheet. Databricks does the fuzzy matching at scale, keeps the pay-period math governed in one Metric View, forecasts in a single SQL function, and serves it with row-level security and full lineage from the dashboard number back to the source records. A nurse leader gets a number they can trust *and* trace.

## 9. Facilitation notes

- **This is the highest-priority use case (!!)** - most eyes on it. Spend your coaching time here.
- **The "aha" to steer toward:** show a team the *wrong* staffing number a naive `JOIN ... ON employee_id` produces (because TAS IDs don't match HR), then show the corrected number after the fuzzy crosswalk. That side-by-side is the demo money shot and the whole justification for the platform.
- **Where teams get stuck:** (1) they underestimate reconciliation and try to `JOIN` on a key that doesn't line up - push them to the crosswalk on hour one; (2) they lose the **pay-period grain** and start reasoning by day - keep dragging them back to dept × shift × pay_period; (3) they try to build the full MLflow model - redirect to `ai_forecast` for the MVP; (4) they try to finish both Genie *and* the app - make them pick one surface.
- **Reconciliation tiering discipline:** deterministic first (free, catches most), fuzzy second (rapidfuzz, catches the name variants), LLM adjudication only on the ambiguous residue. Teams that jump straight to an LLM for every row burn time and money.
- **Cheap win if they're ahead:** the `needs_review` flag + a confidence column turns a data-quality liability into a governance feature - great thing to show leadership.
- **Requesters in the room:** Tim Dougherty, Aireal Horn, Caleb Benedict. Frame the demo as "the spreadsheet, gone" and speak in their language: PTO, LOA, resignations, shift coverage, pay period.
