# SLHS Platform Architecture & Source-System Landscape

Reference context that every blueprint assumes. Read this once; the blueprints won't re-explain it.

## Databricks environment

- **Cloud:** Azure. Region preference **West US 2**.
- **Commercial:** Databricks customer since Aug 2019. Currently Ramping; Azure PAYG transitioning
  to Commit. ARR ~$19.5K today, renewal opp ~$212K TCV (close 3/16/2028).
- **Workspace for the hackathon:** dedicated **non-CSP sandbox workspace** (preview record
  FPA-60779). Kiara owns pre-creating per-group catalogs/schemas, enabling **Genie Code**, and
  confirming **Model Serving / Foundation Model API** access.
- **Access:** AD group **R-AiDevCollab_GG_AP**. SLHS cannot enable Teams guest access - **email is
  the comms channel**. Auto-onboard participants to the sandbox.
- **Unity Catalog** is the governance backbone: per-group catalog isolation, table/row/column ACLs,
  lineage, and audit (system tables) - all of which double as talking points vs. Microsoft Fabric.

### Suggested catalog/schema convention for the sandbox

```
hackathon                         (catalog)
├── shared                        (schema - synthetic source data everyone reads)
│   ├── ckd_*                      (use case 1)
│   ├── sched_*                    (use case 6)
│   ├── nursing_*                  (use case 10)
│   ├── diversion_*                (use case 12)
│   └── htm_*                      (use case 14)
├── group1_diversion              (schema - Group 1 build space)
├── group2_scheduling             (schema - Group 2 build space)
└── group3_ckd_htm                (schema - Group 3 build space)
```

Shared synthetic source tables are read-only to all groups; each group gets a writable schema for
their bronze/silver/gold + agent artifacts.

## The strategic backdrop (why this hackathon matters)

- SLHS is **migrating off Health Catalyst with a hard Sept 2027 cutoff** - converting 1,200+ Power
  BI reports (T-SQL → DBSQL) and expanding self-service from ~100 to 500–600 users.
- **Value-Based Care** covering ~350,000 patients is a stated leadership priority.
- **Competitive:** Health Catalyst (incumbent, being replaced), Snowflake (architecture
  discussions), and **Microsoft Fabric** is the real risk - SLHS is deeply Microsoft-invested
  (Azure, Power BI, Epic, Copilot Studio via partner Avanade). The hackathon's job is to show what
  the Databricks platform does end-to-end that a Copilot/Fabric stack can't govern or productionize.

## Source systems referenced across the use cases

| System | What it is | Use cases | Notes for synthetic data |
|---|---|---|---|
| **Epic** (Clarity, Caboodle, Tapestry) | Primary EHR. Clarity = normalized relational, Caboodle = dimensional warehouse | 1, 4, 8, 9, 12, 13 | Labs, diagnoses, MAR, pain scores, provider orders, problem list |
| **Omnicell** | Automated med dispensing cabinets | 12 | Dispense / waste / return records |
| **Bluesight ControlCheck (IRIS)** | Drug-diversion monitoring | 12 | Per-user IRIS risk scores |
| **Lightning Bolt** (to confirm) | 3rd-party physician scheduling; data already lands in Databricks | 6 | Hospitalist schedules, shifts, rules |
| **TAS** | Time & attendance / scheduling validation | 10 | Shift assignments, PTO |
| **Power BI** | Reporting layer (roster exports, demand-based staffing metrics) | 10 | Employee roster, demand metrics |
| **HR systems** | Roster, leave, benefits, HR events | 2, 10 | Start/term dates, LOA, resignations |
| **TMS** | HTM asset inventory (Health Technology Management) | 14 | 14,000+ rows/table; EOL/EOS dates, Medigate fields, work orders |
| **ServiceNow** | ITSM (HTM data landing here "next year") | 14 (future) | Not in scope for hackathon |
| **SharePoint** | PMO intake lists | 5 | |
| **The Source / C360 / Elsevier** | Nursing knowledge repositories | 11 | Policies, procedures, practice guidance |
| **Teams / OneNote** | Meeting recordings, transcripts, notes | 4, 9 | Unstructured knowledge |

## Governance guardrails (apply to every clinical/HR blueprint)

- **PHI never enters the sandbox.** All clinical and HR use cases use synthetic data generated in
  the starter repo. This is both a compliance requirement and a design constraint - the solutions
  must work on de-identified/synthetic shapes.
- **CKD (#1) and Diversion (#12)** are clinical-decision-support / HR-legal sensitive. Both need:
  role-based access (UC row/column masking + `is_account_group_member`), audit trails (UC system
  tables + MLflow trace logging), human-in-the-loop (flag for review, never autonomous action), and
  BAA alignment before any real-data phase.
- **Clinical decision support governance:** CKD staging suggestions and Hospital-at-Home candidacy
  are *suggestions for clinician review*, never automated diagnoses/placements. Every clinical
  blueprint frames the agent as a triage/flagging assistant with a human decision point.

## Databricks feature palette (what's available to chain)

Data & compute: Unity Catalog, Delta, Lakeflow Declarative Pipelines (SDP/DLT), Auto Loader,
Databricks SQL, Serverless.
AI functions in SQL: `ai_query`, `ai_extract`, `ai_classify`, `ai_parse_document`, `ai_summarize`,
`ai_forecast`, `ai_similarity`, `ai_mask`.
Genie family: **Genie Agents** (governed NL analytics, fka Genie Spaces; "Analyze Files in Volumes"
is Beta), **Genie One** (unified business/exec front door), **Genie Code** (the AI build assistant
teams use to develop pipelines, semantic layers, and agents).
Agent tier: **composable custom agents** (ResponsesAgent/ChatAgent) on **Model Serving** / Databricks
Apps, orchestrating Genie Agents + retrieval + tools (MCP), Foundation Model API.
Retrieval: Vector Search (RAG over unstructured content).
ML: MLflow (tracking, models, **GenAI evaluation** + LLM-judge scorers), AutoML/forecasting.
Serving & apps: Model Serving, **Databricks Apps** (React/FastAPI front ends).
Governance/ops: **Unity AI Gateway** (GA - budgets, guardrails, model routing incl. Smart Routing
[Beta], observability/audit of all AI traffic), UC lineage, system tables (audit/billing), Lakeflow
Jobs, DABs for CI/CD.

*Facilitator note (not for SLHS): the interactive Agent Bricks pieces (Knowledge Assistant,
Multi-Agent Supervisor) and the Supervisor API are winding down into the Genie family + composable
agents. Build on the Genie stack + Unity AI Gateway above; don't tell the customer "Agent Bricks is
deprecated" (no customer-communicable EOL).*

Each blueprint says explicitly which of these it uses and in what order.
