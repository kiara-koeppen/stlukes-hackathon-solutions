# 01 CKD Identification & Risk Flagging

**Group:** Group 3 (shares schema with #14 HTM) | **Requester:** Nephrology / primary-care physician (name TBD, confirm with Yutong) | **Problem archetype:** Clinical extraction + rules-based risk flagging (human-in-the-loop CDS) | **Priority:** Medium (clinical, PHI-sensitive)

## 1. Problem & desired outcome

A St. Luke's physician manually reviews large patient populations to find people whose Chronic Kidney Disease (CKD) is missing or mis-staged in Epic. They shared a ~5,000-patient CKD sample and noted that 31 charts took over an hour, so the full population is roughly 500 hours of chart review. The pain is real and quantifiable: staging drives referral, medication dosing, and intervention timing, so a delayed or wrong stage delays care. The target experience: an overnight pipeline reads labs and diagnoses, applies the KDIGO staging rules, and surfaces a ranked worklist of patients who look like CKD by their labs but have no coded diagnosis, each with a *suggested* stage, a confidence score, and the evidence behind it, for a clinician to confirm or dismiss. The clinician stays the decision-maker; the platform does the 500 hours of reading.

## 2. Solution in one picture

```
        Epic (synthetic)                    Databricks (hackathon.shared + group3_ckd_htm)
  ┌──────────────────────────┐
  │ lab_results (eGFR/Cr/UACR)│      Pattern A: Lakeflow Declarative Pipelines
  │ diagnoses (ICD-10 N18.x)  │──┐   ┌──────────── medallion ────────────┐
  │ encounters                │  ├──▶│ bronze (raw) → silver (typed/     │
  │ patients                  │  │   │ conformed, latest labs per patient)│
  │ clinical_notes (nephro)   │──┘   └───────────────┬───────────────────┘
  └──────────────────────────┘                      │
                                                     ▼
                            ┌─────────────────────────────────────────────┐
                            │ KDIGO rules engine (SQL / silver MV)          │
                            │  • 2+ eGFR <60 over ≥90 days  → CKD G-stage   │
                            │  • UACR → A1/A2/A3 albuminuria                 │
                            │  • join dx: has N18.x?  → CARE GAP if not      │
                            └───────────────┬───────────────────────────────┘
                                            │  (labs ambiguous / conflicting)
                                            ▼
                            ┌─────────────────────────────────────────────┐
                            │ Pattern C: ai_query / ai_extract over         │
                            │ clinical_notes → pull stated stage cues       │
                            └───────────────┬───────────────────────────────┘
                                            ▼
                    gold.ckd_candidates  (patient, suggested_stage, confidence,
                    evidence_json, care_gap_flag, risk_tier)
                       │                       │                        │
        Pattern D ─────┘        Pattern I ─────┘           Pattern J ────┘
     Genie Space for          Databricks App              MLflow genai.evaluate:
     clinician exploration    review worklist             suggested vs gold-label
     ("show me G4 gaps")      (confirm / dismiss)         (accuracy, safety)
```

## 3. The chained architecture (step by step)

**Stage 1: Ingest Epic extracts into a medallion (Pattern A).**
What happens: batch/Auto Loader read of the five synthetic source tables (`patients`, `lab_results`, `diagnoses`, `encounters`, `clinical_notes`) into bronze, then silver typing and conforming (cast lab values, normalize LOINC codes, dedupe, resolve one row per lab event). Which feature: **Lakeflow Declarative Pipelines** (streaming tables + materialized views) on serverless. Why this vs. alternatives: it is the exact pattern replacing the Health Catalyst pipelines, it is declarative (teams describe the tables, not the orchestration), and it gives lineage and data-quality expectations for free, which matters for a clinical use case that will later face validation.

**Stage 2: KDIGO staging rules engine in SQL.**
What happens: a silver/gold materialized view computes each patient's CKD status deterministically. The core logic: find patients with **2 or more eGFR values below 60 mL/min/1.73m² separated by at least 90 days** (the KDIGO chronicity criterion), map the most recent/representative eGFR to a G-stage (G1 ≥90, G2 60-89, G3a 45-59, G3b 30-44, G4 15-29, G5 <15), and map the latest UACR to an albuminuria category (A1 <30, A2 30-300, A3 >300 mg/g). Then left-join to `diagnoses` filtered to ICD-10 **N18.x** to compute `care_gap_flag = (lab_evidence_of_ckd AND no N18.x on problem list)`. Which feature: plain **Databricks SQL** inside the Lakeflow pipeline. Why this vs. alternatives: staging is a published, deterministic clinical rule, so it must be SQL, not an LLM. This keeps the clinically load-bearing logic auditable, testable, and defensible. The LLM is reserved only for the genuinely fuzzy step (Stage 3). This split (deterministic rules in SQL, fuzzy step in AI) is the single most important design decision in the blueprint.

**Stage 3: AI enrichment over unstructured notes where labs are ambiguous (Pattern C).**
What happens: for the subset of patients whose structured labs are ambiguous or conflicting (for example, a single low eGFR with no confirmatory second value, or an eGFR that straddles a stage boundary), call `ai_query` / `ai_extract` over `clinical_notes` to pull staging cues that a nephrologist wrote in prose ("CKD stage 3b, likely diabetic nephropathy", "declining renal function, will recheck GFR"). The extracted cue becomes supporting evidence and can raise or lower the suggested stage's confidence. Which feature: **Pattern C, SQL-native AI functions** (`ai_extract` for structured pull, `ai_query` with a scoped prompt for reasoning) running inside the pipeline. Why this vs. alternatives: it runs where the data lives with no endpoint to manage, results land in a Delta column with lineage, and it is the cheapest governable way to add AI to a batch pipeline. Full RAG (Pattern B) is overkill here because we are enriching rows we already have, not answering open-ended questions over a document corpus. Only the fuzzy step touches the model; everything clinically decisive stayed in Stage 2.

**Stage 4: Gold "CKD candidate" table.**
What happens: assemble `gold.ckd_candidates`, one row per flagged patient, carrying `suggested_stage`, `suggested_albuminuria`, `confidence` (derived from how many criteria agree, lab recency, and whether the note corroborates), `evidence_json` (the specific eGFR values, dates, UACR, and any extracted note snippet that drove the suggestion), `care_gap_flag`, and a `risk_tier` (for example, G4/G5 or rapid eGFR decline = high). Which feature: a gold **materialized view / Delta table** in the group's schema. Why: this is the contract every downstream consumer (Genie, the App, the eval) reads from. Evidence travels with the suggestion so the clinician never sees a naked stage label.

**Stage 5: Genie Space for clinician exploration (Pattern D).**
What happens: a **Genie Space** over the gold schema (plus a Metric View for governed counts like "care-gap patients by suggested stage") lets a clinician or population-health analyst ask natural-language questions: "How many patients look like G4 but have no CKD diagnosis?", "Show me the care-gap patients seen in the last 90 days." Which feature: **Pattern D, Agent Bricks Genie Space** with curated instructions and sample SQL, embeddable via Genie Code. Why this vs. alternatives: Genie answers over *governed* UC tables with lineage and row/column security, not a black box, which is the headline differentiator versus a Copilot/Fabric answer of unknown provenance.

**Stage 6: Databricks App review worklist (Pattern I).**
What happens: a **Databricks App** (React/FastAPI) renders the ranked worklist. Each row shows the patient, the suggested stage, the confidence, and the expandable evidence; the clinician clicks **Confirm** (accept the suggestion), **Adjust** (change the stage), or **Dismiss** (not CKD / already handled), and the decision is written back to a review table. Which feature: **Pattern I, Databricks App + Model Serving**, with **OBO (on-behalf-of) auth** so the app respects each clinician's UC permissions (never make the app service principal an admin). Lakebase holds the review/decision state. Why: this is the POC→prod step, it turns a gold table into something a physician actually works through, and it is the human decision point that keeps the system as clinical decision *support*, never autonomous diagnosis.

**Stage 7: Evaluate the suggestions (Pattern J).**
What happens: because the synthetic generator knows each patient's ground-truth stage, we build an eval set of (patient labs+notes → true stage) and run **MLflow `genai.evaluate()`** to score whether the suggested stage matches the gold-standard label, plus safety/guideline scorers (does it ever assert a diagnosis rather than a suggestion? does it flag high-risk patients it should?). Which feature: **Pattern J, MLflow GenAI evaluation** with built-in and custom scorers; in production, clinician Confirm/Adjust/Dismiss actions become the ongoing labeled feedback that aligns the judges. Why: this is the credibility layer and the honest answer to "how do you know it's right?" that any clinical stakeholder will ask before trusting the worklist.

**The chain:** A (ingest) → SQL rules engine → C (AI over notes) → gold candidates → D (Genie) → I (App worklist) → J (eval).

## 4. Data model

All synthetic, in `hackathon.shared.ckd_*` (read-only source) and the group's `group3_ckd_htm` schema (their bronze/silver/gold). Generated by `synthetic_data/generators/gen_01_ckd.py`; full DDL and column dictionary in `synthetic_data/schemas/01_ckd_schema.md`.

| Table | Grain | Key columns |
|---|---|---|
| `ckd_patients` | one row per patient | `patient_id`, `age`, `sex`, `dob`, `race`, `has_diabetes`, `has_hypertension`, `_true_ckd_stage` (hidden ground truth for eval) |
| `ckd_lab_results` | one row per lab result | `patient_id`, `lab_date`, `loinc`, `test_name` (eGFR / creatinine / UACR), `value`, `unit` |
| `ckd_diagnoses` | one row per coded diagnosis | `patient_id`, `icd10_code`, `dx_date`, `description` (N18.x present only for correctly-coded patients) |
| `ckd_encounters` | one row per encounter | `patient_id`, `encounter_id`, `encounter_date`, `encounter_type`, `department` |
| `ckd_clinical_notes` | one row per note | `patient_id`, `note_date`, `note_type`, `note_text` (subset carry explicit staging language for Stage 3) |

Gold output the solution writes: `gold.ckd_candidates` (`patient_id`, `suggested_stage`, `suggested_albuminuria`, `confidence`, `evidence_json`, `care_gap_flag`, `risk_tier`, `latest_egfr`, `latest_egfr_date`).

Governance shape: `_true_ckd_stage` is present only so teams can run Pattern J; in a real deployment there is no ground-truth column, and the clinician's confirmed stage is the label. Column masking / RBAC applies to any column that would be identifying in real data (see §5).

## 5. Governance & safety

This is a clinical-decision-support, PHI-sensitive use case, so the guardrails from `platform-architecture.md` apply in full:

- **Synthetic data only, no PHI.** The entire solution is designed against the synthetic shapes above. No real patient data enters the sandbox. Any real-data phase is gated behind BAA alignment.
- **Human-in-the-loop is mandatory.** The system produces *suggestions for clinician review*, never an autonomous diagnosis or a write-back to the Epic problem list. Stage 6's Confirm/Adjust/Dismiss is the required decision point. Every clinician-facing string is framed as "suggested stage / for review," never "diagnosis."
- **Deterministic core, auditable AI.** Clinically load-bearing staging is SQL rules (Stage 2), so it is testable and defensible. The LLM only enriches ambiguous cases (Stage 3), and its output is evidence, not a verdict.
- **RBAC + masking.** Unity Catalog row/column security and `is_account_group_member` gate who sees the worklist; identifying columns are masked. The App uses OBO auth so each clinician sees only what their UC grants allow.
- **Audit + traceability.** UC system tables (audit) plus MLflow trace logging capture every AI generation and every clinician decision, so the population-health team can answer "who reviewed this and when, and what evidence did the model use?"
- **Evidence with every suggestion.** `evidence_json` means no suggestion is a black box; the clinician always sees the eGFR values, dates, and note snippet behind it. This is both a safety property and the thing that builds trust.

## 6. What a team can realistically build in the hackathon

**The ~1.5-day MVP:**

**Scope IN:**
- Load the synthetic tables (run the generator).
- Build the medallion (Pattern A) to a silver "one row per patient with their latest eGFR/creatinine/UACR and lab history."
- Write the KDIGO SQL rules engine: the 2+ eGFR<60-over-90-days chronicity check, G-stage mapping, and the N18.x care-gap left-join. **Getting the care-gap list is the win.**
- Produce `gold.ckd_candidates` with suggested stage, a simple confidence heuristic, and `evidence_json`.
- One AI touch: `ai_extract`/`ai_query` over `clinical_notes` for the ambiguous subset (Stage 3) to show the SQL-rules + AI-enrichment split.
- Either a Genie Space (Pattern D) **or** a thin Databricks App worklist (Pattern I), whichever the team is faster at, to make it explorable.

**Scope OUT (coach them away from these):**
- A polished, fully styled App with write-back workflow, auth roles, and Lakebase state, unless they are moving fast. A read-only worklist or a Genie Space is enough to tell the story.
- A trained ML staging model. KDIGO is a rule, not a learned model; resist the urge to "ML-ify" it.
- Full MLflow eval harness (Pattern J). Frame it as the "how we'd prove it" slide unless they have time; a single spot-check of suggested vs `_true_ckd_stage` is enough to gesture at it.
- UACR/albuminuria staging can be a stretch goal; the eGFR care gap alone is a complete story.

## 7. Path to production

- **Package as a DAB.** Move the pipeline, gold tables, Genie Space, App, and eval job into a Databricks Asset Bundle for one-command deploy across dev/prod.
- **Real-data cutover.** Swap the synthetic sources for governed Epic Clarity/Caboodle tables behind a BAA; the medallion and rules engine are unchanged because they were designed against the real shapes. Drop the `_true_ckd_stage` column; the clinician's confirmed stage becomes the label.
- **Eval gate.** Pattern J becomes a CI gate: the suggested-vs-confirmed accuracy and the safety scorers must clear a threshold before a pipeline version ships. Nephrology SME feedback aligns the LLM judges.
- **Monitoring.** Schedule the pipeline as a Lakeflow Job (nightly), monitor via UC system tables and MLflow production traces, and track the operational metric that matters: care-gap patients surfaced, reviewed, and resolved per week.
- **The honest read-out story:** the hackathon MVP is the working rules engine + candidate list; production is roughly a few focused weeks to add write-back, the eval gate, RBAC/masking on real tables, and clinician sign-off, well inside a "2 POC→prod" OKR.

## 8. Competitive angle

A Copilot/Fabric stack can summarize a chart you hand it, but it cannot govern a population-scale pipeline that deterministically applies KDIGO across every patient, keeps the AI step auditable and separate from the clinical rule, carries evidence and lineage with every suggestion, and proves its accuracy with an MLflow eval gate. Governed Genie over real UC tables plus MLflow eval is the "how do you know it's right, and can you prove it at scale?" answer that a black-box Copilot response can't match.

## 9. Facilitation notes

- **The "aha" to steer toward:** the care-gap join. When a team runs the left-join and sees the count of patients with 2+ eGFR<60 over 90 days **and no N18.x code**, that number *is* the demo. Tie it straight back to the physician's 500 hours: "the pipeline just did the 500 hours, and here are the ones that need a human." Get them to that count early.
- **Where teams get stuck #1: the chronicity rule.** They will write `eGFR < 60` and call it CKD. Push them to the *two values ≥90 days apart* window; that is the clinically correct definition and the difference between a real flag and a false positive from a single acute dip. A self-join or window function on `ckd_lab_results` ordered by `lab_date` does it.
- **Where teams get stuck #2: over-using the LLM.** Some teams will try to have the model do the staging. Redirect: staging is SQL, the model only reads the notes for the ambiguous cases. If the clinically decisive step is an LLM call, the design is wrong for a CDS use case.
- **Where teams get stuck #3: no evidence.** A worklist that says "G4" with nothing behind it is useless to a physician. Make sure `evidence_json` (the actual values and dates) rides along; that is what makes it clinically credible.
- **Keep saying "suggestion, not diagnosis."** This is a synthetic-data, human-in-the-loop framing at all times. It is also the honest answer to the governance question a St. Luke's clinician will ask.
- **Confirm with Yutong:** the requesting physician's name and specialty, and the exact catalog/schema/table/column names from the real 5,000-patient sample so the synthetic schema mirrors it.
