# 14 NextGen HTM Equipment Planning

**Group:** Group 3 (CKD / HTM) | **Requester:** Justin Malsam, Ryan Walker (Health Technology Management) | **Problem archetype:** Pattern D – governed natural-language querying (+ Pattern E forecasting) | **Priority:** !! (Day-1 demo anchor: non-PHI, safe to project to the whole room)

## 1. Problem & desired outcome

Justin and Ryan's Health Technology Management (HTM) team plans equipment replacement for the whole health system off a **14,000+ row TMS asset-inventory export**, and today that planning is a manual filtering exercise: open the file, filter by device type, then by facility, then by clinical area, eyeball which devices are nearing end-of-support, cross-check vendor service timelines, and hand-assemble a replacement picture to bring to Finance and clinical leadership. It is slow, it is annual-plus-ad-hoc, and the effort goes into wrangling the spreadsheet rather than into strategy. The target experience: an HTM planner asks in plain language, "What anesthesia machines need replacement in 2026?" and gets a trustworthy, governed answer in seconds; the same governed data drives a replacement-volume-and-spend forecast by year, facility, and clinical area, plus a planning dashboard that lets HTM shift from data wrangling to prioritization, budgeting, and vendor engagement.

This is the **natural-language-querying showcase** and the **Day-1 demo anchor**. It is non-PHI (asset metadata, not patient data), so it is the one solution safe to project to the entire room, and it is where the Genie-versus-Copilot governance story lands hardest against the Fabric risk.

## 2. Solution in one picture

```
SOURCE SYSTEMS                DATABRICKS (hackathon catalog, Unity Catalog governed)               DELIVERED EXPERIENCE
                    ┌──────────────────────────────────────────────────────────────────┐
TMS asset export ──┤ BRONZE  raw assets + work orders (Pattern A: Lakeflow, serverless) │
 (14k+ assets)     │   │                                                                │
TMS work orders ───┤   ▼                                                                │
                    │ SILVER  typed, deduped; work orders joined to assets by           │
                    │   │      asset_number; conformed device_type / facility /         │
                    │   │      clinical_area; parsed EOS + OS-EOL fields                 │
                    │   ▼                                                                │
                    │ GOLD  htm_asset_gold (one row / asset) with DERIVED planning       │
                    │   │    fields: years_to_eos, is_past_eos, os_eol, risk_tier,       │──┐
                    │   │    lifetime WO count / cost / downtime, replacement_cost       │  │
                    │   ▼                                                                │  │  ┌────────────────────────┐
                    │ METRIC VIEWS  governed KPIs: % past EOS, replacement spend by      │  ├─▶│ Genie Agent (Pattern D) │
                    │   │    year, count due by year × facility × clinical_area          │──┤  │ "What anesthesia machines│
                    │   ▼                                                                │  │  │  need replacement in 2026?"│
                    │ ai_forecast (Pattern E)  replacement volume & spend by year,       │  │  └────────────────────────┘
                    │   │    facility, clinical_area → htm_forecast_gold                 │──┤  ┌────────────────────────┐
                    │   ▼                                                                │  ├─▶│ AI/BI Dashboard + App   │
                    │ SCENARIO MODEL (optional)  budget-cap / defer-a-year knobs         │──┘  │ (Pattern I) HTM planners │
                    └──────────────────────────────────────────────────────────────────┘     └────────────────────────┘
                         Access: HTM planner UC group. Non-PHI. Synthetic in the hackathon. Lineage + audit throughout.
```

## 3. The chained architecture (step by step)

The core chain is **A → gold (derived fields) → Metric Views → D (Genie) → E (ai_forecast) → I (app/dashboard)**, with an optional scenario-modeling layer. Genie is the star; everything upstream exists to make the natural-language answer trustworthy, and everything downstream turns the trusted data into a plan.

**Stage 1 – Ingest the TMS asset and work-order tables. (Pattern A: Medallion ingest)**
*What happens:* batch-read the two synthetic TMS extracts (`htm_assets`, `htm_work_orders`) into bronze, then conform to silver with **Lakeflow Declarative Pipelines** on serverless. Silver does the typing and the join that the planner does by hand: parse the end-of-support and create dates, standardize `device_type` / `facility` / `clinical_area` spellings, dedupe on `asset_number`, and left-join work orders to assets on `asset_number` (kept as a left join so an asset with zero work orders still appears).
*Feature:* Lakeflow Declarative Pipelines (streaming tables + materialized views).
*Why this vs. a one-off notebook:* this is the exact bronze/silver/gold motion that replaces the Health Catalyst pipelines everywhere else in the hackathon, it is re-runnable when the quarterly TMS export refreshes, and the lineage it produces is itself an audit artifact. TMS data is not in Databricks yet ("coming soon"), so showing the clean ingest path is also the credibility story for the eventual real feed.

**Stage 2 – Build the gold asset table with derived planning fields.**
*What happens:* aggregate the joined silver down to **one row per asset** (`htm_asset_gold`) and compute the fields the planner cannot easily derive in a spreadsheet:
- `years_to_eos` = months between today and `end_of_support_date`, in years (negative if already past).
- `is_past_eos` = boolean, `end_of_support_date` earlier than today.
- `os_eol` carried through and combined into risk (a device on Windows 7/XP embedded is a security exposure as well as a support exposure, exactly the Medigate angle).
- `wo_count_12mo`, `repair_cost_12mo`, `downtime_hours_12mo` rolled up from work orders (support burden rises as an asset ages).
- `risk_tier` = a deterministic `CASE` combining `is_past_eos`, `years_to_eos`, `os_eol`, and repair burden into Critical / High / Medium / Low.
*Feature:* Lakeflow gold materialized view (Pattern A gold layer); the derivations are plain, auditable SQL.
*Why:* the derived fields are what turn a raw inventory dump into a planning table. Computing `risk_tier` once in gold means Genie, the metric views, the forecast, and the app all agree on it. This is the "single trusted definition" that a spreadsheet can never guarantee.

**Stage 3 – Define governed KPIs as Metric Views.**
*What happens:* create **Unity Catalog Metric Views** over `htm_asset_gold` that encode the numbers leadership will quote: percent of assets past end-of-support, count of assets due for replacement by year, replacement spend by year, spend and count by facility, spend and count by clinical area, and percent of the fleet on an end-of-life operating system. Dimensions: `facility`, `clinical_area`, `device_type`, `manufacturer`, `eos_year`, `risk_tier`. Measures: `asset_count`, `past_eos_pct`, `replacement_spend`, `avg_years_to_eos`, `eol_os_pct`.
*Feature:* Metric Views (certified metric definitions in YAML).
*Why this vs. metrics-in-each-query:* a Metric View is the certified semantic layer. Genie answers, the dashboard, and any ad-hoc SQL all pull the *same* definition of "replacement spend," so the CFO number and the Genie number never disagree. This is the governed contract that makes natural-language querying safe.

**Stage 4 – Genie Agent for governed natural-language querying. (Pattern D – the star)**
*What happens:* build a **Genie Agent** over `htm_asset_gold` plus the Metric Views, curated with instructions and sample SQL so an HTM planner can ask, in plain English:
- "What anesthesia machines need replacement in 2026?" (the headline moment)
- "Show me all Critical-risk imaging equipment at Magic Valley."
- "Which clinical areas have the most devices past end-of-support?"
- "What's our projected replacement spend by facility next year?"
- "List devices still running Windows 7."
Curate the space with: certified metric definitions, a glossary (EOS = end-of-support, PM = preventive maintenance, HTM), sample SQL for the archetypal questions, and instructions that scope answers to the governed gold table.
*Feature:* **Genie Agent**, embedded via Genie Code or a Databricks App panel.
*Why vs. Copilot / Fabric:* Genie answers over *governed Unity Catalog tables* with lineage, row/column security, and the certified Metric View definitions. Every answer is a SQL query the planner can inspect, not a black-box summary. When someone asks "what needs replacement in 2026," the answer is reproducible, auditable, and traceable back to source. This is the headline differentiator for the whole hackathon and the reason #14 anchors Day 1.

**Stage 5 – Forecast replacement volume and spend. (Pattern E)**
*What happens:* from `htm_asset_gold`, build a time series of assets reaching end-of-support per `eos_year` (and per facility, per clinical area), with `replacement_cost` attached, then project **replacement volume and replacement spend** forward across the next several years into `htm_forecast_gold`. Because end-of-support dates are known, the near-term forecast is close to deterministic (a roll-forward), and `ai_forecast` extends the curve and smooths facility/clinical-area splits.
*Feature:* **`ai_forecast`** for the quick path (pure SQL over the gold time series, no model to manage); graduate to an **MLflow-logged model** if leaders want covariates (budget constraints, utilization-driven early replacement) and scenario knobs.
*Why this vs. alternatives:* `ai_forecast` produces a credible year-by-year replacement-spend curve in one SQL statement during the hackathon, and the output is a `htm_forecast_gold` table the dashboard and Genie both read. It turns "here is what is expiring" into "here is what to budget."

**Stage 6 – Serve it to HTM planners. (Pattern I)**
*What happens:* an **AI/BI Dashboard** (fastest path) or a **Databricks App** (React/FastAPI) for the planners: a replacement-pipeline view (count and spend by year), a facility × clinical-area heatmap of past-EOS and Critical-risk counts, an end-of-life-OS security tile, drill-down to the asset list behind any number, and an **embedded Genie panel** for the natural-language questions. **OBO auth** so each planner sees only the facilities they are authorized for.
*Feature:* **AI/BI Dashboard** and/or **Databricks App + Model Serving**, OBO auth, Lakebase for app-side state (saved scenarios, replacement decisions).
*Why:* this is the POC-to-production step: it turns the query into the tool the HTM team actually plans from, replacing the manual spreadsheet filter-and-eyeball cycle.

**Stage 7 – Scenario modeling (optional stretch).**
*What happens:* let a planner apply a **budget cap** ("we only have $4M for imaging next year, what gets deferred and what is the added risk?") or **defer-a-year** knobs, and see the downstream effect on risk tier and next-year spend. The deterministic risk model plus the forecast make this a re-computation, not a guess.
*Feature:* Databricks App state (Lakebase) + a SQL/notebook recompute; `ai_query` can narrate the trade-off in plain language.
*Why:* this is the "strategic planning" the requesters asked for, and the clearest demonstration that the platform does more than answer questions: it supports the budgeting-and-prioritization decision.

## 4. Data model

Synthetic tables generated by `synthetic_data/generators/gen_14_htm.py`, schema documented in `synthetic_data/schemas/14_htm_schema.md`. Both live in `hackathon.shared.htm_*` (read-only source); groups write gold, metric views, forecast, and app artifacts to their own schema.

| Table | Grain | Key columns | Notes |
|---|---|---|---|
| `htm_assets` | one row / asset | `asset_number`, `description`, `device_type`, `manufacturer`, `model`, `serial_number`, `facility`, `clinical_area`, `purchase_date`, `create_date`, `end_of_support_date`, `vendor_support_notes`, `operating_system`, `os_eol`, `medigate_device_class`, `mac_address`, `ip_address`, `replacement_cost`, `asset_status` | ~14,000 assets across 8 facilities and ~13 clinical areas. `end_of_support_date` is deliberately spread (some past, a large 2026 cohort, some future) so "what needs replacement in 2026?" returns a satisfying answer. Recognizable device types (anesthesia machines, infusion pumps, ventilators, CT/MRI/ultrasound imaging) so NL queries feel real. |
| `htm_work_orders` | one row / work order | `wo_id`, `asset_number`, `wo_date`, `wo_type` (PM / Repair), `cost`, `downtime_hours`, `technician` | ~5–6 work orders per asset. Work-order **frequency and cost correlate with asset age**: older assets near or past end-of-support accumulate more (and pricier) repairs and more downtime, so the support-burden signal is real. Joins to `htm_assets` on `asset_number`. |

**Derived gold columns the teams build** (in their own schema): `years_to_eos`, `is_past_eos`, `risk_tier`, `wo_count_12mo`, `repair_cost_12mo`, `downtime_hours_12mo`, `eos_year`, plus the Metric Views and `htm_forecast_gold`.

**Governance shape of the data:** this is **non-PHI** asset metadata, which is exactly why it is the safe Day-1 demo. The one governance nuance worth showing is `facility` as a row-filter dimension (a planner at one facility sees only their fleet via `is_account_group_member`) and the `mac_address` / `ip_address` fields as a nod to the Medigate device-security angle: these are network-connected clinical devices, and the end-of-life-OS metric is a real security exposure, not just a support one.

## 5. Governance & safety

- **Non-PHI, synthetic in the hackathon.** No patient data is involved: this is equipment inventory. That is precisely why it is projected to the whole room on Day 1. The synthetic generator mirrors the shape of the real TMS export (which is last quarter's prod data, not yet in Databricks).
- **Governed single source of truth.** The whole point versus a shared spreadsheet: `risk_tier`, `replacement_spend`, and "due in 2026" are defined **once** in gold and the Metric Views, so Genie, the dashboard, and Finance all quote the same number, with lineage back to the TMS source.
- **RBAC by facility.** Unity Catalog row filters (`is_account_group_member`) can scope a planner to their facilities; the full-system view is for HTM leadership. Straightforward to show even in the MVP.
- **Genie answers are inspectable and auditable.** Every Genie response is a SQL query over governed tables that the planner can open and verify, and every access is captured in UC system audit tables. This is the honest answer to "how do I know the replacement list is right?"
- **Human-in-the-loop planning.** The platform surfaces what is expiring, ranks risk, and forecasts spend; the HTM team, Finance, and clinical leadership make the replacement and budget decisions. Nothing is auto-ordered.
- **Device-security angle (Medigate).** The `os_eol` metric and the network fields make the "end-of-life operating system on a networked clinical device" exposure visible: a governance win a spreadsheet filter does not surface.

## 6. What a team can realistically build in the hackathon

**Scope IN (the ~1.5-day MVP, and the Day-1 demo):**
1. Run the generator; land `htm_assets` + `htm_work_orders` in bronze/silver via a Lakeflow pipeline, with the work-order-to-asset join (Pattern A).
2. Build `htm_asset_gold` with the derived fields, above all `years_to_eos`, `is_past_eos`, `eos_year`, and a deterministic `risk_tier` (Stage 2).
3. Define **2–3 Metric Views**: percent past EOS, replacement count by year, replacement spend by year (Stage 3).
4. Build the **Genie Agent** and get the headline questions answering cleanly, starting with "What anesthesia machines need replacement in 2026?" (Stage 4). *This is the part to polish: it is the demo.*
5. One `ai_forecast` call projecting replacement spend/volume by year → `htm_forecast_gold` (Stage 5).
6. A minimal **AI/BI Dashboard** with the replacement-pipeline chart and the facility/clinical-area heatmap, or just the embedded Genie panel (Stage 6).

**Scope OUT (don't drown here):** the full Databricks App with OBO row-level security (a dashboard or Genie embed is enough for the demo); scenario modeling (Stage 7 is a stretch); an MLflow forecasting model with covariates (use `ai_forecast`); more than a handful of Metric Views. Steer teams to make the **Genie moment land** on the anesthesia-2026 question rather than polishing any single upstream stage.

## 7. Path to production

- **Real TMS feed:** TMS data is "coming soon" to Databricks. Replace the synthetic loads with Lakeflow Connect / Auto Loader against the real quarterly TMS export; the medallion shape and the join stay identical. HTM data eventually lands in ServiceNow (next year), so design the ingest to accept a second source without reshaping gold.
- **Confirm the work-order join:** the candidate doc flags whether work orders reliably join to asset records. Validate the `asset_number` key on real data before trusting the support-burden fields; keep the left join so unmatched assets still appear.
- **Package as a DAB:** databricks.yml with the pipeline, Metric Views, Genie Agent, forecast job, and dashboard/app as bundle resources for one-command redeploy across dev to prod.
- **Genie curation with SMEs:** co-develop the Genie instructions, glossary, and certified metrics with Justin and Ryan so device-type and clinical-area vocabulary matches how HTM actually talks.
- **Forecast upgrade:** `ai_forecast` to an MLflow-logged model when leaders want budget-constrained and utilization-driven scenarios; back-test against known replacement history.
- **RBAC + app:** OBO app with per-facility row filters; Lakebase for saved scenarios and replacement decisions.
- **Confirm the org context:** business sponsor, broader user group, decision bodies (HTM, Finance, clinical leadership), review cadence, and the data-ownership/verification team.

## 8. Competitive angle

Copilot Studio over Fabric can filter a spreadsheet and even summarize it, but it cannot answer "what anesthesia machines need replacement in 2026?" over a **governed, certified, lineage-tracked asset table where the answer is an inspectable SQL query, the replacement-spend definition is identical everywhere it is quoted, and every access is audited**, then forecast next year's replacement spend in the same governed layer and serve it with per-facility row security. Genie versus Copilot is the whole story here: one answers over trusted Unity Catalog data with a definition Finance signed off on; the other summarizes whatever file it was pointed at. For a Microsoft-invested shop weighing Fabric, this is the clearest, lowest-risk (non-PHI) demonstration of the difference.

## 9. Facilitation notes

- **This is the Day-1 anchor. It runs first and it is projected to the whole room.** It is non-PHI, so there is no compliance hesitation about showing it live. Treat this blueprint as the demo-script backbone.
- **The aha moment to steer toward:** type "What anesthesia machines need replacement in 2026?" into Genie and watch it return a clean, governed list, then open the generated SQL to show it is inspectable and traceable. Immediately contrast: "Copilot would summarize a file; this is querying a governed table with a certified definition." That single moment is the competitive story for the entire event.
- **Where teams will get stuck:** (1) skipping the derived-fields discipline and trying to make Genie compute `years_to_eos` on the fly, which produces inconsistent answers. Push them to bake `is_past_eos`, `eos_year`, and `risk_tier` into gold first. (2) Under-curating the Genie Agent: without instructions, a glossary, and sample SQL, Genie guesses. Coach them to curate before they demo. (3) Losing time on the app when the dashboard or Genie embed is enough.
- **Make the numbers satisfying:** the generator seeds a large 2026 end-of-support cohort on purpose, so the headline question returns a meaningful answer. Point teams at the generator's printed EOS-year distribution so they can sanity-check.
- **Requesters in the room:** Justin Malsam and Ryan Walker. Speak HTM language: end-of-support, vendor service timelines, replacement prioritization, budgeting timelines, vendor engagement, and the shift "from data wrangling to strategic planning."

### Day-1 demo flow (the backbone)

1. **Frame the pain (30 sec):** "HTM plans replacement off a 14,000-row spreadsheet, filtering by hand. Watch this."
2. **The Genie moment (2 min):** ask "What anesthesia machines need replacement in 2026?" live. Then ask a follow-up ("only at Magic Valley," "which are on Windows 7"). Open the generated SQL to prove it is governed and inspectable.
3. **The governance contrast (1 min):** show that "replacement spend" is a certified Metric View, identical for Genie, the dashboard, and Finance. This is the Genie-versus-Copilot / Fabric line.
4. **From question to plan (2 min):** flip to the AI/BI dashboard: replacement pipeline by year, facility × clinical-area risk heatmap, the `ai_forecast` spend curve, and the end-of-life-OS security tile.
5. **The close (30 sec):** "Same governed data, question to forecast to plan, per-facility secured, fully audited, on the platform that will hold the real TMS feed. That is the shift from wrangling to strategy."
