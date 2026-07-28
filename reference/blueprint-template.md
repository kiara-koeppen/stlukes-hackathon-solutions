# Blueprint Template

Every full hackathon blueprint follows this shape. Keeps them comparable and makes sure each one
answers the same questions: what are we solving, which Databricks features, how do they chain, what
does the data look like, and how do we productionize.

---

# [##] Use Case Name

**Group:** | **Requester:** | **Problem archetype:** | **Priority:**

## 1. Problem & desired outcome
2–4 sentences. Current-state pain (from the candidate doc) + the target experience. Name the human
who feels the pain today.

## 2. Solution in one picture
A text/ASCII architecture diagram showing the flow from source systems → Databricks layers → the
delivered experience. Reader should grasp the whole thing in 15 seconds.

## 3. The chained architecture (step by step)
Numbered stages. For each stage: **what happens**, **which Databricks feature** (reference a Pattern
letter from databricks-patterns.md), and **why this feature vs. alternatives**. This is the core of
the blueprint - the "which feature does what and how it chains" the whole exercise is about.

## 4. Data model
The synthetic tables this solution reads/writes, with key columns and grain. Cross-reference the
generator in the starter repo. Call out the governance shape (masking, RBAC) where relevant.

## 5. Governance & safety
PHI/PII handling, RBAC, audit, human-in-the-loop decision points, clinical/HR-legal constraints.
Mandatory for clinical (#1, #13) and sensitive (#12, #2, #10) use cases.

## 6. What a team can realistically build in the hackathon
The MVP slice that fits in ~1.5 days. Explicitly scope IN and scope OUT so a group doesn't drown.
This is what I coach them toward.

## 7. Path to production
What it takes to go from hackathon MVP → deployed. The honest "N days to prod" story for the
leadership read-out. DABs, real-data cutover, eval gates, monitoring.

## 8. Competitive angle
One line: what this shows that Copilot Studio / Fabric can't do (governance, lineage, scale,
optimization, eval). Ties back to the Fabric-risk backdrop.

## 9. Facilitation notes
Gotchas, the "aha" moment to steer toward, and where teams will get stuck. For my eyes at the event.
