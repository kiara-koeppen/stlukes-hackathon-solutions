# Backlog Use Case Sketches (Non-Hackathon Candidates)

These are the 9 candidate use cases that did **not** make the starred hackathon shortlist. They live
here as lightweight architecture sketches, not full blueprints. The five hackathon use cases get the
full nine-section treatment (problem, solution picture, chained architecture, data model, governance,
MVP scope, path to prod, competitive angle, facilitation notes). Each sketch below is deliberately
thinner: one problem restatement, the recommended Databricks feature chain (referencing Pattern
letters A-J from `reference/databricks-patterns.md`), the primary delivered experience, and one honest
note on data readiness or the biggest open question. The goal is coverage, so Kiara can speak to any
of these if a stakeholder asks, without over-investing in ones that may never run. All patterns,
source systems, and governance guardrails assume the shared context in `reference/platform-architecture.md`.

A quick honesty pass up front: **#5 (PMO intake)** is the cleanest quick win and the easiest to
productionize; **#8 (dictation)** is the weakest native Databricks fit and is mostly an integration
play; and **#11 (Nursing Navigator)** and **#13 (Hospital at Home)** are both governance-heavy and
exploratory, so those two are gated on content/CDS approval more than on engineering.

---

## 2. HR Leaves & Benefits Agent

**Problem:** HR benefit/leave/insurance policies are complex and scattered across many documents, and
the current case-manager model feels impersonal and transactional.

This is the archetypal RAG-plus-tools agent. Policy documents (benefits, leave, insurance) get ingested
via **Pattern B** (`ai_parse_document` -> chunk -> Vector Search -> Agent Bricks Knowledge Assistant)
so the agent answers warmly and cites the actual policy language. Leave status, accrual balances, and
case state come from a governed **Pattern D** Genie Space over HR tables, and a **Pattern H** Multi-Agent
Supervisor routes between the policy KA and the status Genie plus custom tools (open an HR case when it
detects complexity, draft an OOO message, propose an Outlook status, generate welcome-back messaging).
Serve it as a **Pattern I** Databricks App with OBO auth so each employee only sees their own leave
context, and wrap it in **Pattern J** eval for tone and groundedness given the legal sensitivity.

**Feature chain:** B (policy KA) + D (leave-status Genie) -> H (MAS) -> I (App, OBO) -> J (eval).

**Delivered experience:** a conversational HR companion that answers benefits questions personally,
proactively reminds employees of leave milestones, and drafts the communications around a leave.

**Readiness / open question:** the Outlook and Teams write-integrations (drafting status, posting team
notifications) are outside Databricks and need Graph API / partner plumbing; and employee identity
context plus HR-legal review are hard prerequisites before any real-data phase. HR benefit docs exist
but case-workflow data may need shaping into queryable tables.

---

## 3. Architecture Review Workflow Agent

**Problem:** architecture reviews are manual and rely on tribal knowledge; architects hand-research
vendors, check for overlap with existing solutions, and prep for governance boards from scratch.

The core is a **Pattern B** RAG agent (Knowledge Assistant) over the internal solution/app catalog plus
the corpus of past architecture-review docs and decisions, so an architect can ask "have we evaluated a
vendor like this before?" and get a grounded, cited answer. Incoming vendor questionnaires and solution
docs are structured with **Pattern C** (`ai_extract` / `ai_classify`) to pull capabilities, categories,
and integration points into a Delta table, which makes overlap and redundancy detection a query rather
than a memory exercise (compare the new vendor's extracted capability tags against the existing catalog
via `ai_similarity`). Meeting capture (transcript -> `ai_summarize` -> decision/Q&A/rationale record)
closes the loop so review outcomes feed back into the searchable corpus.

**Feature chain:** C (extract vendor questionnaires) + B (RAG over catalog + review docs) -> `ai_similarity`
overlap check -> meeting capture -> optional I (App).

**Delivered experience:** an assistant that flags overlaps and prior evaluations before a review, and
captures decisions and rationale during it.

**Readiness / open question:** biggest dependency is a usable internal solution catalog; if that
inventory is stale or lives only in people's heads, the overlap-detection value collapses. Confirm the
catalog exists in a queryable form before scoping.

---

## 4. Epic Tribal Knowledge & Learning Agent

**Problem:** critical Epic configuration and build knowledge lives in individual experts' heads,
onboarding is slow, and knowledge is lost when people leave.

This is a multimodal **Pattern B** build with an ingest twist. Teams recordings, demo videos, and voice
notes go through transcription; OneNotes, screenshots, and Epic docs/training PDFs go through
`ai_parse_document` (which handles images and mixed content). Everything is chunked and embedded into a
**Vector Search** index, then fronted by an **Agent Bricks Knowledge Assistant** that gives Epic analysts
and new hires source-grounded, cited answers. A **Pattern A** medallion layer keeps the raw media,
transcripts, and derived chunks organized and re-indexable as new recordings arrive. Add **Pattern J**
eval so answers about build configuration are trustworthy, not hallucinated.

**Feature chain:** A (media/transcript medallion) -> B (parse + transcribe -> Vector Search -> KA) ->
J (eval). Multimodal ingest is the distinguishing piece.

**Delivered experience:** a searchable, always-on structured knowledge library and dynamic Q&A for Epic
analysts and onboarding hires.

**Readiness / open question:** volume and quality of transcripts/recordings, and whether they can be
collected into the sandbox at all (Teams recordings may have retention and consent constraints).
Transcription accuracy on Epic-specific jargon will need a domain glossary. Lead: Mike Atkinson.

---

## 5. PMO Project Intake Automation Agent

**Problem:** project intake relies on Word request forms that are manually copy-pasted into SharePoint,
which is inefficient and does not scale.

This is the simplest use case in the backlog to productionize and the strongest quick-win candidate.
It is almost pure **Pattern C**: Auto Loader picks up Word intake forms landing in a volume,
`ai_parse_document` converts them, and `ai_extract` pulls the key fields (requestor, sponsor, business
need, systems, estimated effort, priority) into a structured Delta table. From there a **Pattern D**
Genie Space (or a simple search UI) makes the whole intake backlog queryable and discoverable, replacing
the copy-paste. No agent orchestration, no clinical governance, no forecasting - just a clean
parse-extract-structure-query pipeline.

**Feature chain:** A/Auto Loader ingest -> C (`ai_parse_document` + `ai_extract`) -> Delta -> D (Genie /
search) -> optional I (App).

**Delivered experience:** Word forms become structured, searchable, deduplicated intake records with
near-zero manual data entry.

**Readiness / open question:** cleanest of the nine - the main dependency is getting intake Word docs
and SharePoint list access into the sandbox. The open question is write-back: does the PMO want results
pushed back into SharePoint, or is a Databricks-side view sufficient?

---

## 7. New Solution Review "Butler" Agent

**Problem:** requesters cannot navigate the complex multi-council approval process (NSRB, AIAC, DRRC,
SAF, Cyber, HR), causing confusion, redundancy, and delays.

Two halves. A **Pattern B** RAG agent (Knowledge Assistant) over the process documentation and each
council's requirements answers "what do I need for the AIAC step and what happens next?" with grounded,
cited guidance. A **Pattern D** Genie Space over a workflow-state table tracks where a given submission
sits across all councils, so the agent can also answer "where is my request right now?" These combine
naturally under a **Pattern H** Multi-Agent Supervisor, but the headline is the delivery surface: a
**Pattern I** Databricks App that renders a visual journey map with milestones and timelines, giving
requesters a single source of truth.

**Feature chain:** B (process/requirements KA) + D (status Genie over workflow-state table) -> H (MAS) ->
I (App with journey-map UI).

**Delivered experience:** a guided, visual "butler" that walks a requester through every council step,
explains required documents, and shows live status across the approval journey.

**Readiness / open question:** the status-tracking half only works if there is a real workflow-state
data source; today that state may be spread across email, SharePoint, and people, so the biggest open
question is whether council status can be captured in a single table. The RAG half can ship on process
docs alone.

---

## 8. Clinical Dictation & Transcription Enhancement

**Problem:** dictation and transcription in the Boise Gross Lab are slow and error-prone, existing tools
need too much manual interaction, and hands-free operation is poor.

Honest framing: this is the **weakest native Databricks fit** in the backlog. The core need - a
low-latency, voice-activated, hands-free ASR experience integrated with Epic Beaker - is fundamentally a
specialized real-time speech product (Nuance/DAX, Dragon Medical, or a dedicated streaming ASR service),
not something Databricks does well as a live capture layer. Where Databricks *does* fit is the
back-end enhancement and integration layer: batch transcripts (from whatever ASR captures them) can be
post-processed with `ai_fix_grammar` and a **Pattern C** `ai_query` step tuned with a pathology
terminology prompt to correct domain-specific errors, and structured/normalized output can be routed
toward Beaker workflows. Any accuracy improvement should be measured with **Pattern J** eval against
reference transcripts.

**Feature chain:** (external ASR captures audio) -> C (`ai_fix_grammar` + `ai_query` domain post-processing)
-> J (accuracy eval). Live capture stays with a specialized ASR vendor.

**Delivered experience:** cleaner, terminology-corrected transcripts feeding pathologist workflows -
realistically a post-processing assist, not a full dictation replacement.

**Readiness / open question:** be candid with the Pathology/PA team that Databricks is the enrichment and
integration layer here, not the microphone. The real decision is which ASR engine owns capture; only
then does the Databricks post-processing story make sense.

---

## 9. Clinical Research Shared Mailbox & Meeting Agent

**Problem:** research requests arrive via a shared mailbox, and the team struggles to prep for meetings
and track projects, documents, decisions, and next steps.

A blend of classification and retrieval. Incoming emails are triaged with **Pattern C** (`ai_classify`
to categorize the request type, `ai_extract` to pull requirements, requested materials, and requester
details) into a structured Delta table. Attached and referenced research docs feed a **Pattern B** RAG
index so the agent can assemble a meeting brief grounded in the actual materials. Meeting capture
(transcript -> `ai_summarize`) records decisions and action items back into each request's record. The
delivery surface is a **Pattern I** Databricks App that organizes every request into a single project
view - email thread, extracted requirements, related docs, meeting notes, and next steps in one place.

**Feature chain:** C (`ai_classify` / `ai_extract` on emails) + B (RAG over research docs) -> meeting
capture (`ai_summarize`) -> I (project-view App).

**Delivered experience:** each research request becomes a self-organizing project workspace, with
auto-categorized email, a prepared meeting brief, and captured decisions/actions.

**Readiness / open question:** shared-mailbox access is the gating dependency (Graph API / Exchange
export into the sandbox), and the "participates in meetings to capture decisions" piece needs a
transcript source. Clarify whether real-time meeting participation is expected or async transcript
processing is acceptable.

---

## 11. Nursing Knowledge Navigator

**Problem:** nurses need a faster, reliable way to find trusted policies, procedures, and practice
guidance that today are spread across The Source, C360, Elsevier, and related repositories.

Architecturally this is a classic, almost textbook **Pattern B** RAG / Agent Bricks Knowledge Assistant
with strict source-grounding and mandatory citations - approved nursing content is parsed, chunked,
embedded into Vector Search, and answered over by a KA that always shows its sources. **Unity Catalog**
governs which repositories a given user (bedside nurse, charge nurse, manager, educator) can retrieve
from, and **Pattern J** eval enforces that answers are grounded only in approved content, with no
free-lancing. The engineering is well-understood; the hard part is not technical.

**Feature chain:** B (approved-content KA, citations enforced) + UC repository ACLs -> J (groundedness /
safety eval) -> optional I (App, possibly Epic-linked).

**Delivered experience:** natural-language questions returning trusted, source-grounded, cited answers
scoped to what each nursing role is approved to see.

**Readiness / open question:** this is **governance-heavy and exploratory, not yet approved.** Before any
build, confirm content governance (which repositories are in scope and who approves inclusion), Elsevier
licensing for ingestion, repository access, the intended summarization scope, and whether the eventual
Epic linkage is validated. The blocker is approval and content governance, not RAG mechanics.
Requesters: Scott Pyrah & Jenny Hopkins.

---

## 13. Hospital at Home Patient Candidate Identification

**Problem:** the team needs a consistent way to identify patients appropriate for Hospital at Home review,
but the clinical criteria, workflow, data availability, and governance still need discovery.

Same shape as CKD (#1): a screening-and-flagging workflow, never an autonomous placement engine. A
**Pattern A** medallion layer conforms the relevant Epic signals (admission status, diagnosis/problem
list, acuity/stability indicators, home-support and payer/program eligibility) into gold feature tables.
A deterministic **rules engine** encodes the approved eligibility criteria first, and only the fuzzy
summarization step calls a model - a **Pattern C** `ai_query` that generates a plain-language "why this
patient does or does not meet criteria" narrative. Flagged candidates land on a **human-in-the-loop**
worklist in a **Pattern I** Databricks App (mirroring the CKD triage app), where a clinician makes the
actual decision. Every generation is logged via MLflow for audit.

**Feature chain:** A (Epic signal medallion) -> deterministic rules engine -> C (`ai_query` eligibility
summary) -> I (human-review worklist App) -> J (eval + audit). Rules are the backbone; the LLM only
explains.

**Delivered experience:** a candidate worklist that flags patients for clinician review and summarizes
the eligibility rationale, supporting referral conversations rather than making placements.

**Readiness / open question:** the biggest gap is **discovery itself** - the eligibility rules, workflow,
and data availability are all "to confirm," and the requesting team is not yet locked. Like CKD, this
needs a full CDS governance path (human-in-the-loop, RBAC, audit, BAA alignment) and framing as clinical
*decision support*, never automated placement. Intake details are the prerequisite before any build.
