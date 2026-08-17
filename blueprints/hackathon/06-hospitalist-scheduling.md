# 06 Hospitalist Scheduling Optimization

**Group:** Group 2 (scheduling) | **Requester:** Nampa Clinical Scheduling Office | **Problem archetype:** Pattern F, constrained optimization + human-in-the-loop | **Priority:** High (single-point-of-failure, active pain)

## 1. Problem & desired outcome

Today one person in the Nampa Clinical Scheduling Office builds the hospitalist schedule by hand in spreadsheets. Every block she has to juggle provider preferences, PTO, coverage minimums, credentialing, and pay/union compliance in her head, and a single change (a call-out, a swapped weekend) forces her to recompute the whole thing manually. It is slow, stressful, and it lives in one person's head. If she is out, scheduling stops. Microsoft Copilot-in-Excel was tried on exactly this and failed: an LLM cannot hold dozens of interacting hard constraints and produce a feasible schedule.

The target experience: the scheduler acts as a **reviewer and communicator**, not a calculator. She sets the rules and preferences once, the system generates a compliant **draft** schedule automatically, and when something changes she asks for a re-solve instead of redoing arithmetic. A plain-language assistant explains *why* the schedule looks the way it does ("why is Dr. Okafor on nights again?") so she can defend it to physicians without reverse-engineering the math.

## 2. Solution in one picture

```
  SOURCE SYSTEMS                DATABRICKS (Unity Catalog: hackathon.*)                DELIVERED EXPERIENCE
  ─────────────                 ──────────────────────────────────────                ────────────────────

  Lightning Bolt*  ┐            ┌──────────────────────────────────────┐
  (schedules,      │  Pattern A │ bronze → silver → gold                │
   shift history)  ├──────────► │  sched_providers   sched_preferences  │
                   │            │  sched_pto_requests                    │
  Preferences /    │            │  sched_coverage_requirements           │
  PTO / rules      │            │  sched_pay_rules   sched_existing_...  │
  (spreadsheets)   ┘            └───────────────┬──────────────────────┘
                                                │ gold feature tables
                                                ▼
                                ┌──────────────────────────────────────┐
                                │  CONSTRAINT MODEL + SOLVER (Pattern F)│
                                │  OR-Tools CP-SAT in a Lakeflow Job    │
                                │  hard: coverage, credentialing,       │      ┌───────────────────────────┐
                                │        max-consecutive, rest, pay     │─────►│  Databricks App (Pattern I)│
                                │  soft: preferences, night/weekend     │      │  scheduler reviews the     │
                                │        equity, PTO honored            │      │  DRAFT, edits, approves,   │
                                └───────────────┬──────────────────────┘      │  publishes, communicates   │
                                                │ draft assignments +          │                            │
                                                │ per-provider "why" facts     │  ┌──────────────────────┐  │
                                                ▼                              │  │ LLM agent (Model      │  │
                                ┌──────────────────────────────────────┐      │  │ Serving / custom agent│  │
                                │  EXPLANATION + WHAT-IF AGENT          │◄────►│  │ "why nights?" /       │  │
                                │  narrates changes, answers "why",     │      │  │ "give Dr. X the 14th  │  │
                                │  triggers a scoped re-solve on request│      │  │ off" → re-solve       │  │
                                └───────────────┬──────────────────────┘      │  └──────────────────────┘  │
                                                │                              └───────────────────────────┘
                                                ▼
                                ┌──────────────────────────────────────┐
                                │  Pattern J: MLflow eval of schedule   │
                                │  quality: feasibility, equity, pref-  │
                                │  satisfaction, explanation faithfulness│
                                └──────────────────────────────────────┘

  * Lightning Bolt = the believed third-party scheduling tool (TO CONFIRM with the Nampa office).
```

## 3. The chained architecture (step by step)

### Stage 1: Ingest schedule data, preferences, PTO, and rules into the medallion (Pattern A)

**What happens:** Lightning Bolt* schedule exports and historical assignments already land in Databricks; provider preferences, PTO requests, coverage requirements, and pay/union rules arrive as spreadsheet extracts. Auto Loader / batch reads land them in **bronze**, a Lakeflow Declarative Pipeline conforms and types them in **silver**, and **gold** produces the clean feature tables the solver consumes: one row per provider, per preference, per PTO window, per coverage slot, per rule.

**Which feature:** Pattern A (Lakeflow Declarative Pipelines on serverless, Delta, Unity Catalog).

**Why this vs. alternatives:** the whole reason Copilot-in-Excel failed is that the data lived in fragile, un-joined spreadsheets. Landing it in governed Delta tables with lineage is the prerequisite for everything downstream. The solver needs a single conformed shape, not five tabs a human reconciles by hand. This is the same medallion pattern that replaces the Health Catalyst pipelines, reused.

### Stage 2: Encode the constraint model

**What happens:** the gold tables are translated into a mathematical model. **Hard constraints** (must hold, or the schedule is invalid):
- **Coverage minimums:** every date/shift/unit slot is filled to its required headcount (e.g., 2 day + 1 night per unit).
- **Credentialing / service-line eligibility:** a provider may only be assigned to shifts/units they are credentialed for.
- **Max consecutive shifts and minimum rest:** e.g., no more than 6 (later 7) consecutive shifts, ≥ 10 h rest between shifts, no day-after-night.
- **Pay / union rules:** FTE-driven shift targets, night-differential eligibility, block caps. (Exact union/pay constraints are **TO CONFIRM** with the office. Model them as parameterized rules so they are easy to adjust.)
- **PTO as a hard block:** approved PTO days cannot be assigned.

**Soft constraints** (penalize violations, don't forbid them) form the objective function to minimize:
- **Preference satisfaction:** honor day/night/swing and weekday preferences by their weights.
- **Fairness / equity:** spread nights and weekends evenly across providers (minimize the max-minus-min per provider); nobody gets stuck with all the undesirable shifts.
- **Requested (not-yet-approved) PTO:** honor where feasible.
- **Continuity:** respect "wants consecutive" so providers get grouped blocks rather than scattered singletons.

**Which feature:** modeling layer for Pattern F, authored in a Databricks notebook.

**Why this framing:** separating hard from soft is the core design move. Hard constraints guarantee a *legal, safe* schedule (coverage and credentialing are patient-safety issues, not preferences); soft constraints + weights let the scheduler tune "how much do we value fairness vs. preference" without touching code. This is exactly the structure an LLM cannot represent. It has no notion of a feasible region.

### Stage 3: Solve for a draft schedule (Pattern F core)

**What happens:** an **OR-Tools CP-SAT** model runs in a **Lakeflow Job** (serverless). Decision variables are binary `assign[provider, date, shift]`; hard constraints are posted as linear/boolean constraints; the weighted soft-constraint penalties become the objective. The solver returns an optimal (or best-found within a time budget) set of assignments, which is written back to a `sched_draft_assignments` Delta table along with a `sched_solver_run` row capturing the objective value, per-constraint slack, and solve status.

**Which feature / why the solver choice:** **OR-Tools CP-SAT over PuLP.** This problem is a rostering / assignment problem dominated by *logical* constraints such as "at most N consecutive," "no night-then-day," "exactly K per slot," reified booleans, and fairness expressed as min/max spreads. CP-SAT is a constraint-programming / SAT-based solver built for exactly these combinatorial-logical shapes, handles them natively and fast, and ships a good free solver in the box. PuLP is a modeling wrapper for LP/MILP solvers. It works, and it is a fine fallback if a team is more comfortable with pure linear formulations, but you end up hand-encoding the logical constraints as big-M linear inequalities, which is slower to write and slower to solve. Recommend CP-SAT; allow PuLP as the "if your team already knows it" alternative. Both run in a plain Python notebook on Databricks, no special infrastructure.

**Why a solver at all (the load-bearing point):** the solver does the math the LLM *cannot*. Generating a feasible, optimal assignment over hundreds of binary variables under dozens of interacting constraints is a discrete-optimization problem with a provable answer, not a language task. This is the precise thing Copilot-in-Excel could not do.

### Stage 4: Explain and handle "what-if" changes (LLM agent)

**What happens:** an LLM agent sits on top of the solved schedule. It does two jobs:
1. **Narration / "why" answers.** When the scheduler asks "why is Dr. Okafor on nights three blocks running?", the agent reads the structured facts the solver emitted (this provider's night-preference weight, the equity balance, who else was credentialed and available) and answers in plain language with the actual drivers, grounded in solver output, never invented.
2. **What-if re-solve.** When the scheduler says "give Dr. Patel the 14th off" or "Dr. Lee called out tonight," the agent translates the request into a constraint change (pin/forbid a variable, add a PTO block), triggers a **scoped re-solve** of just the affected window, and reports back what moved and what it cost ("done, the 14th is now covered by Dr. Reyes, who was your next-fairest option; her weekend count goes from 2 to 3").

**Which feature:** custom agent (ResponsesAgent/ChatAgent) on **Model Serving**, or a **composable agent** setup where a Genie Agent over the gold + draft tables answers the data-lookup "why" questions and a custom tool wraps the solver re-solve. The agent calls the solver as a **tool**; it does not do the optimization itself.

**Why this division of labor:** this is the heart of Pattern F. **The optimizer does the math; the agent does the communication; the human decides.** The LLM is superb at turning solver facts into a sentence a physician will accept and at parsing a messy natural-language change request into a formal constraint edit, and it is categorically unfit to *produce* the schedule. Keeping those roles separate is what makes the system both trustworthy and useful, and it is the direct answer to why Copilot failed: Copilot tried to make the language model do the optimization.

### Stage 5: Serve it: the scheduler's review app (Pattern I)

**What happens:** a **Databricks App** (React/FastAPI) is the scheduler's cockpit. It renders the draft as an editable calendar/grid, surfaces coverage gaps and any soft-constraint violations, lets her drag-and-drop or pin assignments (which re-triggers a scoped re-solve), exposes the chat agent for "why" and what-if, and, on approval, publishes the final schedule and generates the communication (per-provider views, a diff vs. the prior block). App-side state (draft edits, approval status, chat history) lives in **Lakebase**.

**Which feature:** Pattern I: Databricks App + Model Serving, **OBO (on-behalf-of) auth** so the app respects each user's Unity Catalog permissions.

**Why:** this is the POC→prod step. It is what turns "a notebook that solves a schedule" into the thing that actually removes the manual spreadsheet from the Nampa office and makes the scheduler a reviewer. Without the app there is no product.

### Stage 6: Evaluate schedule quality (Pattern J, optional but recommended)

**What happens:** an MLflow evaluation harness scores each generated schedule on objective, computable metrics: feasibility (are all hard constraints satisfied?), coverage fill rate, preference-satisfaction rate, night/weekend equity (Gini or max-min spread), and PTO-honor rate, plus **explanation faithfulness** for the agent (does the "why" answer match the actual solver drivers?), scored with an LLM judge aligned to the scheduler's feedback.

**Which feature:** Pattern J: MLflow `genai.evaluate()` with custom scorers.

**Why:** it is the honest answer to "how do we know the draft is good, and that the assistant isn't making up reasons?" It also lets the team compare weightings and solver settings objectively rather than by eyeballing.

## 4. Data model

All synthetic source tables live in `hackathon.shared.sched_*` (read-only to all groups). Generated by `synthetic_data/generators/gen_06_scheduling.py`; full DDL and column dictionary in `synthetic_data/schemas/06_scheduling_schema.md`.

| Table | Grain | Key columns |
|---|---|---|
| `sched_providers` | one row per hospitalist | `provider_id`, `full_name`, `fte`, `credentials` (array), `seniority_years`, `service_line`, `max_consecutive_shifts`, `wants_night_differential` |
| `sched_preferences` | one row per provider × shift_type × weekday | `provider_id`, `shift_type` (day/night/swing), `weekday`, `preference_weight` (−5..+5), `wants_consecutive`, `max_shifts_per_block` |
| `sched_pto_requests` | one row per provider × PTO window | `provider_id`, `start_date`, `end_date`, `status` (approved/requested/denied), `reason` |
| `sched_coverage_requirements` | one row per date × shift_type × unit | `date`, `shift_type`, `unit`, `required_headcount`, `required_credential` |
| `sched_pay_rules` | one row per rule | `rule_id`, `rule_type`, `description`, `param_name`, `param_value`, `applies_to` |
| `sched_existing_schedule` | one row per historical assignment | `assignment_id`, `provider_id`, `date`, `shift_type`, `unit`, `source_system` |

**Tables the solution *writes* (in the group's own schema, e.g. `hackathon.group2_scheduling.*`):** `sched_draft_assignments` (the solver output: provider × date × shift + a `why_factors` struct), `sched_solver_run` (objective value, status, per-constraint slack, timestamp), and any Lakebase-backed app state.

**Governance shape:** this data is provider-scheduling, not clinical PHI, so the RBAC surface is lighter than the clinical use cases, but it is still HR-sensitive (PTO reasons, pay eligibility). PTO `reason` should be column-masked from non-scheduler roles via `is_account_group_member`, and the draft/approval tables carry an audit trail (who edited, who approved).

## 5. Governance & safety

- **No PHI.** Scheduling data has no patient content; all data here is synthetic. The one sensitive surface is HR data (PTO reasons and pay-differential eligibility), which gets **column masking** for non-scheduler roles.
- **Human-in-the-loop is the design, not an add-on.** The system produces a **draft**. Nothing is published without the scheduler's explicit approval in the app. The optimizer proposes, the human disposes. This is the same "assistant, not autonomous actor" stance as the clinical blueprints, applied to operations.
- **Audit trail.** Every solver run, every manual edit, and every approval is written to Delta with actor + timestamp (UC system tables + app-side logs). If a physician disputes a block, there is a defensible record of why it was assigned and who signed off.
- **Explanation faithfulness.** Because the agent explains schedules physicians will push back on, its "why" answers must be grounded in real solver facts (Stage 4) and are eval-scored for faithfulness (Stage 6). No hallucinated justifications.
- **Constraint correctness is a safety property.** Coverage minimums and credentialing are patient-safety constraints, not preferences, so they are modeled as **hard** constraints so the solver can never trade them away for a happier preference score.

## 6. What a team can realistically build in the hackathon

**Scope IN (the ~1.5-day MVP):**
- Read the `sched_*` gold tables.
- A CP-SAT (or PuLP) model with a **meaningful subset** of constraints: coverage minimums + credentialing + max-consecutive + PTO-hard as the hard set; preference weight + night/weekend equity as the soft objective. Solve one 28-day block.
- Write `sched_draft_assignments` and a `sched_solver_run` row.
- A minimal explanation layer: even a single `ai_query` call that turns the solver's per-provider factors into a plain-language "why" paragraph counts. Bonus: wire one what-if ("give provider X day D off") that re-solves.
- A thin front end: a Databricks App **or** even a notebook dashboard / Genie Agent over the draft table showing the calendar, coverage fill, and equity spread.

**Scope OUT (don't drown):**
- Full Lightning Bolt round-trip / write-back integration: read the synthetic export, don't build the connector.
- Every real union/pay rule: model 2–3 parameterized ones, leave the rest as `sched_pay_rules` rows the model *could* consume.
- Production auth, Lakebase, DABs, multi-user editing: that is path-to-prod.
- A polished drag-and-drop calendar. A read-only grid plus chat is plenty for the read-out.

**The demo that wins:** show a spreadsheet-shaped mess of preferences/PTO/coverage, press a button, get a feasible compliant draft in seconds, then ask the agent "why is Dr. X on nights?" and "give her the 14th off" and watch it re-solve. That is the exact moment Copilot couldn't reach.

## 7. Path to production

1. **Confirm the source system** (Lightning Bolt vs. other, TO CONFIRM) and build the real read (and, later, write-back) integration in a Lakeflow pipeline.
2. **Complete the rule set** with the Nampa office: capture every real union/pay/credentialing rule as parameterized `sched_pay_rules` and encode them; validate feasibility against a few historical blocks.
3. **Harden the solver job** with a time budget, infeasibility diagnostics (report *which* constraint made it infeasible so the scheduler can relax it), and warm-starting from the prior block for fast re-solves.
4. **Productionize the app:** OBO auth, Lakebase state, role-based masking, multi-week horizon, publish + communicate (per-provider exports, calendar feeds).
5. **Eval gate + monitoring** (Pattern J): a schedule cannot be published unless it passes the feasibility + equity gate; trace every re-solve and agent answer in prod; align the faithfulness judge to the scheduler's thumbs-up/down.
6. **DABs** for CI/CD across dev → the Nampa production workspace.

Honest read-out estimate: a credible piloted version (real data, real rules, review app, one office) is on the order of a few focused weeks, not months. The solver and the app are the two real builds, and both start from the hackathon artifact.

## 8. Competitive angle

**This is the use case that names the loss out loud: Microsoft Copilot-in-Excel was already tried here and failed:** it cannot hold the interacting constraints, because an LLM is the wrong tool for discrete optimization. The Databricks answer is architectural: a real solver (OR-Tools CP-SAT) does the math on governed Delta tables, an LLM does *only* what LLMs are good at (explaining and translating requests), and a governed app keeps the human in charge. Copilot Studio / Fabric has no optimization engine, no governed feature layer with lineage feeding it, and no way to keep the model from being asked to do the one thing it fundamentally can't. "Let the LLM write the schedule" is why they failed; "let the LLM explain the schedule the solver wrote" is why we win.

## 9. Facilitation notes

- **The aha to steer toward:** the moment a team stops trying to get the LLM to *produce* the schedule and instead gives that job to the solver, everything clicks. Some teams will instinctively reach for "just prompt the model to make a schedule," which is literally the Copilot failure. Redirect fast: the LLM's job is explanation and translation, not optimization.
- **Where teams get stuck #1: infeasibility.** If they over-constrain (too-tight coverage, too few credentialed providers), CP-SAT returns INFEASIBLE with no schedule and they panic. Coach: start with hard constraints loose enough to be satisfiable, add soft ones to the objective, and use the generator's knobs (it's tuned so a solution exists). Teach them to relax one hard constraint at a time.
- **Where teams get stuck #2: modeling the "no night-then-day" / consecutive rules.** These reified/sequence constraints are the fiddly part of CP-SAT. Point them at OR-Tools examples for shift scheduling; a simpler team can drop these to soft penalties for the MVP.
- **The data is deliberately in tension:** preferences and coverage don't all fit, so a naive "everyone gets their first choice" fails and the equity/preference trade-off actually matters. That tension is the point; don't let them assume the generator is broken when not everyone is happy.
- **Solver choice:** default them to CP-SAT. Only steer a team to PuLP if they're already fluent in LP and want a pure-linear formulation, and warn them they'll hand-encode the logical constraints.
- **Keep the explanation grounded.** Watch for teams whose "why" agent just makes up plausible reasons. The reasons must come from solver output. That's both the credible-demo move and the Pattern J eval.
