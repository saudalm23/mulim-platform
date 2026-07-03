# PRD — Regulatory Impact Simulation Platform for the Financial Sector
### منصة محاكاة الأثر التشريعي للقطاع المالي
**Codename suggestion: "Mulim" (مُلم)** · Version 1.0 · Prepared for: Amad Hackathon (Saudi Arabia) · Date: July 2026

---

## 1. Overview

**One-liner:** Before any regulation is applied, know its cost, risks, and impact in minutes instead of weeks.
(قبل تطبيق أي تشريع، اعرف تكلفته ومخاطره وتأثيره بدقائق بدل أسابيع)

Mulim ingests (a) a new regulation/circular from a Saudi regulator (SAMA, CMA, NCA, SDAIA) and (b) the institution's internal corpus — policies, procedures, org structure, risk register — then computes the **intersection** between the two. The output is an Impact Score (0–100), a gap map showing exactly which policies/departments/systems are affected, an estimated compliance cost and timeline, and an auto-generated implementation plan.

**Core design principle (the 80/20 rule):** AI does ~80% of the mapping work with a confidence score per link; a human compliance officer reviews, corrects, and approves the remaining 20%. Every AI output is traceable to a source clause (human-in-the-loop governance, same posture as Lameh's "governed layer" model — mandatory for regulated environments).

---

## 2. Problem Statement

When a regulator issues a new regulation (e.g., a circular with 120 requirements), banks and financial institutions cannot quickly answer:

1. What is the real magnitude of the impact?
2. Which departments are affected?
3. Which internal policies need updating?
4. What will compliance cost?
5. How long will implementation take?

Today this analysis takes **weeks of manual work** by compliance, legal, and risk teams reading the regulation line-by-line and cross-referencing hundreds of internal documents. The result is slow responses to regulators, missed deadlines, unbudgeted compliance costs, and penalty exposure (Saudi AML penalties reach ~SAR 5M and prison terms).

---

## 3. Target Users & Personas

| Persona | Role | What they need from Mulim |
|---|---|---|
| **P1 — Chief Compliance Officer** | Owns regulatory response | Impact Score, executive summary, cost/time estimate for board reporting |
| **P2 — Compliance Analyst** | Does the mapping work | Requirement-by-requirement gap view, review/approve AI mappings (the 20%) |
| **P3 — Risk Manager** | Owns risk register | Risk-level changes triggered by the regulation |
| **P4 — Department Head (e.g., Cybersecurity, Marketing)** | Executes changes | Task list scoped to their department only |

**Target segments:** Banks · Fintech companies · Finance companies · Insurance companies (all SAMA/CMA-regulated entities; 261+ licensed fintechs in KSA alone).

---

## 4. Inputs & Outputs

### Inputs (المدخلات)
1. **New regulation** — PDF/DOCX, Arabic or English (SAMA circulars, CMA rules, NCA frameworks, PDPL)
2. **Current policies** — internal policy documents (PDF/DOCX)
3. **Current procedures** — SOPs, process documents
4. **Org structure** — departments + headcount per department (CSV/JSON upload or manual entry)
5. **Risk register** — existing risks with severity levels (CSV/XLSX)
6. *(Optional)* Systems inventory — list of IT systems and owning departments

### Outputs (المخرجات)
1. **Impact Score 0–100** with breakdown by dimension
2. **Gap Map** — e.g., "12 policies need amendment · Marketing dept affected · Cybersecurity dept affected"
3. **Auto-generated implementation plan** — tasks grouped by department, with priority, effort estimate, and suggested deadline
4. **Executive report** — exportable PDF (Arabic + English) for board/regulator

---

## 5. The Three-Stage Pipeline (المراحل)

### Stage 1 — Regulatory Requirement Extraction (استخراج المتطلبات التنظيمية)
- Parse the uploaded regulation (OCR fallback for scanned PDFs).
- LLM extracts atomic **requirements** (المتطلبات): each requirement = {id, source clause text, article number, plain-language summary AR/EN, type}.
- Requirement **type classification** (borrowed from industry best practice): `prescriptive` (must do), `prohibitive` (must not), `permissive` (may), `informative`, `definition`. Only prescriptive/prohibitive requirements drive impact.
- Output: structured requirements table. Human can edit/merge/delete before Stage 2.

### Stage 2 — Mapping to Internal Corpus (الربط مع السياسات)
- All internal documents are chunked and embedded into a vector store (multilingual embedding model that handles Arabic well — e.g., `multilingual-e5-large` or Cohere embed-multilingual).
- For each requirement: **hybrid retrieval** = semantic similarity (embeddings) + keyword/BM25 (regulatory terms, e.g., "غسل الأموال", "العناية الواجبة") → top-k candidate policy sections.
- LLM judges each candidate pair (requirement, policy section) and outputs:
  - `relation`: `covered` (fully addressed) / `partial` (addressed but needs amendment) / `gap` (not addressed anywhere)
  - `confidence`: 0.0–1.0
  - `rationale`: one sentence explaining why (traceability)
- **This is the answer to the "how do we know who's affected" weakness:** the confidence score + hybrid keyword/semantic search IS the mechanism. Thresholds:
  - confidence ≥ 0.85 → auto-accepted (the AI's 80%)
  - 0.5 ≤ confidence < 0.85 → routed to human review queue (the human's 20%)
  - confidence < 0.5 → treated as gap, flagged for review
- Department attribution: each policy document is tagged with its owning department at upload time; affected departments = union of departments owning affected policies + departments named in the requirement text itself.

### Stage 3 — Impact Computation (حسبة التأثير)
Deterministic scoring engine (NOT an LLM — must be explainable and reproducible) over the mapping results.

**The 8 variables (المتغيرات):**

| # | Variable | How computed |
|---|---|---|
| 1 | Affected policies count | # documents with ≥1 `partial` or `gap` mapping |
| 2 | Affected procedures count | same, over procedure corpus |
| 3 | Affected departments count | union as defined in Stage 2 |
| 4 | Affected employees count | Σ headcount of affected departments (from org structure input) |
| 5 | Affected systems count | systems owned by affected departments ∩ systems mentioned in requirements |
| 6 | Risk level | max/weighted-avg severity of risk-register items linked to gap requirements; gaps with no existing risk item raise risk automatically |
| 7 | Expected cost (SAR) | cost model: (policy amendments × avg. rework cost) + (affected employees × training cost/employee) + (affected systems × system-change cost band S/M/L) — all unit costs configurable per institution |
| 8 | Required time | critical path over the implementation plan; default effort heuristics per task type, configurable |

**Impact Score (0–100):** weighted normalization of variables 1–6:
`Score = Σ wᵢ · normalize(vᵢ)` with default weights (configurable): gaps 30%, departments 15%, employees 10%, systems 15%, risk level 20%, procedures 10%. Score bands: 0–25 Low (منخفض) · 26–50 Medium (متوسط) · 51–75 High (مرتفع) · 76–100 Critical (حرج).

**Implementation plan generation:** every `gap`/`partial` mapping → task {title, department owner, priority = f(risk, deadline in regulation), effort estimate, suggested due date}. Grouped by department, exportable, and (post-MVP) pushable to Jira/Planner.

---

## 6. MVP Scope (Hackathon Build)

### In scope — must ship
- [ ] Upload flow: 1 regulation + N policy documents + org structure CSV + risk register CSV
- [ ] Stage 1 extraction with editable requirements table
- [ ] Stage 2 hybrid mapping with confidence scores and 3-tier triage (auto / review / gap)
- [ ] Human review screen: analyst approves/rejects/re-links each medium-confidence mapping
- [ ] Stage 3 scoring engine + Impact Score gauge
- [ ] Dashboard: score, gap map (Sankey or matrix: requirements × policies, colored full/partial/none), affected-departments panel, 8-variable summary cards
- [ ] Implementation plan view grouped by department
- [ ] PDF executive report export (Arabic UI-first, English toggle)
- [ ] Full traceability: click any result → see source clause + source policy section side-by-side
- [ ] **What-if simulator** — sliders for key assumptions (training cost, deferred gaps, team capacity) recompute score/cost/time live (this is what makes it a *simulation* platform)
- [ ] **Deadline feasibility indicator** — extracted regulatory deadline vs. required implementation time → green/red banner
- [ ] **Chat with the regulation (Arabic)** — RAG Q&A over the regulation + analysis results, every answer cites the article number

### Out of scope for MVP (say it on stage as roadmap)
- Live regulatory feed / horizon scanning (monitoring SAMA/CMA sites for new circulars)
- Multi-regulation portfolio view; regulation-vs-regulation conflict detection
- Jira/ServiceNow/Teams integrations
- Fine-tuned Arabic legal model (MVP uses prompted frontier LLM)
- Multi-tenant SaaS billing

### Demo scenario (prepare this dataset before demo day)
Use a real public SAMA/NCA document (e.g., NCA Essential Cybersecurity Controls or a SAMA counter-fraud framework) + a fabricated fictional bank ("بنك النموذج") with ~15 sample policies, 8 departments with headcounts, and a 20-item risk register. Live demo: upload → 120 extracted requirements → mapping runs → Impact Score 68/100 → "12 policies need amendment, Cybersecurity + Marketing affected, est. SAR 2.4M, 14 weeks" → analyst reviews 3 borderline mappings live (shows the 80/20 story) → export report.

---

## 7. System Architecture (Reference)

```
┌────────────┐   ┌──────────────────────────────────────────────┐
│  Frontend  │──▶│                Backend API                    │
│ React+RTL  │   │  FastAPI (Python)                             │
└────────────┘   │                                               │
                 │  ┌─────────────┐  ┌───────────────────────┐  │
                 │  │ Ingestion    │  │ Mapping Engine        │  │
                 │  │ svc          │  │ hybrid retrieval +    │  │
                 │  │ parse/OCR/   │──▶ LLM judge +           │  │
                 │  │ chunk/embed  │  │ confidence triage     │  │
                 │  └─────────────┘  └───────────┬───────────┘  │
                 │  ┌─────────────┐  ┌───────────▼───────────┐  │
                 │  │ Scoring      │◀─│ Review queue (human)  │  │
                 │  │ engine       │  └───────────────────────┘  │
                 │  │ (determin.)  │                              │
                 │  └─────────────┘                              │
                 └───────┬───────────────┬──────────────────────┘
                    Postgres          Vector DB (pgvector/Qdrant)
                    (docs, reqs,      (policy & procedure chunks)
                     mappings, tasks)
                 LLM: frontier model API (extraction + judging)
                 Embeddings: multilingual (Arabic-capable)
```

**Stack recommendation (hackathon-speed):** React + Tailwind (RTL support), FastAPI, PostgreSQL + pgvector, Claude/GPT API for extraction & judging, `unstructured`/`PyMuPDF` for parsing, Tesseract-ara fallback OCR. Everything containerized; single VM deploy.

**Non-functional requirements:**
- **Traceability:** every AI decision stores {model, prompt version, source chunk IDs, confidence, timestamp} — this is your regulated-environment story.
- **Data residency:** design for KSA-hosted deployment (on-prem/STC Cloud/AWS Bahrain-KSA regions) — say this in the pitch; banks will ask first.
- **Performance target:** full pipeline on a 100-page regulation + 15 policies ≤ 5 minutes.
- **Language:** Arabic-first UI (RTL), bilingual outputs.
- **Security (post-MVP):** SSO, role-based access (CCO vs analyst vs dept head), audit log, encryption at rest.

---

## 8. Data Model (Core Entities)

- `Regulation` (id, title, regulator, issue_date, file)
- `Requirement` (id, regulation_id, article_ref, text_ar, summary_en, type, status)
- `InternalDoc` (id, kind: policy|procedure, department_id, file, version)
- `Chunk` (id, doc_id, text, embedding)
- `Mapping` (id, requirement_id, chunk_id, relation: covered|partial|gap, confidence, rationale, review_status: auto|approved|rejected, reviewer_id)
- `Department` (id, name, headcount)
- `System` (id, name, department_id, change_cost_band)
- `RiskItem` (id, title, severity, linked_requirement_ids)
- `Task` (id, mapping_id, department_id, priority, effort_days, due_date, status)
- `Assessment` (id, regulation_id, impact_score, variables_json, created_at)

---

## 9. Key Screens

1. **New Assessment wizard** — upload regulation → upload/select internal corpus → confirm org structure → run
2. **Requirements table** — extracted requirements, editable, type badges
3. **Review queue** — side-by-side requirement vs. policy clause, Approve / Reject / Re-link buttons, confidence bar (this screen wins the demo)
4. **Impact Dashboard** — score gauge, 8 variable cards, gap Sankey/heatmap, affected departments list
5. **Implementation Plan** — kanban/table by department, export
6. **Report export** — branded PDF, AR/EN

---

## 10. Success Metrics

| Metric | Target |
|---|---|
| Analysis time vs. manual baseline | Weeks → < 1 hour (incl. human review) |
| Mapping precision on demo dataset (human-validated) | ≥ 85% of auto-accepted mappings correct |
| % of mappings auto-resolved (the "80") | ≥ 70% at launch, 80%+ as prompts improve |
| Requirements extraction recall | ≥ 90% of requirements captured vs. manual read |
| Demo pipeline runtime | ≤ 5 min end-to-end |

---

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Arabic legal text quality (extraction/embedding errors) | Hybrid keyword+semantic retrieval; human review tier; test on real SAMA docs early |
| Hallucinated mappings | LLM must cite chunk IDs; rationale required; deterministic scoring layer on top; confidence thresholds |
| Cost/time estimates seen as inaccurate | Frame as configurable model with institution's own unit costs; show formula transparently ("estimate, not quote") |
| Banks won't upload policies to cloud | Data-residency + on-prem story in pitch; MVP uses fictional bank data |
| Judges say "4CRisk/Regology exist" | Counter: Arabic-first, SAMA/CMA/NCA/PDPL-native, impact **simulation** (cost/headcount/time) not just gap analysis, KSA data residency, fraction of $1,700/user/mo pricing |

---

## 12. Roadmap After Hackathon

1. **Phase 2 — Horizon scanning:** auto-monitor SAMA/CMA/NCA portals; alert when new circular drops, with instant pre-assessment.
2. **Phase 3 — Continuous compliance:** policies live in the platform; every policy edit re-scores coverage in real time (traceability map always fresh).
3. **Phase 4 — Regulator-side offering (RegTech → SupTech):** let SAMA/CMA themselves simulate sector-wide impact before issuing a regulation — aligns with the Saudi Digital Government Authority's published interest in RegTech for government.
4. Fine-tuned Arabic regulatory language model; benchmark dataset of KSA regulations.
