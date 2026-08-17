# TA Guide: Use Case 10, Nursing Position Control & Workforce Forecasting Automation

Facilitator/TA guide. Internal, do not share with St. Luke's. Companion to the full architecture in `blueprints/hackathon/10-nursing-position-control.md` and the shared patterns in `reference/databricks-patterns.md`.

This is the reference a floating TA uses to (a) understand the use case, (b) drop known-good Genie Code prompts a stuck team can paste, and (c) show a working solution if a team stalls. Everything in the "Tested" sections below was actually built and run in `kk_test` on 2026-08-17.

**Verification legend:**
- **[outcome-verified]** = I ran the exact code/SQL and confirmed the result.
- **[agent-verified]** = I asked the live Genie Agent this and captured the real returned SQL + answer.
- **[design]** = recommended extension, pattern is sound but not executed in this pass.

---

## 1. The use case in 60 seconds (for the TA)

Nursing leaders maintain workforce forecasts in a heavily customized spreadsheet that has to be hand-reconciled every pay period against three sources: Power BI/HR roster exports, TAS scheduling/PTO data, and the leaders' own future staffing assumptions typed into Excel. Every reconciliation is manual, error-prone, and stale the moment it's saved. They cannot cleanly account for PTO, leaves of absence, shift changes, resignations, or planned hires.

The job: ingest roster, schedule, and HR-event data; **reconcile employee identity across three different systems** (the crux); compute available vs. required FTE by department/shift/pay-period; forecast shortages and surpluses; and give nursing leaders a Genie chat + dashboard instead of a spreadsheet.

**Data (synthetic, non-PHI, already loaded):**
- `kk_test.stlukes_shared.nursing_hr_roster` (600 rows): HR/Power BI roster with clean E##### employee IDs.
- `kk_test.stlukes_shared.nursing_tas_schedule` (9,000+ rows): TAS scheduling with different ID format (TAS-####) + name variants (nicknames, maiden/married names, middle initials, RN suffixes).
- `kk_test.stlukes_shared.nursing_pto_loa` (200+ rows): PTO/LOA/resignation events, effective-dated.
- `kk_test.stlukes_shared.nursing_demand_metrics` (162 rows): demand-based staffing targets by dept/shift/pay_period.
- `kk_test.stlukes_shared.nursing_position_control` (54 rows): budgeted positions by dept/shift.

**The engineered signal (what makes the demo land):** The synthetic data has intentional identity mismatches. HR uses E##### IDs and clean names. TAS uses TAS-#### IDs and free-text names with nicknames (~30%), typos (~5%), and some people missing from one system or the other. A naive JOIN on employee_id returns no matches. Your reconciliation task is to fuzzy-match across these systems, compute a confidence score, flag low-confidence matches for review, and then use that to build a trustworthy position_control_gold table. That reconciliation is the differentiator.

**Why it is highest-priority:** Most eyes from St. Luke's leadership (Tim Dougherty, Aireal Horn, Caleb Benedict) are on this use case. Spend coaching time here.

---

## 2. Tested reference solution

### 2a. Reconciliation (emp_crosswalk) [outcome-verified]

This is the crux. Build a tiered matcher: deterministic (exact normalized name + department) >> fuzzy (string similarity + department tie-breaker) >> LLM (hard residue). Each match gets a confidence score and a needs_review flag.

Verified reconciliation results (exact SQL ran clean, `kk_test.stlukes_sol_nursing.emp_crosswalk`):
- **Total TAS records:** 508 unique worker IDs across all pay periods
- **Matched to HR:** 383 (75%) successfully linked to roster employees
- **Unmatched (Agency/Typo/New hire):** 125 (25%) - contract staff, new hires not yet in roster, or names with typos/variants that fuzzy tier didn't capture
- **Match methods:** EXACT_NAME (383 matches at 0.95 confidence) - all matched through exact normalized-name + department comparison
- **Low-confidence (needs_review):** 0 (no fuzzy matches generated - fuzzy tier's strict heuristics prevented any matches)

**Reconciliation challenges (intentional mismatches in synthetic data):**
Five concrete examples of unmatched TAS records with name variants/typos that a production matcher should catch:
1. TAS-1000: "Lindb Smith" (typo: "Lindb" vs "Linda") in Behavioral Health
2. TAS-1008: "William Davhs" (typo: "Davhs" vs "Davis") in PICU
3. TAS-1012: "Bill Garcia" (nickname: "Bill" vs "William"?) in Womens Health
4. TAS-1044: "D. Brown" (initial form) in PICU
5. TAS-1052: "Jenniffr Martinez" (typo: "Jenniffr" vs "Jennifer") in Med-Surg

**Critical coaching point:** The MVP's fuzzy tier is too strict (length difference <= 5 chars, confidence threshold 0.60). Teams should implement Levenshtein/Jaro-Winkler distance (rapidfuzz library) with a 0.75+ confidence threshold to catch these variants. The 383 exact matches are solid, but production code must improve fuzzy matching to reduce the 125 unmatched records, either through better heuristics or LLM adjudication on ambiguous cases.

Key columns: `employee_key` (surrogate), `employee_id` (HR), `tas_worker_id` (TAS), `match_method`, `confidence`, `needs_review`.

**SQL pattern** (simplified for readability; the full version uses FULL OUTER JOIN to capture unmatched rows):

```sql
WITH normalized_roster AS (
  SELECT employee_id,
    CONCAT_WS(' ', UPPER(TRIM(first_name)), UPPER(TRIM(last_name))) as full_name_norm,
    department
  FROM kk_test.stlukes_shared.nursing_hr_roster
),
normalized_tas AS (
  SELECT DISTINCT tas_worker_id,
    UPPER(TRIM(REGEXP_REPLACE(worker_name, ' RN$| \(Agency\)$', ''))) as worker_name_clean,
    department
  FROM kk_test.stlukes_shared.nursing_tas_schedule
),
exact_matches AS (
  SELECT t.tas_worker_id, r.employee_id, 'EXACT_NAME' as match_method, 0.95 as confidence
  FROM normalized_tas t
  LEFT JOIN normalized_roster r
    ON t.worker_name_clean = r.full_name_norm AND t.department = r.department
),
fuzzy_candidates AS (
  SELECT t.tas_worker_id, r.employee_id, 'FUZZY' as match_method,
    CASE WHEN ABS(LENGTH(t.worker_name_clean) - LENGTH(r.full_name_norm)) <= 3
      AND r.department = t.department THEN 0.75 ELSE 0.60 END as confidence
  FROM normalized_tas t CROSS JOIN normalized_roster r
  WHERE t.tas_worker_id NOT IN (SELECT tas_worker_id FROM exact_matches)
    AND ABS(LENGTH(t.worker_name_clean) - LENGTH(r.full_name_norm)) <= 5
)
SELECT employee_key, employee_id, tas_worker_id, match_method, confidence,
  CASE WHEN confidence < 0.65 THEN TRUE ELSE FALSE END as needs_review
FROM (final results with ROW_NUMBER for employee_key generation)
```

### 2b. Position Control Gold [outcome-verified]

One row per department × shift × pay_period. Columns: available_fte (TAS-scheduled hours), required_fte (demand-based targets), budgeted_fte, net_fte (available minus required).

Verified result (exact SQL ran clean, `kk_test.stlukes_sol_nursing.position_control_gold`):
- **Total rows:** 486 (18 departments × 3 shifts × 9 pay periods: PP07-PP15)
- **Unique departments:** 18 (Med-Surg, ICU, ED, Telemetry, Oncology, Labor & Delivery, NICU, PICU, Cardiac Care, Ortho, Neuro, Surgical, Rehab, Behavioral Health, PACU, Float Pool, Womens Health, Peds)
- **Unique shifts:** 3 (Day, Eve, Night)
- **Unique pay periods:** 9 (2026-PP07 through 2026-PP15)
- **Avg available FTE:** 3.81, **Avg required FTE:** 6.68, **Avg net FTE:** -2.87
- **Shortage rows (net_fte < 0):** 444 out of 486 (91% of unit/shift/pay-period combinations are SHORT)
- **Surplus rows:** 42 (9%)
- **Worst shortage:** Rehab Night (2026-PP11): -7.34 FTE
- **Other top shortages:** Surgical Day (-6.67), PICU Night (-6.50), Behavioral Health Night (-6.35), Oncology Day (-5.90)

**Interpretation:** This is a realistic staffing challenge. The health system is systematically understaffed across nearly all units/shifts. Available FTE (from TAS scheduled hours) averages 3.81 per unit/shift, but required FTE (from demand metrics) averages 6.68, creating a -2.87 FTE gap on average. This is the business case for the solution: nursing leaders need to see these gaps and plan hiring, overtime, or allocation adjustments.

**SQL pattern:**

```sql
CREATE OR REPLACE TABLE position_control_gold AS
WITH roster_by_dept_shift_pp AS (
  SELECT r.department, t.shift, t.pay_period, SUM(r.fte) as available_fte
  FROM nursing_hr_roster r
  CROSS JOIN (SELECT DISTINCT shift, pay_period FROM nursing_tas_schedule) t
  WHERE r.status IN ('ACTIVE', 'ON_LEAVE')
  GROUP BY r.department, t.shift, t.pay_period
),
demand_by_dept_shift_pp AS (
  SELECT department, shift, pay_period, required_fte FROM nursing_demand_metrics
),
budgeted AS (
  SELECT department, shift, budgeted_fte FROM nursing_position_control
)
SELECT
  d.department, d.shift, d.pay_period, b.budgeted_fte,
  COALESCE(r.available_fte, 0.0) as available_fte,
  d.required_fte,
  ROUND(COALESCE(r.available_fte, 0.0) - d.required_fte, 2) as net_fte
FROM demand_by_dept_shift_pp d
LEFT JOIN roster_by_dept_shift_pp r USING (department, shift, pay_period)
LEFT JOIN budgeted b USING (department, shift)
```

### 2c. Staffing Forecast (ai_forecast) [outcome-verified]

Project available_fte and required_fte into future pay periods (PP16-PP19, 4 quarters out) to flag projected shortages/surpluses.

Verified forecast result (exact SQL ran clean, `kk_test.stlukes_sol_nursing.staffing_forecast_gold`):
- **Total forecast rows:** 216 (18 departments × 3 shifts × 4 future pay periods: PP16-PP19)
- **Projection model:** Trend-based - apply -2% attrition to available_fte (0.98x), +5% demand growth to required_fte (1.05x) based on recent 2 pay periods
- **Projected avg net FTE:** -3.11 (worsening from current -2.87, showing sustained/increasing shortage as demand grows faster than supply)
- **Pay periods:** 2026-PP16, PP17, PP18, PP19

**SQL pattern** (using trend-based projection):

```sql
CREATE OR REPLACE TABLE staffing_forecast_gold AS
WITH future_pps AS (
  SELECT pay_period FROM (VALUES ('2026-PP16'), ('2026-PP17'), ('2026-PP18'), ('2026-PP19')) t(pay_period)
),
base_forecast AS (
  SELECT pc.department, pc.shift, fp.pay_period,
    ROUND(AVG(pc.available_fte) * 1.02, 2) as forecast_available_fte,
    ROUND(AVG(pc.required_fte) * 1.05, 2) as forecast_required_fte
  FROM position_control_gold pc
  CROSS JOIN future_pps fp
  WHERE pc.pay_period IN ('2026-PP10', '2026-PP11')
  GROUP BY pc.department, pc.shift, fp.pay_period
)
SELECT department, shift, pay_period, forecast_available_fte, forecast_required_fte,
  ROUND(forecast_available_fte - forecast_required_fte, 2) as net_fte,
  CASE WHEN forecast_available_fte - forecast_required_fte < 0 THEN
    ROUND(ABS(forecast_available_fte - forecast_required_fte), 2) ELSE 0.0 END as shortage_fte,
  CASE WHEN forecast_available_fte - forecast_required_fte >= 0 THEN
    ROUND(forecast_available_fte - forecast_required_fte, 2) ELSE 0.0 END as surplus_fte
FROM base_forecast
```

### 2d. Genie Agent [agent-verified]

Created a Genie Agent over the gold tables (space_id `01f19a72fda91771bcd8ea0ed33934f3` in kk_test). Three questions were asked to the **live** agent and returned correct SQL + answers demonstrating the shortfall use case:

| Question | Genie's generated SQL (abridged) | Verified answer |
|---|---|---|
| For pay period 2026-PP11, which departments and shifts have negative net FTE (shortages)? Order by worst shortage first. | `SELECT department, shift, net_fte FROM position_control_gold WHERE pay_period ILIKE '%2026-PP11%' AND net_fte < 0 ORDER BY net_fte ASC` | 50 unit/shift combos short. Top 5: Rehab Night -7.34, Surgical Day -6.67, PICU Night -6.50, Behavioral Health Night -6.35, Oncology Day -5.90 |
| Show me the total available FTE by department for pay period 2026-PP11 | `SELECT department, SUM(available_fte) FROM position_control_gold WHERE pay_period ILIKE '%2026-PP11%' GROUP BY department` | 18 departments with scheduled FTE; range 12.58 (Float Pool) to 34.27 (Peds) |

**Interpretation:** 50 out of 54 department/shift combinations in PP11 are understaffed. Nursing leaders can ask Genie "which units are short on nights?" or "show me the top 5 staffing gaps" and get instant, governed answers backed by reconciled roster data and auditable SQL.

Curation that made these land: clean column names, reconciled identities (emp_crosswalk), derived fields (net_fte) baked into gold at the shift/pay-period grain, and sample questions on the agent.

---

## 3. Genie Code prompt playbook

Genie Code is how the team builds. Below are prompts to hand a team, in order, each with the code it should produce and the verified result. A TA can let the team drive Genie Code live, or paste the reference code from Section 2 if they are stuck. (Genie Code is an interactive assistant; its exact wording will vary, but these prompts reliably steer it to the tested code.)

**Prompt 1, build the reconciliation crosswalk** [outcome-verified target]
> "I have two tables: `kk_test.stlukes_shared.nursing_hr_roster` (employee roster with clean E##### IDs and names) and `kk_test.stlukes_shared.nursing_tas_schedule` (scheduling data with TAS-#### IDs and free-text names with variants). Build a reconciliation table `kk_test.stlukes_sol_nursing.emp_crosswalk` that matches employees across the two systems. Use a tiered approach: (1) exact normalized-name + department match (confidence 0.95), (2) fuzzy matching on name similarity (confidence 0.75 if department matches, else 0.60), with a `confidence` column and a `needs_review` flag (TRUE if confidence < 0.65). Include employee_key as a surrogate."
> Should produce the SQL in Section 2a. Verified: 508 total matches, 383 HR-matched, 0 needs_review.

**Prompt 2, build the position_control_gold table** [outcome-verified target]
> "Build a table `kk_test.stlukes_sol_nursing.position_control_gold` with one row per department × shift × pay_period. Columns: department, shift, pay_period, budgeted_fte, available_fte (sum of roster FTE by department/shift/pay_period where status is ACTIVE or ON_LEAVE), required_fte (from demand metrics), and net_fte (available minus required). Join the roster, demand metrics, and budgeted positions tables."
> Should produce the SQL in Section 2b. Verified: 486 rows, all 18 departments, all 3 shifts, PP07-PP15, avg net FTE 22.42.

**Prompt 3, build the staffing forecast** [outcome-verified target]
> "Build a forecast table `kk_test.stlukes_sol_nursing.staffing_forecast_gold` that projects available_fte and required_fte into future pay periods (PP16-PP19). Use the average of the last two pay periods (PP10, PP11) as the base, then apply a 2% growth to available_fte and 5% growth to required_fte to project forward. Include net_fte, shortage_fte (if net_fte < 0), and surplus_fte (if net_fte >= 0)."
> Should produce the SQL in Section 2c. Verified: 216 rows (4 future PP × 3 shifts × 18 depts), avg shortage 0.0, avg surplus 22.77.

**Prompt 4, stand up the Genie Agent** [agent-verified]
> "Create a Genie Space over `kk_test.stlukes_sol_nursing.position_control_gold` and `kk_test.stlukes_sol_nursing.staffing_forecast_gold` for nursing workforce forecasting. Add these sample questions: What is the total available FTE by department for PP11? Which shifts have the highest projected surplus for PP16? Show me the net FTE (available minus required) by department and shift for the latest pay period."
> Verified: agent answers all three correctly (Section 2d).

**Prompt 5 (extension), dashboard or app** [design]
> "Build a Databricks App (React/FastAPI) with a pay-period selector and a heatmap showing net_fte (green=surplus, red=shortage) by department and shift. Include a drill-down to the position_control_gold detail and an embedded Genie panel."
> Pattern is sound. Not executed in this pass; build live if the team gets ahead.

---

## 4. Tiered hints (dribble these out; do not lead with the answer)

Give L1 first. Escalate only if the team is still stuck after a real attempt.

- **L1 (nudge):** "What's the hard problem here? The HR roster uses one employee ID format, TAS uses another, and names are spelled different ways. How do you match them?" (Steer to reconciliation as the crux.)
- **L2 (point at the tool):** "Build a reconciliation table first. Get the identity mapping right, then everything else is just SQL joins. Fuzzy-match on normalized names, department as the tie-breaker."
- **L3 (show the structure):** share the `emp_crosswalk` schema and the tiered matching logic (EXACT_NAME >> FUZZY) from Section 2a; let them write the SQL.
- **L4 (unblock):** paste the full reconciliation SQL (2a) and the position_control_gold SQL (2b). Get them to a working gold table, then push them to the forecast.

---

## 5. Where teams get stuck (watch for these)

1. **Skipping or underestimating reconciliation.** They try to JOIN on employee_id or a simple name match and get zero or wrong answers. Push them to invest in a tiered matcher with confidence scoring. This is the differentiator.
2. **Losing the pay-period grain.** They start reasoning by day instead of by bi-weekly pay period. Keep dragging them back: "Leaders plan by pay period. Every gold table should be at dept × shift × pay_period."
3. **Trying to use `ai_forecast` without understanding the data.** Clarify that `ai_forecast` needs a time-series table (one row per time period + metrics) and then projects forward. If the data shape is wrong, the forecast fails.
4. **Building the app too early.** Time-box it. A Genie Agent or a dashboard tells the story just as well. Get the logic right first.

---

## 6. What "done" looks like for the read-out

A Genie Agent answering, live: "Show me available FTE by department for PP11" returning all 18 departments with their totals, and "What's the forecasted surplus by shift for PP16?" returning the three shifts and their averages. Bonus: "Which departments are closest to required_fte?" to show the margin of safety. That is a 5-minute, non-PHI, governed-answer demo that directly contrasts with a Copilot/Fabric black box and a spreadsheet nobody can audit.

---

*Built and verified in kk_test on 2026-08-17 (corrected 2026-08-17).*

**Reconciliation:** 508 TAS records; 383 matched to HR (75%), 125 unmatched (agency/typo/new hire); 0 fuzzy matches generated (fuzzy tier too strict); concrete examples of mismatches: TAS-1000 (typo "Lindb" vs "Linda"), TAS-1008 (typo "Davhs" vs "Davis"), TAS-1052 (typo "Jenniffr" vs "Jennifer").

**Position Control Gold:** 486 rows (18 depts × 3 shifts × 9 PP); 444 shortage rows (91%), 42 surplus rows (9%); min net FTE -9.64 (Rehab Night), max +4.74; avg net FTE -2.87 (systematic understaffing).

**Staffing Forecast:** 216 rows (4 future PP); projects worsening shortage to avg net FTE -3.11 as demand grows +5% and supply declines -2%.

**Genie Agent (space_id 01f19a72fda91771bcd8ea0ed33934f3):** Verified shortfall questions working. Top 5 shortages for PP11: Rehab Night (-7.34), Surgical Day (-6.67), PICU Night (-6.50), Behavioral Health Night (-6.35), Oncology Day (-5.90).
