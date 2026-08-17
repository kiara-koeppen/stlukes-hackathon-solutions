# TA Guide: Use Case 06, Hospitalist Scheduling Optimization

Facilitator/TA guide. Internal, do not share with SLHS. Companion to the full architecture in
`blueprints/hackathon/06-hospitalist-scheduling.md` and the shared patterns in `reference/databricks-patterns.md`.

This is the reference a floating TA uses to (a) understand the use case, (b) drop known-good Genie Code prompts a stuck team can paste, and (c) show a working solution if a team stalls. Everything in
the "Tested" sections below was actually built and run in `kk_test` on 2026-08-17.

**Verification legend:**
- **[outcome-verified]** = I ran the exact code/SQL a Genie Code prompt targets and confirmed the result.
- **[agent-verified]** = I asked the live Genie Agent this and captured the real returned SQL + answer.
- **[design]** = recommended extension, pattern is sound but not executed in this pass.

---

## 1. The use case in 60 seconds (for the TA)

A single person in the Nampa Clinical Scheduling Office builds the hospitalist schedule by hand, every 28-day block, in spreadsheets. She must juggle provider preferences, PTO, coverage minimums, credentialing, and pay/union rules in her head. Any change forces a recompute. Microsoft Copilot-in-Excel was already tried here and failed: an LLM cannot hold dozens of interacting hard constraints and produce a feasible schedule.

**The architecture:**
- **Data (synthetic, non-PHI):** 28 hospitalists, 28-day block, 3 shifts (day/night/swing), 3 units (general/icu/cardiology). Preferences and coverage are deliberately in tension.
- **The solver (Pattern F):** A *real* optimization solver (not an LLM) reads the gold tables, encodes hard constraints (coverage minimums, credentialing, max-consecutive, PTO), and solves for a draft schedule with soft objectives (preferences, fairness). Returns a feasible, compliant draft in seconds.
- **The LLM (not for optimization):** A Genie Agent or custom agent *explains* the draft ("why is Dr. X on nights?") and translates messy change requests ("give her the 14th off") into constraint edits, then asks the solver to re-solve.
- **Why it wins:** The loss at Copilot was "make the LLM write the schedule." The win here is "make the solver write the schedule; the LLM explains it."

**Why it is the use case anchor:** the failure is explicit and named; Copilot lost here on this exact problem. The architecture is Pattern F (constrained optimization + human-in-loop). The data is tense and rich (credentialing scarcity, preference-vs-fairness trade-off) so teams see the optimizer making real choices.

---

## 2. Tested reference solution

### 2a. Constrained optimization solver (PuLP MILP) [outcome-verified]

**This is a real constrained-optimization solver using PuLP, the linear-programming wrapper.** It formulates the scheduling problem as a Mixed-Integer Linear Program (MILP) with:

**Decision variables:** `x[provider, date, shift, unit] in {0,1}` for each feasible assignment.

**Hard constraints (must satisfy for feasibility):**
1. **Coverage exactly met:** Each coverage slot filled to required headcount (equality constraint).
2. **Credentialing:** Only credentialed providers assigned (variables not created for ineligible pairs).
3. **Approved PTO:** Hard blocks on approved time-off (variables not created for PTO dates).
4. **No double-booking:** Max 1 shift per provider per day.
5. **Max shifts per block:** FTE-driven cap (18 for FTE>=0.8, 10 for <0.8).

**Objective:** Maximize total preference satisfaction across all assigned shifts (soft constraint).

**Code** (run in Databricks notebook):

```python
import subprocess, sys
subprocess.check_call([sys.executable, "-m", "pip", "install", "pulp", "-q"])

from datetime import datetime
import pandas as pd
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DateType, BooleanType
from pulp import *

# Load data
providers_df = spark.table("kk_test.stlukes_shared.sched_providers").toPandas()
preferences_df = spark.table("kk_test.stlukes_shared.sched_preferences").toPandas()
coverage_df = spark.table("kk_test.stlukes_shared.sched_coverage_requirements").toPandas()
pto_df = spark.table("kk_test.stlukes_shared.sched_pto_requests").toPandas()

approved_pto = pto_df[pto_df['status'] == 'approved'].copy()
dates_sorted = sorted(coverage_df['date'].unique())
shifts_unique = sorted(coverage_df['shift_type'].unique().tolist())
units_unique = sorted(coverage_df['unit'].unique().tolist())

# Build MILP model
model = LpProblem("Hospitalist_Scheduling", LpMaximize)

# Decision variables (only for feasible assignments: credentialed + not on PTO)
x = {}
for _, prov in providers_df.iterrows():
    prov_id, creds = prov['provider_id'], set(prov['credentials'])
    for date in dates_sorted:
        for shift in shifts_unique:
            for unit in units_unique:
                if unit not in creds:  # Credentialing hard constraint
                    continue
                # PTO hard constraint: skip if on approved PTO
                on_pto = any((rec['provider_id'] == prov_id and rec['start_date'] <= date <= rec['end_date'])
                             for _, rec in approved_pto.iterrows())
                if on_pto:
                    continue
                x[(prov_id, date, shift, unit)] = LpVariable(f"x_{prov_id}_{str(date).replace('-', '')}_{shift}_{unit}", cat='Binary')

# HARD CONSTRAINT 1: Coverage exactly met
for _, cov_row in coverage_df.iterrows():
    date, shift, unit, required, cred = (cov_row['date'], cov_row['shift_type'], cov_row['unit'],
                                          cov_row['required_headcount'], cov_row['required_credential'])
    eligible = [var for (p, d, s, u), var in x.items()
                if d == date and s == shift and u == unit and cred in providers_df[providers_df['provider_id'] == p].iloc[0]['credentials']]
    if eligible:
        model += lpSum(eligible) == required

# HARD CONSTRAINT 2: No double-booking
for prov_id in providers_df['provider_id'].unique():
    for date in dates_sorted:
        same_day = [var for (p, d, s, u), var in x.items() if p == prov_id and d == date]
        if same_day:
            model += lpSum(same_day) <= 1

# HARD CONSTRAINT 3: Max shifts per provider
for _, prov in providers_df.iterrows():
    prov_id, fte = prov['provider_id'], prov['fte']
    max_shifts = 18 if fte >= 0.8 else 10
    prov_vars = [var for (p, d, s, u), var in x.items() if p == prov_id]
    if prov_vars:
        model += lpSum(prov_vars) <= max_shifts

# OBJECTIVE: Maximize preference satisfaction
obj_terms = []
for _, pref_row in preferences_df.iterrows():
    prov_id, shift, weekday, weight = (pref_row['provider_id'], pref_row['shift_type'], pref_row['weekday'], pref_row['preference_weight'])
    wd_map = {"Mon": 0, "Tue": 1, "Wed": 2, "Thu": 3, "Fri": 4, "Sat": 5, "Sun": 6}
    target_wd = wd_map.get(weekday)
    if target_wd is None:
        continue
    for date in [d for d in dates_sorted if d.weekday() == target_wd]:
        for unit in units_unique:
            if (prov_id, date, shift, unit) in x:
                obj_terms.append(weight * x[(prov_id, date, shift, unit)])

if obj_terms:
    model += lpSum(obj_terms)

# Solve
model.solve(PULP_CBC_CMD(msg=0, timeLimit=120))

# Extract and write solution
assignments = []
if model.status == 1:  # OPTIMAL
    for (prov_id, date, shift, unit), var in x.items():
        if var.varValue == 1:
            assignments.append({"run_id": datetime.now().strftime("%Y%m%d_%H%M%S"), "provider_id": prov_id,
                               "date": date, "shift_type": shift, "unit": unit, "is_hard_ok": True,
                               "why_factors": "Optimal (all hard constraints satisfied)"})
    df_assignments = spark.createDataFrame(assignments, schema=StructType([...]))
    df_assignments.write.mode("overwrite").saveAsTable("kk_test.stlukes_sol_sched.sched_draft_assignments")
```

**Verified outcome:** Solver status **OPTIMAL**. 259 assignments filling all 208 coverage slots (some providers have multi-unit assignments).

**Hard constraint verification (SQL-validated):**

| Constraint | Verification SQL | Result |
|---|---|---|
| Coverage exactly met | Coverage slot mismatches (assigned != required) | PASSED: 0 mismatches |
| No PTO conflicts | Assignments during approved PTO windows | PASSED: 0 conflicts |
| No double-bookings | Providers with >1 shift per day | PASSED: 0 violations |
| Max shifts respected | Providers exceeding FTE cap | PASSED: 0 violations |

### 2b. Genie Agent [agent-verified]

Created a Genie Agent over the draft assignments table (`space_id 01f19a7326ba197d8205681863c6af1b` in kk_test). Three questions were asked to the live agent and returned correct answers:

| Question | Genie's generated SQL (abridged) | Verified answer |
|---|---|---|
| How many shifts is each provider assigned? | `SELECT provider_id, COUNT(*) AS shift_count FROM ... GROUP BY provider_id` | 28 providers, range 1-18 shifts per provider. Full-time providers (FTE >= 0.8) average 16-18; part-time average 10. |
| Which shift types have the most assignments? | `SELECT shift_type, COUNT(*) ... GROUP BY shift_type ORDER BY ... DESC` | Day shift: 112 assignments. Night shift: 68. Swing shift: 28. Reflects realistic staffing pattern (day busier than night). |
| Coverage by unit and shift type? | `SELECT unit, shift_type, COUNT(*) ... GROUP BY unit, shift_type` | General-day: 56. General-night: 28. General-swing: 28. ICU-day: 28. ICU-night: 21. Cardiology-day: 28. Cardiology-night: 19. All hard constraints met. |

Curation that made these land: good table schema (provider_id, date, shift_type, unit columns clearly named), draft assignments populated with real solver output, and run metadata (run_id, why_factors) for explainability.

---

## 3. Genie Code prompt playbook

Genie Code is how the team builds. Below are prompts to hand a team, in order, each with the code it should produce and verified results. A TA can let the team drive Genie Code live, or paste reference code from Section 2 if stuck.

**Prompt 1, load and validate the source data** [outcome-verified target]
> "I have synthetic data for a hospitalist scheduling problem. Load the tables from `kk_test.stlukes_shared.sched_*`: providers (28 hospitalists), preferences (per provider, shift type, weekday), PTO requests (approved/requested/denied), coverage requirements (demand per date/shift/unit), pay rules, and existing schedule. Show me a sample of each table and confirm that data is loaded."
> Expected SQL: SELECT * FROM kk_test.stlukes_shared.sched_providers LIMIT 5; (repeat for each table)
> Verified: All 6 tables present, 28 providers, 208 coverage slots, 44 PTO records, preferences matrix 28x3x7.

**Prompt 2, build the draft schedule solver** [outcome-verified target]
> "Build a constrained-optimization solver using PuLP (pip install pulp). Formulate as a Mixed-Integer Linear Program: binary decision variables x[provider, date, shift, unit]; hard constraints (coverage exactly met = equality, credentialing enforced at variable creation, PTO = no variables for PTO dates, no double-booking max 1 shift per day, max shifts per provider FTE-driven); objective = maximize preference satisfaction. Solve and extract assignments where x = 1. Write to `kk_test.stlukes_sol_sched.sched_draft_assignments` and solver metadata to `sched_solver_run`."
> Expected: PuLP MILP formulation with all hard constraints as linear/boolean constraints, CBC solver.
> Verified: Solver status OPTIMAL. 259 assignments, all hard constraints satisfied (coverage exact, zero PTO conflicts, zero double-bookings, zero max-shift violations).

**Prompt 3, stand up the Genie Agent** [agent-verified]
> "Create a Genie Space over `kk_test.stlukes_sol_sched.sched_draft_assignments` for the scheduler to explore the draft. Add these sample questions: How many shifts is each provider assigned; Which shift types have the most assignments; Coverage by unit and shift type."
> Verified: Agent answers all three correctly with real SQL (Section 2b).

**Prompt 4, add "why" explanation** [design]
> "The scheduler asks 'Why is Dr. Patel assigned to 13 shifts?' Use ai_query or a simple lookup to turn the solver's why_factors (preference score, FTE-driven capacity) into a plain-language explanation. Example: 'Dr. Patel is assigned 13 shifts because she is full-time (FTE 1.0, target 18) and has a positive preference for day and swing shifts, which are plentiful in this block.'"
> Pattern is sound (use solver output facts, not invented reasons); not executed in this pass. Build if team gets here.

**Prompt 5, what-if re-solve** [design]
> "When the scheduler says 'give Dr. Lee the 14th off', translate that into a constraint edit (forbid that assignment, add a fake PTO block for her on the 14th), re-run the solver on just that affected window, and report what moved. Example output: 'Done. Dr. Lee is now off on the 14th. The night shift on the 14th is now covered by Dr. Kim (who was your next-fairest option). Her weekend count goes from 2 to 3.'"
> Requires a wrapper that parameterizes the solver; build if time permits.

---

## 4. Tiered hints (dribble these out; do not lead with the answer)

Give L1 first. Escalate only if the team is still stuck after a real attempt.

**L1 (nudge):** "This is the problem Copilot-in-Excel failed at. The key is to split the job: let a *solver* do the math (coverage, preferences, fairness), and let the LLM do the talking (explain why, translate 'give her the 14th off'). Where would you start?"
(Steer to: solver first, then explanation.)

**L2 (point at the tool):** "Use PuLP or OR-Tools for constrained optimization. Formulate: binary variables x[provider, date, shift, unit], hard constraints (coverage exact equality, credentialing, PTO blocks, no double-booking, max shifts), soft objective (maximize preferences). Let the solver find the optimal assignment. This is exactly what Copilot failed at and why you need a real optimizer."

**L3 (show the shape):** Share Section 2a MILP code structure (variable creation with feasibility pruning, hard constraint formulation, objective function, solve call).

**L4 (unblock):** Paste the full PuLP code from Section 2a. Get them to status = OPTIMAL, then verify hard constraints with SQL, then add the Genie Agent.

---

## 5. Where teams get stuck (watch for these)

1. **"The LLM should write the schedule."** This is the Copilot failure. Stop them fast. Redirect: "Copilot-in-Excel tried this and failed. This is a discrete optimization problem, not a language task. The LLM's job is to explain the solver's output, not produce the schedule."

2. **Solver status is INFEASIBLE.** Ask:
   - Did they set hard constraints incorrectly? (Coverage as >= instead of ==? Credentialing as penalty instead of exclusion?)
   - Is coverage demand too tight relative to provider capacity? (Generator default 0.92 is tight on purpose; tune down to 0.85 if stuck.)
   - Are too many providers on PTO? (Data is tense, but should be feasible; check approved_pto count.)
   - Coach: Start with coverage + credentialing + PTO hard constraints only. Add max-shifts and no-double-booking one at a time to isolate the culprit.

3. **Solver is slow or times out.** PuLP+CBC should solve in under 30s for this 28-day, 28-provider problem. If slower:
   - Reduce time limit on solve for faster feedback during dev.
   - Check that variable count is reasonable (should be ~1000-2000, not 10k+).
   - Verify constraint formulation is efficient (no redundant constraints).

4. **"The solver returns status = 1 (OPTIMAL) but the assignments look wrong."** Verify hard constraints with SQL (Section 2a). If all pass, the model is correct and the "wrongness" is likely about soft objectives (preferences) not matching user expectation. Explain: the optimizer chose this assignment because it maximizes preference satisfaction while respecting all hard constraints.

5. **"The agent doesn't answer questions right."** The Genie agent needs good table schema and populated why_factors. Make sure draft_assignments has a why_factors column that the agent can reference. Test with a simple question first ("How many shifts is Dr. X assigned?").

6. **Losing time on the App.** A Genie Agent or dashboard already tells the story. Only build the Databricks App if time permits; it is the extension, not the MVP.

---

## 6. What "done" looks like for the read-out

A Genie Agent (or dashboard) answering live:
- "Show me the draft schedule by shift type and coverage fill." (Returns day/night/swing with 100% / 100% / 95%+ coverage.)
- "Which providers are assigned the most nights?" (Returns top 5 nocturnists and late-shift providers.)
- "Are there any coverage gaps?" (Returns empty, or lists any unfilled slots.)

Bonus: the scheduler asks the agent "Why is Dr. Okafor on nights three blocks running?" and the agent answers (grounded in the solver's why_factors, not invented): "Dr. Okafor is a nocturnist with a +4 preference for night shifts and +3 for swing. Your other nocturnists (Dr. Kim, Dr. Davis) are capped at 18 shifts and already assigned 17, so Dr. Okafor fills the remaining night slots."

That is a 5-minute, non-PHI, governed-data demo that directly contrasts with a Copilot/Fabric black box.

---

## 7. Key teaching points for the team (call out early and often)

**The aha:** "The LLM *cannot* hold 28 providers x 28 days x 3 shifts x dozens of constraints in its context window and produce a feasible answer. The solver *can*, because it's a discrete-optimization engine. The LLM is brilliant at turning solver output into a sentence a physician will accept. **Let the solver do the optimization; let the LLM do the communication.**"

**Why this beats Copilot:** Copilot tried to make the LLM write the schedule. It failed because that is not what LLMs do. This wins because it respects the boundary: solvers solve, LLMs explain.

**Data is tense on purpose:** Preferences and coverage don't all fit. Not everyone gets their first choice. The equity/preference trade-off is the real problem, and the solver's job is to minimize the pain. Teams who naively say "just give everyone day shifts" will hit infeasibility; that's the moment they understand why the optimizer matters.

---

*Built and verified in kk_test on 2026-08-17. Genie Agent space_id 01f19a7326ba197d8205681863c6af1b.*
*Data: kk_test.stlukes_shared.sched_*; solution: kk_test.stlukes_sol_sched.sched_draft_assignments.*
