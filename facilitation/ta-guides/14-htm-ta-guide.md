# TA Guide: Use Case 14, NextGen HTM Equipment Planning

Facilitator/TA guide. Internal, do not share with SLHS. Companion to the full architecture in
`blueprints/hackathon/14-nextgen-htm.md` and the shared patterns in `reference/databricks-patterns.md`.

This is the reference a floating TA uses to (a) understand the use case, (b) drop known-good Genie
Code prompts a stuck team can paste, and (c) show a working solution if a team stalls. Everything in
the "Tested" sections below was actually built and run in `kk_test` on 2026-08-17.

**Verification legend:**
- **[outcome-verified]** = I ran the exact code/SQL a Genie Code prompt targets and confirmed the result.
- **[agent-verified]** = I asked the live Genie Agent this and captured the real returned SQL + answer.
- **[design]** = recommended extension, pattern is sound but not executed in this pass.

---

## 1. The use case in 60 seconds (for the TA)

HTM = Healthcare Technology Management (the biomed team that owns ~14,000 medical devices). The job:
plan device **replacement** before machines age out of vendor support, and flag security exposure
from devices on end-of-life operating systems. Today this lives in spreadsheets and TMS exports.

**Data (synthetic, non-PHI, already loaded):**
- `kk_test.stlukes_shared.htm_assets` (14,000 rows): one row per device, with `device_type`,
  `facility`, `end_of_support_date`, `replacement_cost`, `os_eol`, etc.
- `kk_test.stlukes_shared.htm_work_orders` (168,361 rows): PM + repair history per asset.

**The engineered signal (what makes the demo land):** ~36% of assets are already past end of
support, 12.7% run an end-of-life OS, and **2026 is a deliberately large replacement cohort (3,467
assets, ~$395M)** so "what needs replacement in 2026?" returns a rich answer. Work-order burden rises
with age (Critical-tier assets average ~15 work orders vs ~7 for Low).

**Why it is the Day-1 anchor:** non-PHI (safe to project) and the cleanest Genie story.

---

## 2. Tested reference solution

### 2a. Gold table [outcome-verified]

One row per asset with governed derived fields + work-order rollups + a `risk_tier`. This exact SQL
ran clean and built `kk_test.stlukes_sol_htm.htm_asset_gold` (14,000 rows):

```sql
CREATE OR REPLACE TABLE kk_test.stlukes_sol_htm.htm_asset_gold AS
WITH wo AS (
  SELECT asset_number,
         count(*)                                          AS wo_count,
         sum(CASE WHEN wo_type='Repair' THEN 1 ELSE 0 END) AS repair_count,
         round(sum(cost),2)                                AS total_wo_cost,
         round(sum(downtime_hours),1)                      AS total_downtime_hours
  FROM kk_test.stlukes_shared.htm_work_orders
  GROUP BY asset_number
)
SELECT
  a.asset_number, a.description, a.device_type, a.manufacturer, a.model,
  a.facility, a.clinical_area, a.medigate_device_class, a.asset_status,
  a.purchase_date, a.end_of_support_date, a.replacement_cost,
  a.operating_system, a.os_eol,
  year(a.end_of_support_date)                                      AS eos_year,
  (a.end_of_support_date < current_date())                        AS is_past_eos,
  round(datediff(a.end_of_support_date, current_date())/365.25, 2) AS years_to_eos,
  round(datediff(current_date(), a.purchase_date)/365.25, 2)       AS age_years,
  coalesce(w.wo_count, 0)                                          AS wo_count,
  coalesce(w.repair_count, 0)                                      AS repair_count,
  coalesce(w.total_wo_cost, 0)                                     AS total_wo_cost,
  coalesce(w.total_downtime_hours, 0)                              AS total_downtime_hours,
  CASE
    WHEN a.end_of_support_date < current_date() OR a.os_eol          THEN 'Critical'
    WHEN datediff(a.end_of_support_date, current_date())/365.25 <= 1 THEN 'High'
    WHEN datediff(a.end_of_support_date, current_date())/365.25 <= 3 THEN 'Medium'
    ELSE 'Low'
  END                                                              AS risk_tier
FROM kk_test.stlukes_shared.htm_assets a
LEFT JOIN wo w USING (asset_number)
```

Verified `risk_tier` distribution (shows the age-correlated burden the demo relies on):

| risk_tier | assets | avg years_to_eos | avg work orders | replacement spend |
|---|---|---|---|---|
| Critical | 6,172 | -1.43 | 15.1 | $695.6M |
| High | 2,408 | 0.46 | 11.7 | $229.9M |
| Medium | 3,448 | 1.89 | 9.8 | $406.7M |
| Low | 1,972 | 4.03 | 6.8 | $192.1M |

(Column comments were also applied to the gold table so Genie interprets `eos_year`, `risk_tier`,
`os_eol`, etc. correctly. This materially improves Genie answer quality; do not skip it.)

### 2b. Genie Agent [agent-verified]

Created a Genie Agent over the gold table (`space_id 01f19a70f76118bc827d20207c4ffa26` in kk_test).
Three questions were asked to the **live** agent and returned correct answers:

| Question | Genie's generated SQL (abridged) | Verified answer |
|---|---|---|
| What anesthesia machines need replacement in 2026? | `WHERE device_type ILIKE '%anesthesia machine%' AND eos_year=2026` | 317 assets; by facility Boise 79, Meridian 63, Magic Valley 39, Nampa 37, ... |
| Which facilities have the most assets past end of support? | `WHERE is_past_eos=TRUE GROUP BY facility ORDER BY count DESC` | Boise 1,266 › Meridian 942 › Magic Valley 687 › Nampa 663 › ... |
| Total replacement spend by year, next 5 years? | `SUM(replacement_cost) ... eos_year BETWEEN year(current_date) AND +4` | 2026 $394.8M, 2027 $252.5M, 2028 $246.4M, 2029 $175.3M, 2030 $85.7M |

Curation that made these land: good column comments, the five sample questions on the agent, and a
gold table with derived fields baked in (Genie does not have to compute `years_to_eos` on the fly).

---

## 3. Genie Code prompt playbook

Genie Code is how the team builds. Below are prompts to hand a team, in order, each with the code it
should produce and the verified result. A TA can let the team drive Genie Code live, or paste the
reference code from Section 2 if they are stuck. (Genie Code is an interactive assistant; its exact
wording will vary, but these prompts reliably steer it to the tested code.)

**Prompt 1, build the gold table** [outcome-verified target]
> "I have two tables, `kk_test.stlukes_shared.htm_assets` (one row per medical device) and
> `kk_test.stlukes_shared.htm_work_orders` (maintenance history). Build a gold table
> `kk_test.<my_schema>.htm_asset_gold` with one row per asset. Add derived columns: `eos_year`,
> `is_past_eos`, `years_to_eos`, `age_years`, and work-order rollups `wo_count`, `repair_count`,
> `total_wo_cost`. Add a `risk_tier` column: Critical if past EOS or on an end-of-life OS, High if
> within 1 year of EOS, Medium if within 3 years, else Low. Add table and column comments."
> → Should produce the CTAS in Section 2a. Verified: 14,000 rows, risk tiers as tabled above.

**Prompt 2, sanity-check the signal** [outcome-verified target]
> "Show me the count of assets and total replacement cost by `eos_year`, and separately the percent
> of assets that are past EOS and the percent on an end-of-life OS."
> → Verified: 2026 cohort = 3,467 assets / ~$395M; 35.9% past EOS; 12.7% on EOL OS.

**Prompt 3, stand up the Genie Agent** [agent-verified]
> "Create a Genie Space over `kk_test.<my_schema>.htm_asset_gold` for HTM replacement planning. Add
> these sample questions: what anesthesia machines need replacement in 2026; which facilities have
> the most assets past end of support; total replacement spend by year for the next 5 years."
> → Verified: the agent answers all three correctly (Section 2b).

**Prompt 4 (extension), Metric View for governed KPIs** [design]
> "Create a Unity Catalog metric view over `htm_asset_gold` with measures asset_count,
> replacement_spend (sum of replacement_cost), pct_past_eos; dimensions facility, device_type,
> eos_year, risk_tier."
> Pattern is sound (Pattern D). Not executed in this pass; build live if the team gets here.

**Prompt 5 (extension), dashboard / forecast** [design]
> "Build an AI/BI dashboard with replacement spend by year and a facility heatmap of past-EOS
> assets," and/or "use `ai_forecast` to project replacement volume by year." Nice-to-have polish.

---

## 4. Tiered hints (dribble these out; do not lead with the answer)

Give L1 first. Escalate only if the team is still stuck after a real attempt.

- **L1 (nudge):** "Everything you need is in `htm_assets` + `htm_work_orders`. What one table would
  make the replacement-planning question trivial to ask?" (Steer to a gold table.)
- **L2 (point at the tool):** "Bake the hard stuff into gold as columns, `is_past_eos`, `eos_year`,
  `risk_tier`, so Genie never has to compute it. Then a Genie Agent over gold answers the questions."
- **L3 (show the shape):** share the `risk_tier` CASE logic and the work-order rollup CTE from
  Section 2a; let them write the rest.
- **L4 (unblock):** paste the full CTAS (2a) and the Genie Agent prompt (Prompt 3). Get them to a
  working demo, then push them to add a Metric View or dashboard.

---

## 5. Where teams get stuck (watch for these)

1. **Trying to make Genie compute `years_to_eos` / risk on the fly.** Answers get inconsistent. Push
   them to bake `is_past_eos`, `eos_year`, `risk_tier` into gold first.
2. **Under-curating the Genie Agent.** With no column comments, glossary, or sample questions, Genie
   guesses. The comments in 2a and the sample questions in Prompt 3 are what made it reliable.
3. **Losing time building an App** when a Genie Agent or a dashboard already tells the story. Time-box
   the App; it is the extension, not the MVP.
4. **Device-type string matching.** Genie used `ILIKE '%anesthesia machine%'`, which works here.
   If a team hardcodes exact equality and gets zero rows, check casing/spacing.

---

## 6. What "done" looks like for the read-out

A Genie Agent (or dashboard) answering, live: "What anesthesia machines need replacement in 2026?"
returning 317 devices across 8 facilities, plus the replacement-spend-by-year view. Bonus: the
security angle, "show Critical devices on an end-of-life OS." That is a 90-second, non-PHI,
governed-answer demo that directly contrasts with a Copilot/Fabric black box.

---

*Built and verified in kk_test on 2026-08-17. Genie Agent space_id 01f19a70f76118bc827d20207c4ffa26.
Data: kk_test.stlukes_shared.htm_* ; solution: kk_test.stlukes_sol_htm.htm_asset_gold.*
