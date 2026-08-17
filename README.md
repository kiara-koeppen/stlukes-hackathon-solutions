# St. Luke's AI Hackathon - Solution Blueprints (Facilitator's Repo)

**Private. Facilitator-only. Do not share with St. Luke's.**

This repo is Kiara's internal playbook for the St. Luke's Health System (SLHS) AI Hackathon
(Sept 16–17, 2026, Boise ID). It contains the full solution architecture for each candidate use
case: which Databricks features do what, and how they chain together into a production-ready
solution. The point is to walk into the hackathon knowing exactly what "good" looks like for
every use case so I can guide teams without building it for them.

The companion repo `stlukes-hackathon-starter` is the *shareable* one: it holds the synthetic
data generators, table schemas, and skeleton scaffolding the St. Luke's teams load and build on.
Same tables I designed the solutions against, so we're all working from the same data.

## Why two repos

| Repo | Audience | Contents |
|---|---|---|
| `stlukes-hackathon-solutions` (this one) | **Kiara only** | Full worked architectures, the "answer key," facilitation guide |
| `stlukes-hackathon-starter` | **Shared with SLHS teams** | Synthetic data gen, schemas, empty use-case scaffolds, setup instructions |

## The five hackathon use cases (`**` in the candidate doc)

| # | Use case | Group | Problem archetype | Day-1 demo? |
|---|---|---|---|---|
| 1 | CKD Identification & Risk Flagging | Group 3 | Clinical extraction + rules-based risk flagging | |
| 6 | Hospitalist Scheduling Optimization | Group 2 | Constrained optimization, human-in-loop | |
| 10 | Nursing Position Control & Forecasting | (cross) | Forecasting + multi-source reconciliation | |
| 12 | Diversion Support Reporting | Group 1 | Anomaly detection + report generation | |
| 14 | NextGen HTM Equipment Planning | Group 3 | Asset analytics + NL query + forecasting | ✅ anchor (non-PHI) |

Priority signal: the candidate doc marks **#10 Nursing Position Control as highest priority (!!)**,
which matches Yutong saying it has the most eyes on it. #14 HTM is the Day-1 demo anchor precisely
because it's non-PHI and safe to project.

## Repo layout

```
stlukes-hackathon-solutions/
├── README.md                        ← you are here
├── reference/
│   ├── platform-architecture.md     ← SLHS Databricks landscape, source systems, governance
│   ├── databricks-patterns.md       ← reusable building blocks every blueprint pulls from
│   └── blueprint-template.md        ← the standard shape each blueprint follows
├── blueprints/
│   ├── hackathon/                   ← full worked solutions for the 5 starred use cases
│   │   ├── 01-ckd-risk-flagging.md
│   │   ├── 06-hospitalist-scheduling.md
│   │   ├── 10-nursing-position-control.md
│   │   ├── 12-diversion-support.md
│   │   └── 14-nextgen-htm.md
│   └── backlog/                     ← lighter sketches for the other 9
│       └── other-usecases.md
└── facilitation/
    └── hackathon-guide.md           ← group mapping, day-1/day-2 flow, readout plan
```

## Ground rules baked into every blueprint

1. **Synthetic data only, no PHI** in the workshop environment. Every clinical/HR blueprint is
   designed against synthetic tables in the starter repo. Diversion (#12) and CKD (#1) especially:
   de-identified/synthetic, RBAC, audit trails, BAA-aligned - this is called out explicitly in the
   candidate doc.
2. **Production path is the goal, not just a POC.** Molly's AI OKR reportedly wants 2 solutions
   POC→prod. Each blueprint ends with a "productionize" section so the read-out can credibly say
   "we can ship this in N days."
3. **Azure, West 2, Epic-centric.** SLHS is Azure PAYG→Commit, migrating off Health Catalyst by
   Sept 2027. Microsoft Fabric is the real competitive risk. Blueprints lean into what Databricks
   does that Fabric/Copilot can't (governed Genie Agents on real tables, Unity AI Gateway governing
   every model call, MLflow eval, UC lineage/audit).

## Open items to confirm with Yutong on the prep call

- Exact `catalog / schema / table / column / type / count` for all 5 use cases (requested via
  email 7/28; drives the synthetic schemas - see starter repo).
- Hospitalist 3rd-party tool confirmed as **Lightning Bolt** - verify.
- "Jay" and "Reed" from the meeting don't appear in any email/Slack/Glean - verify names.
- Molly's AI OKR (2 dev envs / 2 POC→prod / $2M) and Avanade/Copilot Studio detail are
  transcript-sourced only - not corroborated internally.

---
*Sourced from the Candidate AI Projects doc, the 7/28 metadata-request email thread, and the
SLHS account research doc. See `reference/platform-architecture.md` for citations.*
