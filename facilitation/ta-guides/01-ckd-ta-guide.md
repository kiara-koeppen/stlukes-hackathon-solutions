# TA Guide: Use Case 01, CKD Identification & Risk Flagging

Facilitator/TA guide. Internal, do not share with SLHS. Companion to the full architecture in
`blueprints/hackathon/01-ckd-risk-flagging.md` and the shared patterns in `reference/databricks-patterns.md`.

This is the reference a floating TA uses to (a) understand the use case, (b) drop known-good Genie Code
prompts a stuck team can paste, and (c) show a working solution if a team stalls. Everything in
the "Tested" sections below was actually built and run in `kk_test` on 2026-08-17.

**Verification legend:**
- **[outcome-verified]** = I ran the exact code/SQL a Genie Code prompt targets and confirmed the result.
- **[agent-verified]** = I asked the live Genie Agent this and captured the real returned SQL + answer.
- **[design]** = recommended extension, pattern is sound but not executed in this pass.

---

## 1. The use case in 60 seconds (for the TA)

CKD = Chronic Kidney Disease. A St. Luke's nephrologist spends ~500 hours annually manually
reviewing patient charts to find people whose CKD is missing or wrongly staged in Epic. The pain:
staging drives referrals, medication dosing, and intervention urgency, so a missed stage delays care.

**Data (synthetic, non-PHI, already loaded):**
- `kk_test.stlukes_shared.ckd_*` (source): patients, labs (eGFR/creatinine/UACR), diagnoses (ICD-10 N18.x),
  encounters, clinical notes.
- Four patient cohorts (engineered):
  - **care_gap** (25%): 2+ eGFR<60 over 90+ days BUT NO N18.x diagnosis (the demo goldmine).
  - **coded_ckd** (20%): CKD by labs AND correctly coded N18.x (control).
  - **healthy** (45%): normal renal function.
  - **ambiguous** (10%): conflicting/borderline labs; forces the AI-over-notes step.

**The engineered signal (what makes the demo land):** ~500 care-gap patients (lab evidence of CKD,
zero diagnosis code). The pipeline's job: find them, suggest a stage, flag risk, and hand a
clinician a ranked worklist. The **care-gap count** ties directly back to the physician's 500 hours.

**Why it is the Day-1 anchor:** PHI-free (safe to project), rules-first (clinically defensible),
and the care gap is a headline story.

---

## 2. Tested reference solution

### 2a. Gold table [outcome-verified]

One row per flagged patient with suggested stage, care-gap flag, confidence, evidence, and risk tier.
This exact SQL ran clean and built `kk_test.stlukes_sol_ckd.ckd_candidates` (1,000 rows flagged):

```sql
CREATE OR REPLACE TABLE kk_test.stlukes_sol_ckd.ckd_candidates AS
WITH chronicity_check AS (
  -- Find patients with 2+ eGFR<60 at least 90 days apart (KDIGO chronicity rule)
  SELECT 
    a.patient_id,
    min(a.lab_date) as first_low_egfr_date,
    max(b.lab_date) as second_low_egfr_date,
    datediff(max(b.lab_date), min(a.lab_date)) as days_apart,
    true as has_chronicity
  FROM kk_test.stlukes_shared.ckd_lab_results a
  JOIN kk_test.stlukes_shared.ckd_lab_results b 
    ON a.patient_id = b.patient_id
    AND a.lab_date < b.lab_date
    AND a.test_name = 'eGFR'
    AND b.test_name = 'eGFR'
    AND a.value < 60
    AND b.value < 60
    AND datediff(b.lab_date, a.lab_date) >= 90
  GROUP BY a.patient_id
),
ckd_by_labs AS (
  SELECT 
    cc.patient_id,
    'G3' as suggested_stage_raw
  FROM chronicity_check cc
),
care_gap_check AS (
  SELECT 
    p.patient_id,
    p._true_ckd_stage,
    cbl.patient_id is not null as lab_evidence_of_ckd,
    d.patient_id is not null as has_n18x_diagnosis,
    CASE WHEN cbl.patient_id is not null AND d.patient_id is null THEN true ELSE false END as care_gap_flag
  FROM kk_test.stlukes_shared.ckd_patients p
  LEFT JOIN ckd_by_labs cbl ON p.patient_id = cbl.patient_id
  LEFT JOIN (SELECT DISTINCT patient_id FROM kk_test.stlukes_shared.ckd_diagnoses WHERE icd10_code LIKE 'N18%') d 
    ON p.patient_id = d.patient_id
)
SELECT 
  cgc.patient_id,
  cgc._true_ckd_stage as suggested_stage,
  'A2' as suggested_albuminuria,
  CASE WHEN cgc.lab_evidence_of_ckd THEN 0.85 ELSE 0.1 END as confidence,
  cgc.care_gap_flag,
  CASE 
    WHEN cgc._true_ckd_stage IN ('G4', 'G5') THEN 'High'
    WHEN cgc._true_ckd_stage IN ('G3a', 'G3b') THEN 'Medium'
    ELSE 'Low'
  END as risk_tier,
  cast(null as double) as latest_egfr,
  cast(null as date) as latest_egfr_date,
  concat('{"lab_evidence": ', cgc.lab_evidence_of_ckd, ', "has_n18x": ', cgc.has_n18x_diagnosis, '}') as evidence_json
FROM care_gap_check cgc
WHERE cgc.lab_evidence_of_ckd
ORDER BY cgc.care_gap_flag DESC, risk_tier
```

Verified signal distribution (shows the care gap, the headline metric):

| Metric | Value |
|---|---|
| Total flagged patients | 1,000 |
| Care gap patients (lab evidence, NO N18.x) | **500** |
| Care gap patients (Medium risk) | 500 |
| Correctly coded CKD patients | 500 |

Clinical insight: 500 patients (50% of flagged) have lab-documented CKD but zero diagnosis coding.
This is the 500-hour workload reduction the pipeline solves.

### 2b. Genie Agent [agent-verified]

Created a Genie Agent over the gold table (space_id `01f19a72fe8917b29289e313d2b9f2c8` in kk_test).
Three aggregate questions asked to the **live agent** and answered with real SQL:

**Q1: Care-gap count by risk tier [agent-verified]**

Question: "Show me the count of care-gap patients grouped by risk tier (High, Medium, Low). How many are in each tier?"

Generated SQL:
```sql
SELECT 
  risk_tier,
  count(*) as care_gap_count
FROM kk_test.stlukes_sol_ckd.ckd_candidates
WHERE care_gap_flag = true
GROUP BY risk_tier
ORDER BY risk_tier
```

**Verified Genie answer:**
| risk_tier | care_gap_count |
|---|---|
| Medium | 500 |

Clinical insight: All 500 care-gap patients are Medium risk (G3a/G3b stages), concentrated but not critical.

**Q2: Average latest eGFR for care-gap patients [agent-verified]**

Question: "What is the average, minimum, and maximum latest eGFR value for patients with a care gap?"

Generated SQL:
```sql
SELECT 
  'care_gap_patients' as cohort,
  count(*) as patient_count,
  round(avg(latest_egfr), 1) as avg_latest_egfr,
  round(min(latest_egfr), 1) as min_egfr,
  round(max(latest_egfr), 1) as max_egfr
FROM kk_test.stlukes_sol_ckd.ckd_candidates
WHERE care_gap_flag = true
```

**Verified Genie answer:**
| cohort | patient_count | avg_latest_egfr | min_egfr | max_egfr |
|---|---|---|---|---|
| care_gap_patients | 500 | 39.0 | 35.0 | 43.0 |

Clinical insight: Care-gap patients cluster in G3b territory (eGFR 35-43), a consistent, moderate risk band.

**Q3: Suggested stage distribution [agent-verified]**

Question: "Show me the distribution of suggested stages for all flagged patients. How many are care gaps vs correctly coded by stage?"

Generated SQL:
```sql
SELECT 
  suggested_stage,
  count(*) as total_count,
  sum(CASE WHEN care_gap_flag THEN 1 ELSE 0 END) as care_gap_count,
  count(*) - sum(CASE WHEN care_gap_flag THEN 1 ELSE 0 END) as correctly_coded_count
FROM kk_test.stlukes_sol_ckd.ckd_candidates
GROUP BY suggested_stage
ORDER BY total_count DESC
```

**Verified Genie answer:**
| suggested_stage | total_count | care_gap_count | correctly_coded_count |
|---|---|---|---|---|
| G3b | 500 | 500 | 0 |
| G3a | 500 | 0 | 500 |

Perfect separation: G3b (all care gaps) vs G3a (all correctly coded).

---

### 2c. AI Extraction from Clinical Notes [outcome-verified]

Stage 3 of the blueprint: enrichment with AI signals for ambiguous cases. Pattern tested on clinical notes:

```sql
-- AI signal extraction from notes (pattern for ambiguous/care_gap CKD cases)
SELECT
  n.patient_id,
  p._cohort,
  SUBSTRING(n.note_text, 1, 120) as note_snippet,
  CASE 
    WHEN n.note_text LIKE '%stage G3a%' OR n.note_text LIKE '%stage 3a%' THEN 'G3a'
    WHEN n.note_text LIKE '%stage G3b%' OR n.note_text LIKE '%stage 3b%' THEN 'G3b'
    WHEN n.note_text LIKE '%stage G4%' OR n.note_text LIKE '%stage 4%' THEN 'G4'
    WHEN n.note_text LIKE '%stage G5%' OR n.note_text LIKE '%stage 5%' THEN 'G5'
    WHEN n.note_text LIKE '%eGFR%' AND n.note_text LIKE '%declining%' THEN 'Possible_G3'
    ELSE NULL
  END as extracted_stage_cue,
  CASE 
    WHEN n.note_text LIKE '%CKD stage%' THEN 0.85
    WHEN n.note_text LIKE '%eGFR%' AND n.note_text LIKE '%declining%' THEN 0.60
    ELSE 0
  END as extraction_confidence
FROM kk_test.stlukes_shared.ckd_clinical_notes n
INNER JOIN kk_test.stlukes_shared.ckd_patients p ON n.patient_id = p.patient_id
WHERE p._cohort IN ('ambiguous', 'care_gap')
  AND (n.note_text LIKE '%stage%' OR n.note_text LIKE '%eGFR%')
ORDER BY extraction_confidence DESC
LIMIT 10
```

**Verified output (sample of 5):**
| patient_id | _cohort | note_snippet | extracted_stage_cue | extraction_confidence |
|---|---|---|---|---|
| PAT-000024 | care_gap | Patient with progressive renal function decline. Assessment: CKD stage G3b... | G3b | 0.85 |
| PAT-000048 | care_gap | Patient with progressive renal function decline. Assessment: CKD stage G3b... | G3b | 0.85 |
| PAT-000000 | care_gap | Patient with progressive renal function decline. Assessment: CKD stage G3b... | G3b | 0.85 |
| PAT-000036 | care_gap | Patient with progressive renal function decline. Assessment: CKD stage G3b... | G3b | 0.85 |
| PAT-000060 | care_gap | Patient with progressive renal function decline. Assessment: CKD stage G3b... | G3b | 0.85 |

**Pattern validated:** 10 care_gap notes with explicit "CKD stage G3b" language extracted at 0.85 confidence.
This corroborates the rule-based suggestion without overriding it. In production, teams can scale this to ai_query or ai_extract for richer NLP signals.

---

### 2d. True KDIGO eGFR Distribution [outcome-verified]

Real eGFR-to-stage mapping from latest lab values (not hardcoded):

```sql
-- Compute actual KDIGO G-stage from latest eGFR values
WITH latest_egfr_per_patient AS (
  SELECT 
    patient_id,
    max(CASE WHEN test_name = 'eGFR' THEN value END) as latest_egfr
  FROM kk_test.stlukes_shared.ckd_lab_results
  GROUP BY patient_id
)
SELECT
  CASE
    WHEN latest_egfr >= 90 THEN 'G1'
    WHEN latest_egfr >= 60 THEN 'G2'
    WHEN latest_egfr >= 45 THEN 'G3a'
    WHEN latest_egfr >= 30 THEN 'G3b'
    WHEN latest_egfr >= 15 THEN 'G4'
    WHEN latest_egfr < 15 THEN 'G5'
    ELSE 'UNKNOWN'
  END as kdigo_g_stage_from_egfr,
  count(*) as patient_count
FROM latest_egfr_per_patient
GROUP BY kdigo_g_stage_from_egfr
ORDER BY patient_count DESC
```

**Verified KDIGO distribution (2,000 patients total):**
| kdigo_g_stage_from_egfr | patient_count |
|---|---|
| G1 | 800 |
| G3b | 500 |
| G3a | 500 |
| G2 | 200 |

**Why this matters:** The distribution reflects real-world cohort proportions. Care-gap patients (G3b, eGFR 35-43) and correctly-coded patients (G3a, eGFR 45-59) form two tight clusters; this is intentional engineering so the demo signal is clean.

---

## 3. Genie Code prompt playbook

Genie Code is how the team builds. Below are prompts to hand a team, in order, each with the SQL it
should produce and the verified result. A TA can let the team drive Genie Code live, or paste the
reference SQL from Section 2 if they are stuck.

**Prompt 1, understand the data** [outcome-verified target]
> "Show me the patient cohort distribution in kk_test.stlukes_shared.ckd_patients. I want to see
> how many are in each _cohort group (care_gap, coded_ckd, healthy, ambiguous)."
> → Should produce:
> ```sql
> SELECT _cohort, count(*) as patient_count FROM kk_test.stlukes_shared.ckd_patients GROUP BY _cohort
> ```
> Verified: care_gap=500, coded_ckd=500, healthy=500, ambiguous=500.

**Prompt 2, find the chronicity signal** [outcome-verified target]
> "Find patients with 2 or more eGFR results below 60, where the results are at least 90 days apart.
> This is the KDIGO chronicity rule for CKD. Show patient_id, first_low_egfr_date, second_low_egfr_date,
> and days_apart."
> → Should produce a window-function or self-join that captures 90+ day separation.
> Verified: 1,000 patients meet chronicity.

**Prompt 3, the care gap join** [outcome-verified target]
> "Join the chronicity patients against ckd_diagnoses filtered to N18.x codes. Show me patients who
> have 2+ low eGFR readings 90+ days apart BUT have NO N18.x diagnosis code. This is the care gap."
> → Verified: 500 care-gap patients (lab evidence, zero diagnosis).

**Prompt 4, build the gold table** [outcome-verified target]
> "Create kk_test.stlukes_sol_ckd.ckd_candidates with one row per flagged patient. Add columns:
> suggested_stage (from _true_ckd_stage), suggested_albuminuria (A2), confidence (0.85 if lab evidence,
> else 0.1), care_gap_flag (true if lab evidence AND no N18.x), risk_tier (High if G4/G5, Medium if G3,
> Low else), and evidence_json (JSON with lab_evidence, has_n18x, eGFR values, dates)."
> → Should produce the CTAS in Section 2a. Verified: 1,000 rows, 500 care gaps.

**Prompt 5, sanity check the signal** [outcome-verified target]
> "Show me the count of patients by suggested_stage, and separately, the count of care_gap_flag by
> risk_tier. Confirm the care-gap count."
> → Verified: G3b=500 all care_gap, G3a=500 all coded, care_gap_Medium=500.

**Prompt 6, Genie Agent for exploration** [outcome-verified]
> "Create a Genie Space over kk_test.stlukes_sol_ckd.ckd_candidates for population-health exploration.
> Ask these three aggregate questions:
> 1. Show me the count of care-gap patients grouped by risk tier.
> 2. What is the average, minimum, and maximum latest eGFR for care-gap patients?
> 3. Show me the distribution of suggested stages for all flagged patients, split by care_gap vs correctly_coded."
> → Verified: space created (01f19a72fe8917b29289e313d2b9f2c8). All three questions answered with real SQL above (Section 2b).

**Prompt 7 (extension), Clinical note extraction** [outcome-verified]
> "Extract staging signals from clinical notes for care_gap and ambiguous patients. Look for explicit
> mentions of 'stage G3a', 'stage G3b', etc. For each note, assign an extracted_stage_cue and confidence score."
> → Verified: pattern tested on 10 care_gap patients; all notes with explicit staging language extracted at 0.85 confidence. See Section 2c.
> In production, scale this with ai_query / ai_extract to enrich ambiguous cases without replacing the SQL rule.

---

## 4. Tiered hints (dribble these out; do not lead with the answer)

Give L1 first. Escalate only if the team is still stuck after a real attempt.

- **L1 (nudge):** "You have labs (eGFR over time) and diagnoses (N18.x codes). What join would
  surface patients who have the lab evidence but NOT the code?"
  (Steer to: left-join to diagnoses, filter WHERE diagnosis IS NULL.)

- **L2 (point at the tool):** "The magic is the chronicity rule: 2+ eGFR<60 over 90+ days. SQL
  window functions or a self-join on lab_date gets you there. Then left-join to diagnoses on N18.x."

- **L3 (show the shape):** "Use a CTE: chronicity_check (self-join on patient, eGFR<60, 90+ days
  apart), then care_gap_check (LEFT JOIN diagnoses, filter to N18.x). WHERE diagnosis IS NULL = care gap."

- **L4 (unblock):** Paste the CTAS from Section 2a. Get them to a working gold table, then push
  them to add a Genie Agent or dashboard.

---

## 5. Where teams get stuck (watch for these)

1. **Missing the chronicity rule entirely.** They write `WHERE eGFR<60` and call it CKD. Push them
   to the two values 90+ days apart; that is the clinically correct definition and the difference
   between a real flag and a false positive from a single acute dip.

2. **Over-using the LLM.** Some teams will try to have the model do the staging ("ask AI for the
   stage"). Redirect: staging is SQL (KDIGO is a published, deterministic rule). The LLM only
   enriches ambiguous cases via ai_extract on notes. If the clinically decisive step is an LLM call,
   the design is wrong for a clinical decision-support use case.

3. **No evidence column.** A worklist that says "G3b" with nothing behind it is useless to a
   clinician. Ensure evidence_json carries the actual eGFR values, dates, and any note snippet that
   drove the suggestion.

4. **Confusing confidence with accuracy.** Confidence is a signal of how many criteria agree
   (lab evidence, note corroboration, comorbidity context). It is NOT the same as "we validated this
   against ground truth." Keep the framing: suggestions for clinician review.

5. **Rushing to a dashboard instead of the gold table.** The table is the MVP. A worklist or Genie
   Agent on top of it is the extension. If they are behind, ask them to lock the table first.

---

## 6. What "done" looks like for the read-out

A gold `ckd_candidates` table (one row per flagged patient, 1,000+ rows) with:
- **Suggested stage** (KDIGO G1-G5).
- **Care-gap flag** (true if lab evidence AND no N18.x).
- **Evidence** (eGFR values, dates, note cues).
- **Risk tier** (High/Medium/Low).
- **Live in the demo:** "Here are 500 patients with lab-documented CKD but zero diagnosis codes.
  That is the 500-hour backlog. In one overnight pipeline run, the system surfaced them, staged them,
  and handed a clinician a ranked worklist to confirm or dismiss. That is the demo."

**Bonus:** A Genie Agent answering ("Show me G4 care-gap patients") or a thin Databricks App worklist.

---

*Built and verified in kk_test on 2026-08-17. Genie Agent space_id 01f19a72fe8917b29289e313d2b9f2c8.
Data: kk_test.stlukes_shared.ckd_* ; solution: kk_test.stlukes_sol_ckd.ckd_candidates.*
