# Tracker — Stage-by-Stage Progress Checklist
### SIH26069 — National Weather Big Data Analytics Platform
### See `ImplementationPlan.md` for the full reasoning behind each stage; this file is the living, literally-updatable checklist. Check items off as they're genuinely verified (per §0 below), not as they're merely written.

**How to use this file:** update it as you go, not retroactively at the end of a work session. A checked item means "verified against the real running system," not "code written." If a check later turns out to be wrong (e.g., something breaks after being marked done), uncheck it and note why in the Issues Log (§10) rather than silently re-checking it once fixed.

---

## 0. Definition of "Done" for Any Checkbox in This File

Before checking anything off, confirm:
- [ ] Tested against the actually running system (not just a passing isolated unit test)
- [ ] Tagged correctly as [PHASE 1]/[PHASE 2+]/etc. in the relevant spec file, if this introduces something new
- [ ] Verification method noted below the item (what command/action was used to confirm it)

---

## 1. Stage 1 — Foundation [PHASE 1]

- [ ] `reports` table created with all required fields (`Schema.md` §2.1)
- [ ] `events` table created (`Schema.md` §2.2)
- [ ] PostGIS extension enabled and verified
- [ ] pgvector extension enabled and verified (or confirmed to gracefully no-op if unavailable in current environment)
- [ ] Citizen report submission form (basic version) working end-to-end
- [ ] API read endpoint returns a submitted report correctly

**Stage 1 Definition of Done check:** submitted one real test report through the actual form, confirmed all fields populated correctly via API query — Y/N: ___

---

## 2. Stage 2 — Multi-Source Ingestion + Messaging [PHASE 1]

- [ ] Kafka topics created (`TechSpec.md` §4.1)
- [ ] Source adapter #1 live and verified (name: ______________)
- [ ] Source adapter #2 live and verified (name: ______________)
- [ ] Flink enrichment job skeleton consuming from Kafka
- [ ] End-to-end: adapter → Kafka → Flink → `reports` table confirmed working

**Stage 2 Definition of Done check:** published one event from each adapter, confirmed each lands in `reports` via the real path — Y/N: ___

---

## 3. Stage 3 — Classification + Priority Routing [PHASE 1]

- [ ] Lightweight classifier service built (`MLSpec.md` §1)
- [ ] Rule-based fallback classifier built, multilingual keyword coverage included
- [ ] Priority pre-triage function implemented (<5ms, verified with actual timing)
- [ ] Dedicated priority Kafka topic created
- [ ] Fast-path consumer built and verified

**Stage 3 Definition of Done check:** high/extreme severity test event written via fast path in <1s — measured time: _______. Routine event classified correctly via normal path — Y/N: ___

---

## 4. Stage 4 — Credibility Scoring + Duplicate Detection [PHASE 1]

- [ ] Credibility scoring service built (`MLSpec.md` §3 formula)
- [ ] `credibility_reasons` verified non-empty on real scored events (not just schema-capable of storing it)
- [ ] Lexical duplicate detection built (`MLSpec.md` §2)
- [ ] Event merge/creation logic built (report → existing event match, or new event creation)

**Stage 4 Definition of Done check:** queried a real scored event, confirmed `credibility_reasons` populated — Y/N: ___. Two duplicate test reports correctly merged into one event — Y/N: ___

---

## 5. Stage 5 — Admin Module: Verification [PHASE 1, auth deferred]

- [ ] Verification Queue screen built (`Design.md` §2.2)
- [ ] Queue sorted by priority/aging, not raw recency — verified with a test case where an older lower-priority item and a newer higher-priority item are both present
- [ ] Verify / Needs Review / Suspicious / Duplicate / Reject actions all functional
- [ ] Note required for Suspicious/Reject actions, enforced in UI
- [ ] `verification_log` correctly recording each action

**Stage 5 Definition of Done check:** reviewed one real pending event end-to-end, action correctly reflected in `events.verification_status` and `verification_log` — Y/N: ___

---

## 6. Stage 6 — Real-Time Dashboard [PHASE 1]

- [ ] Live Dashboard screen built (`Design.md` §2.1)
- [ ] Map + list view both functional
- [ ] Filters (date, category, severity, location, status) functional
- [ ] Real-time delivery working, confirmed change-notification-triggered (not blind polling) per `TechSpec.md` §6
- [ ] `CredibilityPanel` / reasoning visibly rendered on the dashboard, not just present in the API response
- [ ] Live/disconnected indicator states both verified (per `Design.md` §3)

**Stage 6 Definition of Done check:** new/verified event appeared on dashboard within a few seconds, no manual refresh, reasoning visible — Y/N: ___

---

## 7. Stage 7 — Semantic Duplicate Detection [PHASE 2]

- [ ] Embedding model-serving service built
- [ ] pgvector similarity query wired into the actual duplicate-detection code path (not left uncalled — verify with a grep/code check, not just "the function exists")
- [ ] Fallback to lexical-only confirmed working if embedding service is stopped/unavailable

**Stage 7 Definition of Done check:** two differently-worded real descriptions of the same event merged via semantic signal alone (confirmed they would NOT have merged on lexical similarity alone) — Y/N: ___

---

## 8. Stage 8 — Authentication [PHASE 2]

- [ ] `users` table populated with real accounts
- [ ] Login flow functional for Admin module
- [ ] Unauthenticated requests to verification-action endpoints correctly rejected (tested, not assumed)
- [ ] Stage 5's temporary no-auth gate fully removed

**Stage 8 Definition of Done check:** confirmed an unauthenticated API call is rejected (exact test performed: ______________) — Y/N: ___

---

## 9. Stage 9+ — Scale & Advanced ML [PHASE 3–4]

Track against `TechSpec.md` §10 and `MLSpec.md` §9's data-gated order. Add rows here as each is actually started (don't pre-fill this section speculatively):

| Item | Data requirement met? | Status |
|---|---|---|
| Isolation Forest (anomaly detection) | | Not started / In progress / Verified |
| Fine-tuned XLM-R/MuRIL classification | | Not started / In progress / Verified |
| Structured-data XGBoost/LightGBM risk scoring | | Not started / In progress / Verified |
| Swin Transformer (satellite imagery) | MOSDAC access status: _______ | Not started / In progress / Verified |
| LSTM/Transformer time-series | | Not started / In progress / Verified |
| XGBoost evidence-fusion layer | Real analyst decisions accumulated: _______ | Not started / In progress / Verified |
| Multi-broker Kafka | | Not started / In progress / Verified |
| Multi-task-manager Flink | | Not started / In progress / Verified |
| Kubernetes migration | | Not started / In progress / Verified |
| WebSocket delivery (replacing SSE) | | Not started / In progress / Verified |

---

## 10. Issues Log (running list, add as discovered)

| Date | Issue | Stage | Resolution |
|---|---|---|---|
| | | | |

---

## 11. Open Questions Tracker (mirrors `PRD.md` §9 — update here as each is resolved)

- [ ] Formal SLA targets for latency/uptime defined
- [ ] Data retention period for citizen-submitted reports/media decided
- [ ] Real-time delivery long-term decision (stay SSE vs. move to WebSocket) confirmed with real load data
- [ ] Exact analyst/admin role-granularity decision (single shared admin vs. individual roles) — currently: single shared admin login per `AppFlow.md` §4

---

## 12. Pre-Milestone / Pre-Demo Checklist (run this before showing the system to anyone outside the team)

- [ ] Fresh environment rebuild performed (not relying on an accumulated dev-session state)
- [ ] No test/placeholder data visible that isn't clearly explainable if asked about
- [ ] One full end-to-end rehearsal performed: submit a normal report, submit a high-priority report, confirm priority ordering is visibly correct, confirm both appear live on the dashboard with reasoning shown
- [ ] Every score/status shown during the rehearsal has visible, sensible reasoning attached — spot-checked, not assumed
