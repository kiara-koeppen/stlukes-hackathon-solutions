# 12 Diversion Support Reporting Automation

**Group:** Group 1 (Diversion) | **Requester:** Ashley Schoonover, Diversion Support Team | **Problem archetype:** Pattern G - anomaly detection + narrative report generation | **Priority:** High (strong production candidate - DNA team already lands this data in Databricks)

## 1. Problem & desired outcome

Ashley's Diversion Support Team runs 1–6 controlled-substance diversion investigations a month, and each one costs **3–5 business days of manual analytical work**: pulling Omnicell dispense logs, Epic MAR and pain-score documentation, and provider orders into Excel, then hand-building pivot tables to compare one nurse or tech against their peers and eyeball the anomalies. The data already lives in Databricks (the DNA team built a pharmacy report), so the analyst is doing by hand what the platform could do in minutes. The target experience: an investigator selects a staff member and a date window, and the platform automatically benchmarks that person's dispensing behavior against their role/unit cohort, applies the team's documented diversion rules (waste-without-witness, dispense-without-administration, pain-score/med mismatch, timing anomalies), and produces a **structured, human-readable investigation report** with a plain-language risk narrative - turning a 3–5 day pivot exercise into a reviewable draft the investigator validates and owns.

## 2. Solution in one picture

```
SOURCE SYSTEMS                DATABRICKS (hackathon catalog, Unity Catalog governed)                DELIVERED EXPERIENCE
                    ┌─────────────────────────────────────────────────────────────────┐
Omnicell  ─────────┤ BRONZE  raw dispense/waste/return, MAR, pain, orders, IRIS         │
Epic MAR  ─────────┤   │  (Pattern A: Lakeflow Declarative Pipelines, serverless)       │
Epic pain ─────────┤   ▼                                                                │
Epic orders ───────┤ SILVER  typed, joined: dispense↔MAR↔order↔pain linked by patient/  │
Bluesight IRIS ────┤   │      staff/time; conformed staff dimension w/ role+unit cohort │
                    │   ▼                                                                │
                    │ GOLD (feature store)  per-staff-per-period features:              │
                    │   │  waste ratio, waste-without-witness %, dispense-no-MAR count, │
                    │   │  pain-not-documented %, after-hours %, override %, timing gaps │──┐
                    │   ▼                                                                │  │
                    │ RULES ENGINE (SQL / deterministic)  documented diversion flags    │  │  ┌───────────────────────┐
                    │   ▼                                                                │  ├─▶│ Databricks App (RBAC)  │
                    │ COHORT BENCHMARK  z-score vs role/unit peers + IRIS blend          │  │  │ investigator selects   │
                    │   ▼                                                                │  │  │ staff + window →       │
                    │ ai_query  → plain-language RISK NARRATIVE (Pattern G)             │──┘  │ sees report + narrative│
                    │   ▼                                                                │     │ + generated PDF        │
                    │ STRUCTURED INVESTIGATION REPORT (replaces Excel pivots)           │─────▶└───────────────────────┘
                    │   ▼                                                                │
                    │ MLflow trace log + UC audit (Pattern J)  every narrative logged    │
                    └─────────────────────────────────────────────────────────────────┘
                         Access: only the diversion-investigator UC group. PHI masked. Synthetic in the hackathon.
```

## 3. The chained architecture (step by step)

**Stage 1 - Ingest the five source feeds (Pattern A: Medallion ingest).**
Batch-read the synthetic Omnicell transactions, Epic MAR, Epic pain scores, Epic provider orders, and Bluesight IRIS scores into bronze, then conform to silver with **Lakeflow Declarative Pipelines** on serverless. Silver does the join work that the analyst does by hand in Excel: link each Omnicell dispense to its matching MAR administration (by patient + staff + med + time window), to the governing provider order, and to the nearest pain-score documentation. A conformed `staff` dimension carries the `role` (RN/tech) and `unit` needed for cohorting. *Why Pattern A / Lakeflow vs. a one-off notebook:* this is the exact bronze→silver→gold shape that replaces the Health Catalyst pipelines, it is declarative and re-runnable for each investigation window, and the lineage it produces is itself an audit artifact.

**Stage 2 - Feature engineering (gold feature table).**
Aggregate silver to a **per-staff, per-period grain** and compute the behavioral features the rules and benchmarks both consume: waste ratio (waste amount / total handled), waste-without-witness percentage, dispense-without-matching-MAR count, pain-not-documented-after-administration percentage, after-hours dispense percentage, cabinet-override percentage, and median dispense-to-administration timing gap. This is a **gold feature table** - the reusable substrate that makes the rules deterministic and the cohort math cheap. *Why:* separating features from logic means a facilitator (or the customer later) can add a new rule without re-deriving data.

**Stage 3 - Deterministic rules engine FIRST (Pattern G, rules half).**
Before any statistics or AI, apply the team's **documented diversion rules** as plain SQL `CASE` logic over the gold features, each emitting a named boolean flag and the evidence rows that triggered it:
- **Waste without witness** - controlled-substance waste with a null `witness_id`.
- **Dispense without administration** - an Omnicell dispense with no corresponding MAR record within the expected window.
- **Pain-score / medication mismatch** - an analgesic administered where documented pain score does not drop (or no pain score documented at all).
- **Timing anomaly** - dispenses clustered outside the patient's order schedule or heavily after-hours.
- **Dosage variance** - dispensed amount exceeds the ordered dose.
*Why deterministic first:* these rules are auditable, explainable, and legally defensible - an investigator can point to the exact rule and the exact rows. AI never *decides* a flag; it only narrates flags the rules already fired. This ordering is the governance backbone of the whole solution.

**Stage 4 - Cohort benchmarking (Pattern G, statistical half).**
For each staff member, compute **z-scores of every feature against their role/unit cohort** (an RN on a med-surg floor is compared to other med-surg RNs, not to an ED tech). Blend in the **Bluesight IRIS risk score** as an independent corroborating signal. The output is a ranked, quantified picture: "waste-without-witness rate is 4.1σ above the med-surg RN cohort mean, IRIS risk score in the 98th percentile." *Why statistics over ML here:* with 1–6 cases a month there is no labeled training set, and a supervised anomaly model would be unexplainable to a legal reviewer. Z-scores against a peer cohort are transparent, defensible, and exactly the comparison the analyst does manually today. (ML clustering/isolation-forest is a documented *scope-out* stretch, not the MVP.)

**Stage 5 - Plain-language risk narrative (Pattern G, `ai_query` / Pattern C).**
Pass the fired rules, the cohort z-scores, and the supporting evidence rows into **`ai_query`** (Foundation Model API) with a tightly-scoped prompt: *explain, in plain language, why this combination of flags is suspicious, citing only the provided facts; do not invent data; do not state a conclusion of guilt.* The model produces the "why this pattern is concerning" paragraph that the investigator would otherwise write from scratch. *Why `ai_query` vs. a standalone agent:* the narrative is a single deterministic-input → text step inside the pipeline, so the SQL-native AI function is the cheapest and most governable option - it runs where the data lives, lands in a Delta column, and every call is traceable. No endpoint to manage.

**Stage 6 - Assemble the structured investigation report.**
Compose the flags, cohort comparison table, evidence detail, IRIS score, and the generated narrative into a **structured, human-readable report** - the artifact that replaces the manual Excel pivot workbook. Persist it as a governed gold table (one row per investigation) so reports are reproducible and comparable over time.

**Stage 7 - Serve it with RBAC (Pattern I) + generated PDF.**
Front the report with a **Databricks App** (React/FastAPI) using **OBO auth** so it respects each user's UC permissions - only members of the diversion-investigator group can open it. The investigator picks a staff member and window, reviews the auto-assembled report, and exports a court-/HR-ready **PDF** (the `generate_and_upload_pdf` capability writes the artifact to a governed UC Volume). *Why an app:* this is the POC→prod step - it turns the notebook into something Ashley's team actually uses, and the PDF is the deliverable HR/legal expects. Genie (Pattern D) is an optional companion for ad-hoc "show me this unit's waste ratios" exploration over the same governed gold tables.

**Stage 8 - Evaluate, log, and audit (Pattern J).**
Every narrative generation is logged with **MLflow tracing** (inputs, prompt, output, who ran it, when). Build a small **MLflow `genai.evaluate()`** set with synthetic cases and expected behavior - scored for groundedness (narrative cites only provided facts), safety, and guideline adherence (no conclusion of guilt). Combined with **UC system-table audit logs**, this answers the two questions HR/legal and compliance will always ask: *how do you know the narrative is accurate, and who looked at what?*

## 4. Data model

Synthetic tables generated by `synthetic_data/generators/gen_12_diversion.py`, schema documented in `synthetic_data/schemas/12_diversion_schema.md`. All in `hackathon.shared.diversion_*` (read-only source); groups write features/reports to their own schema.

| Table | Grain | Key columns | Notes |
|---|---|---|---|
| `diversion_staff` | one row / staff | `staff_id`, `role` (RN/tech), `unit`, `hire_date` | The cohort dimension - cohort = role × unit |
| `diversion_omnicell_transactions` | one row / cabinet event | `txn_id`, `staff_id`, `patient_id`, `med_name`, `txn_type` (dispense/waste/return), `amount`, `timestamp`, `witness_id` (nullable) | Null `witness_id` on waste is the core red flag |
| `diversion_mar` | one row / administration | `patient_id`, `staff_id`, `med_name`, `admin_amount`, `admin_time`, `order_id` | Dispense with no MAR match = diversion signal |
| `diversion_pain_scores` | one row / pain assessment | `patient_id`, `score` (0–10), `documented_time` | Pain not dropping / not documented after admin |
| `diversion_provider_orders` | one row / order | `order_id`, `patient_id`, `med_name`, `dose`, `ordering_provider` | Governs dosage-variance and no-order dispensing |
| `diversion_iris_scores` | one row / staff / period | `staff_id`, `period`, `risk_score` | Bluesight ControlCheck; independent corroborating signal |

**Governance shape of the data:** `patient_id` and `staff_id` are synthetic surrogate keys - no names, no MRNs. In the real-data phase these columns are exactly where UC **column masking** and **row filters** apply: PHI columns masked for anyone outside the investigator group, and rows themselves restricted so an investigator only sees staff within their scope. The generator seeds a **small number of staff with deliberately planted diversion patterns** against a background of normal behavior, so the rules and z-scores have real signal to find (see generator docstring for the planted-persona list).

## 5. Governance & safety

This is the most governance-sensitive use case in the hackathon: it combines **PHI** (patient meds, pain scores) with **HR/legal evidence about employees** who may face investigation, discipline, or referral. The blueprint treats governance as a first-class feature, not an afterthought.

- **Synthetic-only in the hackathon; BAA before real data.** No PHI and no real employee data ever enter the sandbox. The solution is designed against synthetic shapes precisely so it can be validated without risk. Any cutover to real Epic/Omnicell/Bluesight data is gated on a BAA-aligned review and the production access model below - this is stated explicitly to leadership as a prerequisite, not a nice-to-have.
- **Role-based access via Unity Catalog - only investigators see it.** The report tables, the app, and the underlying gold features are granted to a dedicated **diversion-investigator UC group** and no one else. Access is enforced with `is_account_group_member()` in row filters and column masks, so even a user who somehow reached the tables sees nothing outside their authorization. The Databricks App uses **OBO (on-behalf-of) auth** - the app service principal is *never* an admin; every query runs as the signed-in investigator and inherits their UC grants. This is the direct answer to "how is this different from a shared Excel file on a network drive."
- **PHI minimization and masking.** Patient identifiers are surrogate keys; the report shows only the clinical facts needed to substantiate a flag (that an analgesic was dispensed and how pain was documented), not a full chart. Column masking hides any residual PHI from non-clinical reviewers, and `ai_mask` is available to scrub free-text before it reaches a narrative.
- **Deterministic rules own the flags; AI only narrates.** No flag, score, or conclusion is produced by the LLM. The documented rules (Stage 3) and the cohort z-scores (Stage 4) are fully deterministic and auditable - an investigator can trace every flag to a rule and to the exact evidence rows. The `ai_query` narrative is constrained to *explain the provided facts* and is explicitly prompted **not to assert guilt or recommend discipline**. This keeps the analytically-sensitive judgment where it legally belongs.
- **Human-in-the-loop: the investigator owns the conclusion.** The platform produces a *draft investigation report for human review*, never an automated accusation or action. Ashley's investigator validates the evidence, applies professional judgment, and is the sole decision-maker. Framed for leadership: this is a triage-and-assembly assistant that compresses 3–5 days of pivot-building into a reviewable draft - it does not replace the investigator.
- **Full audit trail (Pattern J).** Every narrative generation is logged to **MLflow** (inputs, prompt, model, output, user, timestamp) and every table access is captured in **UC system audit tables**. Together these give compliance a complete, queryable record of who investigated whom, when, and on what evidence - a stronger chain of custody than the current Excel workflow can offer.
- **Evaluation as a credibility gate.** Before the narrative feature is trusted, an **MLflow `genai.evaluate()`** run scores it for groundedness (cites only provided facts), safety, and guideline adherence (no guilt conclusions). LLM-judges are aligned to SME (investigator) feedback. This is the honest answer to "how do you know the AI text is accurate."

## 6. What a team can realistically build in the hackathon

**Scope IN (the ~1.5-day MVP):**
1. Run the generator; land Omnicell + MAR + pain + orders + IRIS in bronze/silver via a Lakeflow pipeline (Pattern A).
2. Build the per-staff-per-period **gold feature table** (Stage 2).
3. Implement **2–3 of the documented rules** in SQL - waste-without-witness and dispense-without-MAR are the highest-signal and easiest wins (Stage 3).
4. Compute **cohort z-scores** for those features against role/unit peers and blend in IRIS (Stage 4).
5. Generate the **plain-language risk narrative** with `ai_query` for one flagged staff member (Stage 5).
6. Assemble a **structured report** (a notebook-rendered report or a simple table is fine for the MVP) for the top-flagged staff member, and demonstrate it correctly surfaces a *planted* diversion persona (Stage 6).

**Scope OUT (don't drown here):** a polished Databricks App UI (a notebook or a minimal Streamlit/App shell is enough - full RBAC app is a path-to-prod item); the full PDF export styling; ML-based anomaly detection (clustering/isolation forest); implementing all five rules; the complete MLflow eval harness. Steer teams to prove the **chain works end-to-end on one staff member** rather than polishing any single stage.

## 7. Path to production

This is a strong production candidate because the data already lands in Databricks (DNA team's pharmacy report). The honest N-days-to-prod story for the read-out:
- **Package as a DAB** (databricks.yml + the pipeline, jobs, app, and eval as bundle resources) for one-command redeploy across dev→prod.
- **Real-data cutover** gated on BAA-aligned review: point the bronze layer at the governed Epic/Omnicell/Bluesight tables the DNA team already built, apply UC row filters + column masks, and grant only the investigator group.
- **Harden RBAC + the App**: production OBO app, investigator-group grants, PDF export to a governed UC Volume with retention aligned to HR/legal record policy.
- **Eval gate + monitoring**: the MLflow eval set must pass groundedness/safety thresholds before the narrative feature is enabled; add production trace monitoring and periodic LLM-judge re-alignment to investigator feedback.
- **SME rule validation**: co-develop the exact rule thresholds with Ashley's team so the deterministic layer matches their documented policy precisely.

## 8. Competitive angle

Copilot Studio over Fabric can summarize a spreadsheet, but it cannot **govern who is allowed to see HR-legal evidence at the row and column level, prove which deterministic rule fired on which evidence row, and produce an auditable MLflow trace of every AI generation** - all on the same platform that already holds the source data. The whole solution runs on **governed Unity Catalog tables with lineage, row/column security, OBO access, and system-table audit**, and keeps the AI strictly downstream of auditable rules. That combination - defensible governance plus explainable analytics plus a full chain of custody - is exactly what a diversion investigation demands and exactly what a Copilot/Fabric stack cannot assemble end-to-end.

## 9. Facilitation notes

- **The aha moment to steer toward:** run the finished chain on a *planted* diversion persona and watch the z-scores + narrative correctly single them out of a normal-behavior background. That "the platform found the one bad actor and explained why" moment is the demo. Make sure teams know which staff_ids are planted (the generator prints them) so they can validate.
- **Order discipline is the whole lesson.** Teams will be tempted to jump straight to `ai_query`. Coach relentlessly: **rules first, statistics second, AI narration last.** If they let the LLM decide flags, the solution is legally indefensible and they've missed the point of Pattern G. The AI's only job is to explain what deterministic logic already found.
- **Where they'll get stuck:** the silver-layer join (dispense ↔ MAR ↔ order ↔ pain) is the fiddly part - matching on patient + staff + med within a time window. Pre-brief the join keys and the time-window logic. Waste-without-witness is a pure `WHERE witness_id IS NULL` and is the fastest first win - point struggling teams there to build momentum.
- **Cohort definition matters.** Remind teams the cohort is role × unit - comparing a tech to an RN produces noise, not signal. Small units may need role-only cohorts to have enough peers for a meaningful z-score.
- **Governance is the story for this room.** SLHS leadership cares deeply about PHI + HR-legal handling. Even in the MVP, have teams state the RBAC/audit/human-in-the-loop model out loud in their read-out - it's the difference between a cool notebook and a production candidate.
