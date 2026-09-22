# ImplementationPlan — Build Order, Phases & Team Practices
### SIH26069 — National Weather Big Data Analytics Platform
### See `PRD.md` for document index. This file defines WHEN to build each piece and in what order. See `Guardrails.md` for the detailed Never Do / Always Do rules referenced throughout, and `tracker.md` for the living checklist that tracks actual progress against this plan.

**Tagging:** [PHASE 1]–[PHASE 4] match the scaling stages defined in `TechSpec.md` §10. Do not start a stage before its Definition of Done for the previous stage is met — this is a strict rule, not a suggestion, for a two-person team where parallel half-finished work compounds into confusion fast.

---

## 1. Team Structure (2 people)

Split by **pipeline stage**, not by module — both the Public and Admin modules (`AppFlow.md`) share the same underlying ingestion/processing/database layer, so splitting by module would have both people touching the same shared pipeline code simultaneously. Splitting by stage keeps each person's working surface mostly independent.

| Person | Owns | Touches |
|---|---|---|
| **Person A — Pipeline** | Ingestion, Kafka/Flink, database schema/migrations | `services/ingestion/`, `services/stream-processing/`, `db/migrations/` |
| **Person B — Intelligence & Interface** | ML model-serving services, API layer, both frontend modules | `services/ml-services/`, `services/api/`, `services/frontend/` |

**Shared contract, non-negotiable:** the exact shape of `reports` and `events` (`Schema.md`) is the interface between the two halves. Any change to these tables must be agreed between both people before implementation, not discovered after the fact via a broken integration.

**Daily sync (short, 10–15 min):** confirm the shared contract hasn't silently drifted, and flag anything blocking the other person's work.

---

## 2. Build Order & Definition of Done

Each stage lists what to build and the concrete, verifiable condition that must be true before moving to the next stage — verified against the actually running system, per the verification standard in `Guardrails.md`, not just "the code compiles."

### Stage 1 — Foundation [PHASE 1]
**Build:**
- `reports` and `events` tables (`Schema.md` §2.1–2.2), PostGIS + pgvector extensions enabled
- One working source: citizen report submission (Public module, `AppFlow.md` §1, `Design.md` §2.1) via the API directly into `reports`
- Basic API read endpoint to confirm data round-trips correctly

**Definition of Done:** submit one real test report through the actual submission form (not a raw DB insert), confirm it's correctly stored with every required field populated, query it back via the API.

### Stage 2 — Multi-Source Ingestion + Messaging [PHASE 1]
**Build:**
- Kafka topics (`TechSpec.md` §4.1) — modest partition counts, single broker
- 1–2 additional real source adapters (e.g., a public weather API, one disaster-alert RSS feed)
- Flink enrichment job skeleton, consuming Kafka, writing basic normalized data to `reports`

**Definition of Done:** publish one event from each real adapter, confirm each lands correctly in `reports` via the real Kafka→Flink path, not a shortcut.

### Stage 3 — Classification + Priority Routing [PHASE 1]
**Build:**
- Phase 1 lightweight classifier + rule-based fallback (`MLSpec.md` §1), as its own model-serving service
- Priority pre-triage + dedicated priority Kafka topic + fast-path consumer (`TechSpec.md` §4.3)

**Definition of Done:** a test report with high/extreme severity is written to the database via the fast path in under 1 second; a routine report goes through the normal path and still gets classified correctly. Verify both, don't assume the fast path works just because the code was written.

### Stage 4 — Credibility Scoring + Duplicate Detection [PHASE 1]
**Build:**
- Phase 1 hand-weighted credibility formula (`MLSpec.md` §3), as its own model-serving service, called from the enrichment job
- Phase 1 lexical duplicate detection (`MLSpec.md` §2)
- Event merging logic: matching a new report to an existing `event` vs. creating a new one

**Definition of Done:** `credibility_reasons` is populated (non-empty) on every scored event, verified by direct query — not just "the formula exists somewhere in the code." Two clearly-duplicate test reports (same wording, close in time/location) correctly merge into one `event`.

### Stage 5 — Admin Module: Verification [PHASE 1, auth still deferred]
**Build:**
- Verification Queue screen (`Design.md` §2.2) and its API endpoints
- Verify/Needs Review/Suspicious/Duplicate/Reject actions, writing to `verification_log`
- Temporary no-auth gate (a simple flag or basic-auth placeholder — not real role-based access yet, that's Stage 8)

**Definition of Done:** an analyst-role tester can review a real pending event, see its full evidence bundle (per `AppFlow.md` §3), take an action, and see it correctly reflected in `events.verification_status` and logged in `verification_log`.

### Stage 6 — Real-Time Dashboard [PHASE 1]
**Build:**
- Public module Live Dashboard (`Design.md` §2.1) — map, filterable list
- Real-time delivery via SSE, triggered by genuine change-notification (`TechSpec.md` §6), not a blind poll loop
- Explainability surfaced in the UI (`CredibilityPanel` component, `Design.md` §4) — never populated API data left unrendered

**Definition of Done:** insert/verify one test event, confirm it appears on the live dashboard within a few seconds without a manual refresh, with its credibility reasoning visibly shown.

### Stage 7 — Semantic Duplicate Detection [PHASE 2]
**Build:**
- Embedding model-serving service (`MLSpec.md` §2 Phase 2+), pgvector integration
- Semantic signal added to the duplicate-detection score, lexical detection retained as automatic fallback

**Definition of Done:** two differently-worded real-test descriptions of the same event correctly merge via the semantic signal alone (i.e., they would NOT have merged on lexical similarity alone) — confirm this distinction explicitly, don't just confirm "it merged."

### Stage 8 — Authentication [PHASE 2]
**Build:**
- Real auth for the Admin module (`SecurityCompliance.md` §auth design), replacing the Stage 5 placeholder
- `users` table enforcement (`Schema.md` §2.5)

**Definition of Done:** an unauthenticated request to any verification-action endpoint is rejected; a real analyst/admin login works end to end.

### Stage 9+ — Scale & Advanced ML [PHASE 3–4]
Follow `TechSpec.md` §10's scaling path (multi-broker Kafka, multi-task-manager Flink, Kubernetes, WebSocket delivery) and `MLSpec.md` §9's data-gated model upgrade order (Isolation Forest → fine-tuned text model → structured risk model → satellite/time-series models → learned fusion layer). Do not reorder the ML upgrade sequence based on perceived importance — it is explicitly ordered by real data-availability, per `MLSpec.md`.

---

## 3. Engineering Practices (apply throughout, not just at the end)

1. **Verify against the real running system before calling anything done** — an isolated unit test passing is necessary but not sufficient; see `Guardrails.md` for the full verification standard.
2. **Small, scoped changes** — when using an AI coding assistant, give it one stage (or a clear sub-piece of one) at a time with explicit "don't touch X" boundaries, not an open-ended "build the whole thing" instruction.
3. **Collect real data continuously from Stage 1 onward** — every real report submitted, every real analyst decision made, is future training data for the Phase 2+/3+ ML upgrades in `MLSpec.md`. Don't treat data collection as something to start later.
4. **Keep the shared contract (`reports`/`events` shape) changes rare and agreed** — a schema change mid-stage, made unilaterally by one person, is the most likely source of the other person's work silently breaking.
5. **Tag discipline** — when adding anything new, mark it [PHASE 1]/[PHASE 2+]/etc. immediately, and never describe a [PHASE 2+] item as working until it's been verified and its tag updated.

---

## 4. What "Start Low, Scale Later" Means Concretely Here

- Infrastructure: single Kafka broker, single Flink task manager, Docker Compose on one machine — genuinely sufficient through Stage 8. Do not introduce Kubernetes, multi-broker Kafka, or a dedicated vector database before Stage 9's real scaling need is evidenced, not assumed.
- ML: Phase 1's lightweight classifier and hand-weighted credibility formula are the correct, real components for Stages 1–8 — not placeholders to feel embarrassed about. They get upgraded exactly once their Phase 2+/3+ data requirement (`MLSpec.md` §9) is genuinely met, never before.
- Team: this plan assumes 2 people through Stage 8. Stage 9+'s full production scope (per `TechSpec.md` and `MLSpec.md`'s complete roadmap) realistically needs the team to grow — plan for that conversation once Stage 8 is solid, not before.
