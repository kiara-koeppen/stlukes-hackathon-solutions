# Hackathon Facilitation Guide

Kiara's playbook for running the St. Luke's AI Dev Collaborative Hackathon. Internal.

## At a glance

- **When:** Wednesday Sept 16, 2026 (full day) + Thursday Sept 17, 2026 (half day). *(Confirmed by
  Yutong Aug 4 - moved from the earlier Sept 2–3 hold.)*
- **Where:** St. Luke's, Boise ID (room TBD).
- **Host:** Yutong Luo (Office of the CIO) + the AI Dev Collaborative.
- **Databricks team:** Kiara (lead instructor), Michael Comerford (AE), Mobeen Vaid (SA).
- **Environment:** non-CSP sandbox workspace (Azure, West US 2), AD group `R-AiDevCollab_GG_AP`.
  Genie Code + Model Serving/FMAPI + Unity AI Gateway enabled. **Comms = email** (no Teams guest
  access).
- **Data:** synthetic only, no PHI. Everything loads from the `stlukes-hackathon-starter` repo.

## Groups and use case assignments

Three groups of four, each mixing an app/Azure dev, a data person, and a cyber engineer.

| Group | Members | Primary use case | Build schema |
|---|---|---|---|
| **Group 1** | Yutong Luo, Gabe Alves Civelli, Jerome Emilo, Shawn Wood | **#12 Diversion Support** | `group1_diversion` |
| **Group 2** | Drake Anshutz, Clara Lyon, Justin Hamilton, Terry Parker | **#6 Hospitalist Scheduling** | `group2_scheduling` |
| **Group 3** | Tyler Hodgkin, Christopher Grapes, Max Ruiz (intern), Carter Brinton | **#1 CKD** + **#14 HTM** | `group3_ckd_htm` |

*#10 Nursing Position Control is the highest-priority use case (!!) but wasn't cleanly assigned to a
single group in the last email. Options: fold it into Group 2 (scheduling-adjacent) as a stretch, or
pitch it as the cross-group "if you finish early" target. Decide on the prep call. It's the one
leadership most wants to see move to prod, so keep it visible even if no group fully builds it.*

## The competitive backdrop (keep this in mind all event)

St. Luke's is deeply Microsoft-invested and is actively using Copilot Studio (via Avanade). The real
risk is Microsoft Fabric. Every "aha" you steer a group toward should implicitly answer *why
Databricks and not Copilot/Fabric*: governed **Genie Agents** over real UC tables (lineage,
row/column security), **Unity AI Gateway** as the one control plane governing every model call
(budgets, PHI guardrails, audit - the thing a Copilot/Fabric stack can't show), constraint
optimization Copilot-in-Excel couldn't do (#6), MLflow eval for "how do you know it's right," and
the clean POC to prod path. That last one matters because Molly's
AI OKR reportedly wants 2 solutions moved POC to prod, and Kiara's whole thesis is "we built it in a
day, we can ship it in N."

## Day 1 (Wednesday, full day)

**Morning: Databricks-led demos - hybrid shape (one anchor deep-dive, then a capability tour).**
Full beat-by-beat script lives in `facilitation/agenda-and-demo-plan.md`; quick version:
- **Anchor deep-dive on HTM (#14, non-PHI, safe to project).** Walk the room through the blueprint
  live: ingest to medallion, Metric Views, then the money shot - a **Genie Agent** answering "What
  anesthesia machines need replacement in 2026?" over governed tables, then surface the same answer
  to a "leader" through **Genie One**. This is the demo that lands the Genie-vs-Copilot story.
- **Capability tour** of the hackathon toolkit, each tied to a use case: **Genie Code** (how they'll
  actually build), **Genie Agents** (+ Analyze Files in Volumes for CKD notes), **Genie One** (the
  leader surface), AI functions in SQL, Vector Search, Model Serving + composable custom agents,
  Databricks Apps, MLflow eval, and **Unity AI Gateway** (budgets + PHI guardrails + audit - the
  governance beat that separates us from Copilot/Fabric).
- **Thread #10 Nursing Position Control as the north star** - the one leadership most wants in prod.
  Reference it throughout even though it's the safe-to-project anchor that's HTM, so the read-out
  narrative connects to Molly's OKR.
- Show the starter repo, the setup notebooks, and how synthetic data loads.

**Late morning: environment onboarding.** Everyone into the sandbox, run `00_setup_catalog` and load
their use case's synthetic data. Confirm Genie Code + Model Serving access per person.

**Afternoon: groups start building.** Each group opens its `usecases/<uc>/README.md` and gets going.
Kiara/Mobeen float. Goal by end of day: data loaded, medallion started, the core "engine" of their
use case scoped (the rules for CKD, the constraints for scheduling, the anomaly features for
diversion, the Genie Agent for HTM).

## Day 2 (Thursday, half day)

**Morning: build sprint.** Push each group to a demoable slice (see each blueprint's "What a team can
realistically build" section for the MVP scope). Help them get *something* end-to-end rather than one
polished piece.

**11:00 leadership read-out.** Molly, Rex, and other leaders join only for this. Each group demos.
Kiara frames the through-line: what got built in ~1.5 days, and the concrete path to production for
the 1-2 strongest (tie to the AI OKR). Have the "path to production" section from each blueprint
ready as talking points.

## Per-group coaching cheat-sheet

Each full blueprint has a **Section 9: Facilitation notes** with the specific "aha" to steer toward
and where teams get stuck. Skim all five before Day 1. Quick version:

- **#1 CKD (G3):** the aha is finding lab-evidence-of-CKD patients with NO N18.x code (the care gap).
  Steer them to rules-first (KDIGO staging in SQL), then `ai_extract` on notes. Watch for them
  overreaching into a full clinical model, keep it to flagging for review.
- **#6 Scheduling (G2):** the optimizer does the math, the LLM explains it, the human approves. Watch
  for them trying to make the LLM do the scheduling, it can't. Point them at OR-Tools/PuLP early.
- **#10 Nursing (stretch):** the crux is entity reconciliation across roster/TAS/HR (the synthetic
  data has intentional mismatches). Then `ai_forecast` by pay period.
- **#12 Diversion (G1):** rules before ML. The synthetic data has planted diversion patterns, so they
  get signal. Keep governance front and center (RBAC, audit) since Yutong's group owns this and it's
  the most sensitive.
- **#14 HTM (G3):** the Genie Agent is the star. Get the gold asset table + Metric Views right and
  the NL queries just work. Easiest to make demo-ready, good confidence builder.

## Open items to close on the prep call (propose ~1 week out - week of Sept 8, MT)

- [ ] Confirm the final use case ↔ group assignments (esp. where #10 lands).
- [ ] Get the real `catalog / schema / table / column / type / count` for all 5 (emailed 7/28).
      Feed corrections into the starter-repo schemas so synthetic shapes match reality.
- [ ] Confirm the hospitalist 3rd-party tool (Lightning Bolt?).
- [ ] Confirm room, timing, and who from leadership makes the 11:00 read-out.
- [ ] Verify participant list + workspace access provisioned (AD group populated).
- [ ] Verify "Jay"/"Reed" names from the planning meeting (not found in any record yet).
