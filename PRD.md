# PRD — Product Requirements Document
### SIH26069 — National Weather Big Data Analytics Platform
### This is the entry-point document. Read this first for WHY the product exists and WHAT it must do; other files cover HOW.

---

## 0. Document Set (for orientation)

| File | Covers |
|---|---|
| `PRD.md` | *(this file)* — product vision, users, requirements, scope |
| `TechSpec.md` | Architecture, API, Kafka, deployment |
| `AppFlow.md` | User flows per role |
| `Design.md` | Screens, UI states, component inventory |
| `Schema.md` | Data schema reference |
| `MLSpec.md` | AI/ML — classification, dedup, credibility, embeddings, multimodal roadmap |
| `ImplementationPlan.md` | Build order, phases, team practices |
| `tracker.md` | Stage-by-stage checklist / progress tracker |
| `Guardrails.md` | Never Do / Always Do rules |
| `SecurityCompliance.md` | Auth, privacy, backup, monitoring |

**Tagging convention used across ALL these documents:** every requirement/feature is marked **[BUILT]** (verified working in the current prototype) or **[ROADMAP]** (planned, not yet implemented). Never treat a [ROADMAP] item as delivered.

---

## 1. Problem Statement

India needs a way to detect and verify weather-related disaster events (floods, cyclones, heatwaves, storms, etc.) faster than official channels alone can provide, by combining citizen reports, social media, news, and official APIs into one trustworthy source — while filtering out fake, duplicate, or unverified information.

**Official problem statement (SIH26069, Ministry of Earth Sciences):** Design and develop a scalable National Weather Big Data Analytics Platform capable of collecting and processing real-time weather-related information for India from multiple internet-based sources including social media platforms, public datasets, websites, APIs, and citizen reports. The platform should automatically collect weather-related posts tagged with #IMD and other relevant weather hashtags, along with metadata (date/time, city, state, GPS location, photos, videos, event category), and store this in a centralized database. The system should leverage big data technologies for large-scale real-time ingestion, processing, storage, and visualization, and use ML/AI to identify fake or misleading reports, verify untrusted sources, remove duplicate entries, and automatically categorize weather events. A web dashboard and Admin Panel must support date-wise, event-wise, and location-wise filtering, verification status tracking, and real-time visualization.

**The core hard problem, restated simply:** anyone can post "flood in Mumbai" — the platform's real value is not collecting reports, it's deciding which ones to trust, merging duplicates of the same real event, and surfacing that trustworthy picture fast enough to matter during an actual disaster.

---

## 2. Target Users / Roles

| Role | Who | Primary need |
|---|---|---|
| **Citizen** | General public in an affected area | Submit a weather/disaster report quickly, with location and optional photo/video |
| **Public Viewer** | Anyone checking conditions in their area (may overlap with Citizen) | See a trustworthy, live, filterable view of current weather events near them |
| **Analyst** | Disaster-management staff, government/NGO reviewers | Review flagged/pending events, verify or reject them, see the evidence/reasoning behind each score |
| **Admin** | Platform/system operators | Manage sources, monitor system health, manage analyst access (once auth exists — see `SecurityCompliance.md`) |

Detailed step-by-step flows per role are in `AppFlow.md`. Screen-level detail for each role is in `Design.md`.

---

## 3. Goals

1. **Speed:** a real, credible weather event should be visible on the dashboard within seconds to a few minutes of the first report — not hours.
2. **Trust:** every event shown must have a visible, explainable reason for its credibility/verification status — never an unexplained black-box score.
3. **Coverage:** ingest from multiple source types simultaneously (citizen, social, news/RSS, official weather APIs, government datasets) without any one source type being a single point of failure.
4. **Accuracy without overclaiming:** classification and credibility scoring must be evaluated honestly — a suspiciously perfect result is treated as a bug signal, not a success (see `MLSpec.md` and `Guardrails.md`).
5. **Resilience under load:** an urgent/critical report must never be meaningfully delayed behind a backlog of routine reports.
6. **Multilingual reach:** the platform must work for Hindi/Marathi/Hinglish content, not English-only, given the real-world source of Indian citizen reports.

---

## 4. Success Metrics [ROADMAP — targets to be validated with real usage/deployment stakeholder, not invented in isolation]

| Metric | Current prototype status |
|---|---|
| End-to-end latency (report → visible on dashboard) | [BUILT] measured and displayed on dashboard per event; no formal target SLA defined yet |
| Critical-event latency vs normal-event latency | [BUILT] critical path verified faster in manual tests; not yet load-tested at volume (see `tracker.md`) |
| Classification accuracy | [BUILT] ~99% test accuracy on current real+synthetic training mix (honest number, not 100% — see `MLSpec.md`) |
| Duplicate detection recall (catching reworded duplicates) | [BUILT] semantic matching verified on manual paraphrase test cases; not yet measured at scale |
| System uptime / reliability | [ROADMAP] — no formal SLA yet, single-node prototype |
| Real user adoption / reports submitted per day | [ROADMAP] — not yet deployed to real users |

---

## 5. Scope

### 5.1 In scope — MVP / Current Prototype [BUILT]
- Multi-source ingestion (weather API, RSS/disaster-alert feeds, citizen report submission, simulated social feed)
- Automatic event classification into a fixed taxonomy (see `Schema.md`)
- Duplicate/near-duplicate detection (lexical + semantic)
- Explainable credibility scoring
- Priority routing so urgent events aren't delayed behind routine ones
- Live dashboard with filtering, verification center for analyst review
- Full details in `TechSpec.md` and `MLSpec.md`

### 5.2 In scope — Near-term Roadmap [ROADMAP]
- Real government data source integration (IMD, MOSDAC, expanded NDMA coverage) replacing current simulated/limited sources
- Authentication and role-based access (currently all endpoints open — see `SecurityCompliance.md`)
- Multi-broker/clustered infrastructure for real production load

### 5.3 In scope — Longer-term Production Vision [ROADMAP]
- Fine-tuned multilingual NLP (XLM-R/MuRIL), satellite imagery analysis (Swin Transformer on MOSDAC data), structured-data anomaly detection, time-series storm prediction, and a learned evidence-fusion model — full detail in `MLSpec.md`
- Kubernetes-based scaling, Flink-based true real-time stream processing — full detail in `TechSpec.md`

### 5.4 Explicitly Out of Scope (for now)
- Predictive disaster modeling beyond basic trend/time-series signals (this is a detection/verification platform first, a forecasting platform second)
- Direct emergency dispatch/response coordination (this platform informs responders, it does not replace dispatch systems)
- Non-weather-related citizen reporting (crime, infrastructure complaints, etc.) — strictly weather/disaster-event scoped

---

## 6. Functional Requirements (mapped to the official PS)

| PS requirement | How it's addressed | Status |
|---|---|---|
| Collect from social media, public datasets, websites, APIs, citizen reports | Source adapter architecture, one adapter per source type | [BUILT] (some sources real/live, some simulated pending API access — see `TechSpec.md`) |
| Posts tagged #IMD and weather hashtags | `hashtags TEXT[]` field in schema; real hashtag-based ingestion (e.g., from X/Twitter) is [ROADMAP] pending API access decision | [BUILT schema] / [ROADMAP ingestion] |
| Metadata: date/time, city, state, GPS, photos, videos, event category | Present in canonical event schema | [BUILT] — see `Schema.md` |
| Centralized database | PostgreSQL + PostGIS | [BUILT] |
| Big data tech for real-time ingestion/processing/storage/visualization | Kafka + Spark Structured Streaming | [BUILT] prototype scale; [ROADMAP] production scale (Flink, multi-broker) |
| ML to detect fake/misleading reports, verify untrusted sources, dedupe, auto-categorize | Classification, credibility scoring, dedup — see `MLSpec.md` | [BUILT] baseline; [ROADMAP] advanced multimodal |
| Web dashboard + Admin Panel with date/event/location filters, verification status tracking, real-time visualization | React dashboard, Verification Center, live SSE updates | [BUILT] |

---

## 7. Non-Functional Requirements

1. **Explainability:** every automated score (credibility, classification confidence) must be accompanied by human-readable reasoning, visible in the UI — not just a number.
2. **Graceful degradation:** every advanced component (embeddings, semantic search, advanced ML) must have a working fallback if unavailable — the system must never hard-fail due to one optional component's absence. This is a hard architectural constraint, detailed in `Guardrails.md`.
3. **Multilingual support:** text-processing components must handle Hindi/Marathi/Hinglish content, not English-only.
4. **Honest metrics:** no reported accuracy/performance number should be accepted at face value without investigating suspiciously perfect results — see `Guardrails.md` and `MLSpec.md`.
5. **Auditability:** every verification decision (human or automated) must be logged with who/what made the decision and why.
6. **Data privacy:** citizen-submitted personal/location data handled per DPDP Act considerations — see `SecurityCompliance.md` [ROADMAP].

---

## 8. Assumptions & Dependencies

- **MOSDAC (ISRO) data access** — registration/approval process assumed to take time; satellite-imagery-dependent features are blocked on this.
- **Real government API access** (IMD, expanded NDMA, data.gov.in structured feeds) — currently simulated/limited pending formal access; treat any claim of "live government data" as [ROADMAP] until an actual credentialed connection is verified.
- **Social media API access** (e.g., for real #IMD hashtag tracking) — currently simulated; real access may involve cost/rate-limit constraints that affect design (see `TechSpec.md`).
- **Team capacity** — current build assumes a small team (see `ImplementationPlan.md` for team-size-appropriate phasing); production-scale roadmap items assume the team/resources will grow accordingly, not that the current team executes all of it unassisted.

---

## 9. Open Questions (track resolution in `tracker.md`)

- Exact authentication/role model details (who grants analyst/admin roles, and how) — placeholder design exists in `SecurityCompliance.md`, not finalized.
- Formal SLA targets for latency/uptime — not yet defined with a real deployment stakeholder.
- Data retention period for citizen-submitted reports/media — not yet decided.
- Whether real-time delivery upgrades to WebSocket or stays SSE-based long-term — currently SSE (poll-based internally), evaluate once real load data exists.
