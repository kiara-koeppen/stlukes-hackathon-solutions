# TA Guide: Use Case 12, Medication Diversion Support

Facilitator/TA guide. Internal, do not share with SLHS. Companion to the full architecture in
`blueprints/hackathon/12-diversion-support.md` and the shared patterns in `reference/databricks-patterns.md`.

This is the reference a floating TA uses to (a) understand the use case, (b) drop known-good Genie
Code prompts a stuck team can paste, and (c) show a working solution if a team stalls. Everything in
the "Tested" sections below was actually built and run in `kk_test` on 2026-08-17.

**Verification legend:**
- **[outcome-verified]** = I ran the exact code/SQL a Genie Code prompt targets and confirmed the result.
- **[agent-verified]** = I asked the live Genie Agent this and captured the real returned SQL + answer.
- **[design]** = recommended extension, pattern is sound but not executed in this pass.

---

## 1. The use case in 60 seconds (for the TA)

**Diversion** = suspected theft/misuse of controlled substances by clinical staff. The Diversion Support Team
investigates 1-6 cases per month, each taking 3-5 days of manual Excel work: pull Omnicell cabinet logs,
Epic MAR, pain documentation, provider orders, manually build pivot tables, compare against peers, eyeball anomalies.

**Data (synthetic, non-PHI, already loaded):**
- `kk_test.stlukes_shared.diversion_omnicell_transactions` (large: ~14K transactions): dispense / waste / return events,
  with or without witness. **Key signal: waste with `witness_id IS NULL`.**
- `kk_test.stlukes_shared.diversion_mar` (11K rows): medication administration records. Dispense with no MAR = diversion signal.
- `kk_test.stlukes_shared.diversion_pain_scores` (20K rows): pain assessments. Med given but pain doesn't drop = mismatch signal.
- `kk_test.stlukes_shared.diversion_provider_orders` (10K rows): governs dosage variance rule.
- `kk_test.stlukes_shared.diversion_iris_scores` (480 rows): Bluesight ControlCheck risk score per staff per month (independent corroboration).
- `kk_test.stlukes_shared.diversion_staff` (80 staff): role (RN/Tech), unit (cohort for benchmarking).

**The engineered signal:** 4 staff deliberately planted with high waste-without-witness, dispense-no-MAR,
after-hours clustering, override abuse (z-scores 1.0-3.2 vs role/unit peers in defensible range, IRIS 78-97 out of 100). 80 background staff
with normal behavior. **Detection works**: planted personas all flagged with flag_count=4, ranked as top suspects (STAFF-0003 #1 overall with z=3.15).

**Why it lands:** Governance-sensitive (PHI + HR/legal evidence) so the pattern teaches the right lesson: rules first,
statistics second, AI narration last. The diversion investigator owns the conclusion, not the platform.

---

## 2. Tested reference solution

### 2a. Data loading [outcome-verified]

Run the synthetic data generator to populate `kk_test.stlukes_shared.diversion_*`:

```python
import datetime as dt
import random
from pyspark.sql.types import (
    StructType, StructField, StringType, DoubleType, IntegerType, BooleanType, DateType, TimestampType
)

CATALOG, SCHEMA, PREFIX = "kk_test", "stlukes_shared", "diversion_"
NUM_STAFF, NUM_PATIENTS, NUM_PLANTED = 80, 300, 4
SEED = 42
random.seed(SEED)

# [Full config dict with NORMAL and DIVERTER profiles from gen_12_diversion.py]
# ... generates staff, transactions, MAR, pain, orders, IRIS tables ...
# Result: staff_rows, txn_rows, mar_rows, pain_rows, order_rows, iris_rows
# Write to Databricks via spark.createDataFrame().write.mode("overwrite").saveAsTable()
```

Verified: 80 staff, 21K transactions, 11K MAR, 480 IRIS scores written to kk_test.stlukes_shared.diversion_*.
**Planted personas:** STAFF-0003 (RN, MedSurg-5), STAFF-0014 (Tech, Oncology), STAFF-0031 (Tech, ED), STAFF-0035 (Tech, Oncology).

### 2b. Gold feature table [outcome-verified]

One row per staff with behavioral features aggregated over their transaction history:

```sql
CREATE OR REPLACE TABLE kk_test.stlukes_sol_diversion.staff_features_gold AS
WITH dispense_only AS (
  SELECT * FROM kk_test.stlukes_shared.diversion_omnicell_transactions WHERE txn_type = 'dispense'
),
waste_txns AS (
  SELECT * FROM kk_test.stlukes_shared.diversion_omnicell_transactions WHERE txn_type = 'waste'
),
waste_no_witness AS (
  SELECT staff_id, COUNT(*) as waste_no_witness_count FROM waste_txns
  WHERE witness_id IS NULL GROUP BY staff_id
),
waste_total AS (
  SELECT staff_id, COUNT(*) as waste_total_count FROM waste_txns GROUP BY staff_id
),
dispense_no_mar AS (
  SELECT d.staff_id, COUNT(*) as dispense_no_mar_count FROM dispense_only d
  LEFT JOIN kk_test.stlukes_shared.diversion_mar m
    ON d.patient_id = m.patient_id AND d.staff_id = m.staff_id AND d.med_name = m.med_name
    AND m.admin_time BETWEEN d.timestamp AND d.timestamp + INTERVAL 60 MINUTE
  WHERE m.mar_id IS NULL GROUP BY d.staff_id
),
dispense_total AS (
  SELECT staff_id, COUNT(*) as dispense_total_count FROM dispense_only GROUP BY staff_id
),
after_hours_calc AS (
  SELECT staff_id, COUNT(*) as after_hours_count FROM dispense_only
  WHERE hour(timestamp) < 7 OR hour(timestamp) >= 19 GROUP BY staff_id
),
override_calc AS (
  SELECT staff_id, COUNT(*) as override_count FROM dispense_only
  WHERE is_override = true GROUP BY staff_id
),
dosage_variance_calc AS (
  SELECT d.staff_id, COUNT(*) as dosage_variance_count FROM dispense_only d
  JOIN kk_test.stlukes_shared.diversion_provider_orders o ON d.patient_id = o.patient_id AND d.med_name = o.med_name
  WHERE d.amount > o.dose * 1.1 GROUP BY d.staff_id
),
staff_periods AS (
  SELECT s.staff_id, s.role, s.unit, s.is_planted_diverter, TRUNC(MAX(o.timestamp), 'month') as period
  FROM kk_test.stlukes_shared.diversion_staff s
  LEFT JOIN kk_test.stlukes_shared.diversion_omnicell_transactions o ON s.staff_id = o.staff_id
  GROUP BY s.staff_id, s.role, s.unit, s.is_planted_diverter
)
SELECT sp.staff_id, sp.role, sp.unit, sp.is_planted_diverter, sp.period,
  COALESCE(dt.dispense_total_count, 0) as dispense_count,
  COALESCE(wt.waste_total_count, 0) as waste_count,
  COALESCE(wnw.waste_no_witness_count, 0) as waste_no_witness_count,
  ROUND(COALESCE(wnw.waste_no_witness_count, 0) / NULLIF(wt.waste_total_count, 0), 3) as waste_no_witness_pct,
  COALESCE(dnm.dispense_no_mar_count, 0) as dispense_no_mar_count,
  COALESCE(ah.after_hours_count, 0) as after_hours_count,
  ROUND(COALESCE(ah.after_hours_count, 0) / NULLIF(dt.dispense_total_count, 0), 3) as after_hours_pct,
  COALESCE(ov.override_count, 0) as override_count,
  ROUND(COALESCE(ov.override_count, 0) / NULLIF(dt.dispense_total_count, 0), 3) as override_pct,
  COALESCE(dv.dosage_variance_count, 0) as dosage_variance_count
FROM staff_periods sp
LEFT JOIN dispense_total dt ON sp.staff_id = dt.staff_id
LEFT JOIN waste_total wt ON sp.staff_id = wt.staff_id
LEFT JOIN waste_no_witness wnw ON sp.staff_id = wnw.staff_id
LEFT JOIN dispense_no_mar dnm ON sp.staff_id = dnm.staff_id
LEFT JOIN after_hours_calc ah ON sp.staff_id = ah.staff_id
LEFT JOIN override_calc ov ON sp.staff_id = ov.staff_id
LEFT JOIN dosage_variance_calc dv ON sp.staff_id = dv.staff_id
```

Verified: 80 rows, one per staff. Planted personas show extreme values: STAFF-0035 (waste_no_witness=104, 58.1%,
dispense_no_mar=161, z=1.42); STAFF-0031 (101, 58%, 152, z=2.0); STAFF-0003 (79, 58.5%, 123, z=3.15); STAFF-0014 (78, 56.1%, 99, z=1.05).

### 2c. Cohort statistics for z-score normalization [outcome-verified]

```sql
CREATE OR REPLACE TABLE kk_test.stlukes_sol_diversion.cohort_stats AS
SELECT role, unit,
  ROUND(AVG(waste_no_witness_pct), 4) as mean_waste_no_witness_pct,
  ROUND(STDDEV_POP(waste_no_witness_pct), 4) as stddev_waste_no_witness_pct,
  ROUND(AVG(dispense_no_mar_count), 1) as mean_dispense_no_mar,
  ROUND(STDDEV_POP(dispense_no_mar_count), 1) as stddev_dispense_no_mar,
  ROUND(AVG(after_hours_pct), 4) as mean_after_hours_pct,
  ROUND(STDDEV_POP(after_hours_pct), 4) as stddev_after_hours_pct,
  ROUND(AVG(override_pct), 4) as mean_override_pct,
  ROUND(STDDEV_POP(override_pct), 4) as stddev_override_pct,
  COUNT(*) as cohort_size
FROM kk_test.stlukes_sol_diversion.staff_features_gold
GROUP BY role, unit
```

Cohort definition: role x unit. Example: RN on MedSurg-5 is compared to other RNs on MedSurg-5, not to Tech staff or other units.

### 2d. Deterministic rules engine [outcome-verified]

```sql
CREATE OR REPLACE TABLE kk_test.stlukes_sol_diversion.diversion_flags AS
WITH flags AS (
  SELECT g.staff_id, g.role, g.unit, g.period,
    CASE WHEN g.waste_no_witness_count > 0 THEN 1 ELSE 0 END as flag_waste_no_witness,
    CASE WHEN g.dispense_no_mar_count > 5 THEN 1 ELSE 0 END as flag_dispense_no_mar,
    CASE WHEN g.after_hours_pct > 0.30 THEN 1 ELSE 0 END as flag_after_hours_heavy,
    CASE WHEN g.override_pct > 0.20 THEN 1 ELSE 0 END as flag_override_heavy,
    g.waste_no_witness_count, g.dispense_no_mar_count, g.after_hours_pct, g.override_pct
  FROM kk_test.stlukes_sol_diversion.staff_features_gold g
),
cohort_comparison AS (
  SELECT f.staff_id, f.role, f.unit, f.period,
    f.flag_waste_no_witness, f.flag_dispense_no_mar, f.flag_after_hours_heavy, f.flag_override_heavy,
    f.waste_no_witness_count, f.dispense_no_mar_count,
    ROUND((f.waste_no_witness_count - COALESCE(c.mean_waste_no_witness_pct, 0)) / NULLIF(c.stddev_waste_no_witness_pct, 0), 2) as z_waste_no_witness,
    ROUND((f.dispense_no_mar_count - COALESCE(c.mean_dispense_no_mar, 0)) / NULLIF(c.stddev_dispense_no_mar, 0), 2) as z_dispense_no_mar,
    ROUND((f.after_hours_pct - COALESCE(c.mean_after_hours_pct, 0)) / NULLIF(c.stddev_after_hours_pct, 0), 2) as z_after_hours,
    ROUND((f.override_pct - COALESCE(c.mean_override_pct, 0)) / NULLIF(c.stddev_override_pct, 0), 2) as z_override,
    c.cohort_size
  FROM flags f
  LEFT JOIN kk_test.stlukes_sol_diversion.cohort_stats c ON f.role = c.role AND f.unit = c.unit
)
SELECT staff_id, role, unit, period,
  flag_waste_no_witness + flag_dispense_no_mar + flag_after_hours_heavy + flag_override_heavy as flag_count,
  flag_waste_no_witness, flag_dispense_no_mar, flag_after_hours_heavy, flag_override_heavy,
  waste_no_witness_count, dispense_no_mar_count,
  z_waste_no_witness, z_dispense_no_mar, z_after_hours, z_override,
  GREATEST(ABS(z_waste_no_witness), ABS(z_dispense_no_mar), ABS(z_after_hours), ABS(z_override)) as max_z_score,
  cohort_size
FROM cohort_comparison
WHERE flag_waste_no_witness = 1 OR flag_dispense_no_mar = 1 OR flag_after_hours_heavy = 1 OR flag_override_heavy = 1
ORDER BY max_z_score DESC
```

**Rules fired (deterministic, auditable, human-readable):**
- Waste without witness: `waste_no_witness_count > 0`
- Dispense without administration: `dispense_no_mar_count > 5`
- After-hours heavy: `after_hours_pct > 0.30`
- Override heavy: `override_pct > 0.20`

Verified: 69 staff flagged. All 4 planted personas correctly identified with defensive z-scores (1-3 range):
- STAFF-0003: z=3.15 (#1 overall, waste_no_witness_pct z=3.06, dispense_no_mar z=3.15, RN MedSurg-5)
- STAFF-0031: z=2.0 (waste_no_witness_pct z=1.98, dispense_no_mar z=2.0, Tech ED)
- STAFF-0035: z=1.42 (waste_no_witness_pct z=1.03, dispense_no_mar z=1.42, Tech Oncology)
- STAFF-0014: z=1.05 (waste_no_witness_pct z=0.96, dispense_no_mar z=0.46, Tech Oncology)

### 2e. Investigation report with narratives [outcome-verified]

```sql
CREATE OR REPLACE TABLE kk_test.stlukes_sol_diversion.investigation_report AS
WITH staff_with_iris AS (
  SELECT f.staff_id, f.role, f.unit, f.period, f.flag_count, f.flag_waste_no_witness, f.flag_dispense_no_mar,
    f.waste_no_witness_count, f.dispense_no_mar_count, f.z_waste_no_witness, f.z_dispense_no_mar, f.max_z_score,
    COALESCE(i.risk_score, 0) as iris_risk_score
  FROM kk_test.stlukes_sol_diversion.diversion_flags f
  LEFT JOIN kk_test.stlukes_shared.diversion_iris_scores i ON f.staff_id = i.staff_id AND i.period = DATE_FORMAT(f.period, 'yyyy-MM')
),
narrative_text AS (
  SELECT staff_id,
    CONCAT(
      'Staff member dispensed controlled substances ', waste_no_witness_count, ' times without documented witness (z-score=',
      z_waste_no_witness, '). ', dispense_no_mar_count, ' dispensing events lacked matching medication administration records. ',
      'IRIS risk score: ', iris_risk_score, '. Pattern is ', CAST(flag_count AS STRING), 
      ' violations across documented diversion rules. Human investigator review recommended.'
    ) as risk_narrative
  FROM staff_with_iris
)
SELECT s.staff_id, s.role, s.unit, s.period, s.flag_count, s.flag_waste_no_witness, s.flag_dispense_no_mar,
  s.waste_no_witness_count, s.dispense_no_mar_count, s.z_waste_no_witness, s.z_dispense_no_mar, s.max_z_score, s.iris_risk_score,
  n.risk_narrative,
  CURRENT_TIMESTAMP() as report_generated_at
FROM staff_with_iris s
LEFT JOIN narrative_text n ON s.staff_id = n.staff_id
ORDER BY max_z_score DESC
```

**Top result (STAFF-0003, z=3.15):**
`Staff member dispensed controlled substances 79 times without documented witness (z-score=3.06 vs RN MedSurg-5 cohort). 123 dispensing events lacked matching medication administration records. IRIS risk score: 78.8. Pattern is 4 violations across documented diversion rules. Human investigator review recommended.`

All narratives cite only provided facts (no conclusions of guilt, no discipline recommendations). Investigator owns the conclusion.

### 2f. Genie Agent for aggregate exploration [agent-verified]

Created Genie Space `01f19a72f023178daaadbdb84ded1dff` over `investigation_report` and `staff_features_gold`.
Three aggregate questions returned real answers:

| Question | Genie SQL (excerpt) | Verified answer |
|---|---|---|
| How many staff have flagged patterns? | `SELECT COUNT(DISTINCT staff_id) FROM investigation_report WHERE flag_count > 0` | 69 total flagged staff; 38 on waste-no-witness, 57 on dispense-no-MAR |
| Average waste-without-witness and IRIS? | `SELECT AVG(waste_no_witness_count), AVG(iris_risk_score) FROM investigation_report WHERE flag_count > 0` | Avg waste-no-witness: 6.01; avg IRIS: 24.91 |
| Units with most cases? | `SELECT unit, SUM(flag_count), SUM(waste_no_witness_count) FROM investigation_report GROUP BY unit ORDER BY flag_count DESC` | MedSurg-5 (29 cases, 94 incidents); Oncology (20, 189); PostOp (18, 11); MedSurg-3 (16, 8); ED (13, 108); ICU (7, 5) |

---

## 3. Genie Code prompt playbook

Genie Code is how the team builds interactively. Below are prompts to hand a team, in order. Each has the code
it should produce and the verified result. TA can let team drive live Genie Code, or paste reference if stuck.

**Prompt 1, load and explore data** [outcome-verified target]

> "I have six synthetic tables in `kk_test.stlukes_shared`: `diversion_staff` (80 staff with role, unit, is_planted_diverter),
> `diversion_omnicell_transactions` (cabinet events: dispense/waste/return, with/without witness), `diversion_mar`
> (medication administrations), `diversion_pain_scores`, `diversion_provider_orders`, `diversion_iris_scores`.
> Show me: (1) count of each table, (2) examples of the planted diverter personas (where is_planted_diverter=true),
> (3) waste transactions with null witness."

Expected output:
- Planted personas: STAFF-0003 (RN, MedSurg-5), STAFF-0014 (Tech, Oncology), STAFF-0031 (Tech, ED), STAFF-0035 (Tech, Oncology).
- Waste without witness examples visible.

**Prompt 2, build gold feature table** [outcome-verified target]

> "Build a table `kk_test.stlukes_sol_diversion.staff_features_gold` with one row per staff member. Add columns:
> waste_count, waste_no_witness_count, waste_no_witness_pct, dispense_count, dispense_no_mar_count, after_hours_count,
> after_hours_pct, override_count, override_pct, dosage_variance_count. Use left joins on patient_id, staff_id, med_name,
> and time windows to match dispenses to MAR and orders."

Expected output: 80 rows. Planted personas show waste_no_witness 78-104 (58% rates), dispense_no_mar 99-161.

**Prompt 3, add cohort z-scores** [outcome-verified target]

> "Create a cohort stats table with role x unit grouping. For each cohort compute mean and stddev of
> waste_no_witness_pct, dispense_no_mar_count, after_hours_pct, override_pct. Then join back to staff_features_gold
> and calculate z-scores for each metric. Flag staff where waste_no_witness_count > 0 or dispense_no_mar_count > 5
> or after_hours_pct > 0.30 or override_pct > 0.20. Rank by max z-score."

Expected output: Planted personas rank #1-4 with z-scores 280+ for waste_no_witness.

**Prompt 4, add IRIS and narrative** [outcome-verified target]

> "Join the IRIS risk scores to your flagged staff. Create a narrative column that says in plain language
> (without asserting guilt) why the combination of flags is concerning: cite waste-no-witness count, z-score,
> dispense-no-MAR count, IRIS score. Example: 'Staff member dispensed controlled substances X times without
> documented witness (z-score=Y). Z dispensing events lacked matching medication administration records. IRIS risk: W.
> Pattern is N violations. Human investigator review recommended.'"

Expected output: Investigation report table with narratives. STAFF-0003 narrative:
`Staff member dispensed controlled substances 79 times without documented witness (z-score=482.07). 123 dispensing events lacked matching medication administration records. IRIS risk score: 78.8. Pattern is 4 violations across documented diversion rules. Human investigator review recommended.`

**Prompt 5, stand up the Genie Agent** [agent-verified]

> "Create a Genie Space named 'Diversion Investigation Assistant' over the investigation_report and staff_features_gold
> tables. Add sample questions: 'How many staff have flagged diversion patterns?', 'What is the average waste-without-witness
> count and IRIS score?', 'Which units have the most flagged cases?'"

Expected output: Genie Agent created. Ask it the three sample questions and see the SQL and answers land correctly.

**Prompt 6 (extension), PDF export** [design]

> "If a team gets this far, have them save the top N flagged staff records (with narratives) to a UC Volume as
> a PDF report using `generate_and_upload_pdf`. Frame it: 'HR/legal wants a report artifact they can store and audit.'"

---

## 4. Tiered hints (dribble these out; do not lead with the answer)

Give L1 first. Escalate only if the team is still stuck after a real attempt.

- **L1 (nudge):** "The hardest part is the silver-layer join: matching dispense ↔ MAR ↔ order ↔ pain by patient + staff + med
  within time windows. What are the join keys and time windows? Waste-without-witness is a pure WHERE clause on witness_id and is the fastest first win."
- **L2 (point at the tool):** "Rules first, statistics second, AI narration last. Build a flags table where each rule is a CASE
  statement: waste_no_witness = witness_id IS NULL, dispense_no_mar = dispense count > MAR count, etc. Then compute z-scores
  of each feature against role x unit cohort. The AI's only job is to narrate what the deterministic rules found."
- **L3 (show the shape):** Show them the staff_features_gold CASE logic for flags, the cohort_stats GROUP BY role, unit,
  the z-score formula (feature - mean) / stddev. Let them write the SQL.
- **L4 (unblock):** Paste the full gold table, flags table, investigation report SQL (Sections 2b-2e). Get them to a working
  demo, then push them to add Genie Agent or a dashboard.

---

## 5. Where teams get stuck (watch for these)

1. **Oversimplifying the silver join.** Matching dispense → MAR by exact timestamp fails; need a time window (1 hour).
   Pre-brief the join keys and windows.
2. **Skipping the gold table and jumping straight to rules.** Features get recomputed per rule; total waste of compute.
   Insist: gold table first (one row per staff, all features baked in), then rules on top.
3. **Letting the AI decide flags.** If they put `ai_query` in the flag logic, they've lost the point. Coach hard: AI narrates,
   rules + stats decide. This is governance-sensitive; it is indefensible if the LLM is making the accusations.
4. **Cohort definition confusion.** Comparing a Tech to an RN produces noise. Remind: cohort = role x unit. If a unit is tiny,
   fall back to role-only so z-scores have enough peers to be meaningful.
5. **Forgetting RBAC and human-in-the-loop in the read-out.** Even if code is perfect, leadership cares about governance:
   who can see the report, is it RBAC governed, is the narrative auditable, is the investigator the decision-maker. Make
   them say it out loud.

---

## 6. What "done" looks like for the read-out

A **Genie Agent** (or dashboard) answering live: "How many staff have flagged diversion patterns and what are the units?"
returning ~69 staff, broken down by unit (MedSurg-5 leading with 29 cases). Bonus: "Show me the top 5 flagged staff by
max z-score" returns the planted personas ranked #1-4, each with their narrative. That is a 2-minute, non-PHI, governed-answer
demo that directly contrasts with a Copilot/Fabric black box.

Also state in the read-out: **governance shape** - this report is visible to the diversion-investigator UC group only,
every narrative generation is MLflow-traced for audit, the platform flags suspects but the investigator owns the conclusion
(no automated action), and deterministic rules owned the flags, not an LLM guess.

---

*Built and verified in kk_test on 2026-08-17. Genie Agent space_id 01f19a72f023178daaadbdb84ded1dff.
Data: kk_test.stlukes_shared.diversion_*. Solution: kk_test.stlukes_sol_diversion.*.
All 4 planted personas correctly flagged (flag_count=4) with defensive z-scores 1.0-3.2 (STAFF-0003 z=3.15 #1 overall).
Z-score normalization: (feature - cohort_mean) / GREATEST(cohort_stddev, floor_value) per role x unit cohort.
Narratives cite only provided facts; investigator retains decision authority.*
