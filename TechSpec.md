# TechSpec — Architecture, API, Kafka & Deployment
### SIH26069 — National Weather Big Data Analytics Platform
### See `PRD.md` for product context and document index. This file defines the fresh, chosen technical design — not a description of any prior prototype's specific implementation.

**Tagging in this file:** **[PHASE 1]** = build this first, runs on modest infrastructure, is the real target design (not a throwaway prototype to later replace) — just deployed at small scale initially. **[PHASE 2+]** = the same architecture scaled up, added once real usage justifies it. Nothing here is a "toy version to rewrite later" — Phase 1 is a smaller deployment of the exact same design as Phase 2+, per the scaling philosophy in `ImplementationPlan.md`.

---

## 1. Guiding Architectural Principles

These principles govern every design decision below:

1. **Graceful degradation** — every advanced component must have a working fallback if unavailable. No optional component's failure may crash the core pipeline.
2. **Explainability by default** — every automated score/decision must carry a human-readable reason, end to end, from the scoring function to the UI.
3. **Priority is structural, not cosmetic** — an urgent event must be architecturally incapable of being stuck behind a backlog of routine events, not just labeled "urgent" while sharing the same queue.
4. **One canonical event shape** — every source adapter normalizes into the same schema (`Schema.md`) before entering the pipeline; no downstream component ever special-cases by source.
5. **Design once, scale by configuration** — Phase 1 and Phase 2+ use the same architecture; scaling changes topology/replica counts, not component choices.
6. **Backpressure and lag are first-class, not diagnostic afterthoughts** — every queue/consumer pairing in the pipeline exposes lag as a metric from Phase 1 onward (§4.4, §9), so a slow consumer is visible before it becomes an outage, not discovered after.

---

## 2. Chosen Technology Stack

| Layer | Technology | Why |
|---|---|---|
| Message Queue | **Apache Kafka** | Industry-standard durability and ecosystem maturity for high-stakes ingestion; single broker at Phase 1, multi-broker cluster at Phase 2+ |
| Stream Processing | **Apache Flink** | True event-at-a-time processing, not micro-batching — removes the batch-window delay that a Spark-style architecture would introduce; this matters directly for a system whose core value proposition is speed |
| Database | **PostgreSQL + PostGIS** | Relational integrity + first-class geospatial querying in one system |
| Vector/Semantic Search | **pgvector** (Phase 1) → **dedicated vector DB (Milvus/Qdrant)** (Phase 2+, **decision is benchmark-driven, not volume-guessed** — see §8.2) | Avoid operating a separate vector database until measured query latency/throughput on pgvector actually justifies the operational cost, not a headcount-of-embeddings guess |
| ML Serving | **Dedicated model-serving service(s)**, decoupled from the stream processor, **each scaled independently of the Flink job** (§5, §10) | Prevents cross-service dependency/version conflicts — a lesson worth keeping regardless of stack, detailed in `Guardrails.md`; inference is a different compute profile (CPU/GPU-bound, bursty) from stream processing and must scale on its own axis |
| API Layer | **FastAPI** | Async-native, strong typing, pairs well with both REST and streaming endpoints |
| Real-time delivery | **Server-Sent Events (SSE)**, fed by a Kafka-backed changefeed (§6), for both Phase 1 **and** Phase 2+ | SSE is simpler to build and operate correctly with a small team, and remains genuinely real-time (not polling-disguised) when driven by a changefeed rather than a blind DB poll; **do not switch to WebSocket purely because of scale** — a Kafka-fed SSE stream scales the same way a WebSocket gateway would. Move to WebSocket only when the product genuinely needs client→server push or sub-second bidirectional messaging, which this dashboard does not currently require |
| Frontend | **React + Vite**, Tailwind, a geospatial map library (e.g., Leaflet/Mapbox) | Standard, well-supported choice for a data-dense live dashboard |
| Containerization | **Docker** (Phase 1) → **Kubernetes** (Phase 2+) | Docker Compose is sufficient for single-machine Phase 1; Kubernetes only once multi-machine orchestration/auto-scaling is genuinely needed |
| Observability | **OpenTelemetry** (tracing + metrics) across ingestion, Flink, API, and model-serving, from Phase 1 | A score/decision's reasoning must be traceable end-to-end (principle 2); OpenTelemetry gives one instrumentation standard across every service rather than a bespoke logging approach per component — see §9 |
| ML — text, structured, vision, time-series, fusion | See `MLSpec.md` in full | Kept out of this file to avoid duplication — this file only notes where each model plugs into the architecture |

---

## 3. High-Level Architecture Flow [PHASE 1]

```
SOURCES (weather APIs, RSS/CAP disaster feeds, citizen report API, social/news feeds where access permits)
    ↓
SOURCE ADAPTERS (one per source: cursor/checkpoint-tracked pulls, source-level dedup,
                  then normalize into canonical event schema — Schema.md; see §3.1)
    ↓
LIGHTWEIGHT PRE-TRIAGE (cheap keyword/severity check, <5ms, runs BEFORE Kafka)
    ↓
    ├── urgent? → dual-write to a dedicated PRIORITY topic (separate queue, not just a flag)
    │        ↓
    │   FAST CONSUMER (lightweight, rule-based classification only — no heavy model dependency)
    │        ↓
    │   Direct write to the database (idempotent upsert, keyed by event ID)
    │        ↓
    └── normal → KAFKA (standard topics, per source type)
             ↓
        KAFKA PARTITIONS
             ↓
        FLINK JOB — VALIDATION & ENRICHMENT (true streaming, stateful — keyed on canonical
        event ID, not micro-batch and not a stateless per-event transform; see §5):
             1. Schema validation & normalization
             2. Classification (see MLSpec.md — model call, not embedded logic)
             3. Duplicate detection (lexical + semantic — MLSpec.md)
             4. Credibility / evidence-fusion scoring (MLSpec.md)
             5. Spatial-temporal clustering
             6. Reason/explanation generation
             ↓
        FLINK JOB — DATABASE WRITER:
             - matches/merges into canonical event record
             - re-scores with historical DB context
             - generates & stores embeddings
             - queries vector similarity for corroboration
             - records processing timestamps for latency tracking
             ↓
        POSTGRESQL + POSTGIS + PGVECTOR
             ↓
        FASTAPI (REST + SSE)
             ↓
        REACT DASHBOARD (progressive live updates — fast/priority write renders
                          immediately, then patches in place as full enrichment
                          completes; see §6)
```

### Historical/reference data path [PHASE 1]
```
Government historical datasets → batch ETL → columnar data lake (e.g., Parquet)
    → reference lookups for live enrichment (never blocks the real-time path)
```

---

## 3.1 Source Adapter Contract [PHASE 1]

Every source adapter (§11 — one file per source type) must satisfy two requirements before it is considered complete, neither of which was explicit in the original design:

**Cursor / checkpoint.** Each adapter persists its own cursor — the last successfully processed offset, timestamp, or record ID for that source — after every successful pull, not just at shutdown. On restart (crash, redeploy, or scheduled restart), the adapter resumes from the last committed cursor rather than re-pulling its entire lookback window or, worse, silently skipping ahead and losing a gap. The cursor store is a small keyed table (`source_id → last_cursor, updated_at`) in PostgreSQL — no separate coordination service is needed at Phase 1 scale.

**Source-level deduplication.** Before an event is handed to pre-triage, the adapter deduplicates against its own recent cursor-tracked window using a source-native identifier or content hash (e.g., a citizen-report submission ID, an RSS `<guid>`, a government feed's item ID). This is deliberately separate from, and prior to, the Flink job's lexical/semantic duplicate detection (§3, step 3): source-level dedup stops a single noisy or retrying source from flooding Kafka with literal repeats of the same raw item; Flink's downstream dedup handles the harder cross-source case (the same real-world event reported independently by two different sources).

---

## 4. Kafka Design

### 4.1 Topics [PHASE 1]
- One topic per source category (weather, citizen, social, government) — keeps producer/consumer contracts simple and lets each source type scale its own partition count independently later.
- One dedicated **priority topic** for urgent/critical events — a structurally separate queue, not a partition-based workaround (Kafka only guarantees ordering within a partition, never globally, so a backlogged partition still delays anything behind it regardless of severity — a separate topic is the only structural fix).
- A post-enrichment topic (consumed by the DB writer job).

### 4.2 Partitioning — capacity-based [PHASE 1 → PHASE 2+]
Partition counts are derived from a capacity calculation, not picked arbitrarily, at every phase:

```
partitions_needed = ceil( target_topic_throughput / achievable_throughput_per_partition )
```

- `achievable_throughput_per_partition` is measured, not assumed — benchmark a single partition under this pipeline's actual message size and the slowest consumer in its group (typically the Flink enrichment job, since it does the most per-event work), and re-measure whenever consumer logic changes materially.
- `target_topic_throughput` for Phase 1 is set from the real expected ingestion rate for the connected sources (§12 — Phase 2 government sources), with headroom (~2x) for burst periods (e.g., a disaster event spiking citizen reports and social/news volume simultaneously) rather than steady-state average alone.
- Phase 1: single broker, replication factor 1, partition count from the formula above (commonly 2-4 per topic at Phase 1 volumes, but this is a calculated output, not a fixed default).
- Phase 2+: recalculate with real measured production throughput (never re-guessed), multi-broker cluster, replication factor 3 for fault tolerance. Partition count only increases (Kafka does not support safely decreasing partitions on a live topic), so err toward the lower bound of the formula at Phase 1 rather than over-provisioning.

### 4.3 Priority dual-write contract [PHASE 1]
1. Pre-triage runs in the ingestion layer, before any Kafka produce call.
2. If urgent: event is produced to BOTH its normal topic AND the priority topic (same event ID) — this is a dual-write, not a reroute, so the event still receives full enrichment via the normal path in addition to the fast write.
3. Idempotent upsert (keyed by event ID) at the database layer ensures the fast write and the later, fuller enrichment write never conflict or duplicate — the fuller write's additional fields (embeddings, full credibility with historical context) simply fill in what the fast path didn't have time to compute.

### 4.4 Consumer lag monitoring [PHASE 1]
Every Kafka consumer group (Flink enrichment, DB writer, fast/priority consumer) exposes consumer lag — messages produced minus messages committed, per partition — as a metric from day one, not added later as a debugging afterthought:
- Exported via OpenTelemetry/Kafka's own JMX metrics into the same observability stack as everything else (§9), not a separate one-off dashboard.
- Alerting threshold defined per topic: the priority topic's lag threshold must be far tighter than standard topics, since lag there directly violates principle 3 (urgent events must not queue behind routine ones) — a growing priority-topic lag is treated as a page-worthy incident, not a background warning.
- Lag on the priority topic and its fast consumer is the single most important number to watch operationally, since it's the direct, measurable proxy for whether the "urgent events aren't stuck behind a backlog" guarantee (principle 3) is actually holding in production, not just architecturally true on paper.

---

## 5. Stream Processing (Flink) [PHASE 1]

- True event-at-a-time processing — no artificial batch-window delay between an event arriving and being processed.
- Use Flink's event-time semantics with watermarking for correctness under out-of-order arrival (a real concern with multiple independent source adapters polling at different cadences).
- Deduplication and enrichment logic should be implemented as composable Flink operators, each independently testable — avoid one monolithic processing function that's hard to unit-test in isolation.
- **Canonical event state is an explicit, first-class stateful streaming component, not an implicit side effect of the enrichment steps.** The enrichment job keys its Flink state (via `keyBy` on canonical event ID) and holds the in-progress canonical record — matched/merged fields, dedup fingerprints, running scores — in Flink-managed keyed state as it moves through steps 1-6 (§3), rather than treating each enrichment step as a stateless transform that happens to write to the same downstream row. This makes the canonical-event lifecycle itself recoverable via Flink checkpointing (below), not just the individual step outputs.
- ML model calls (classification, credibility scoring) should call OUT to a separate model-serving service (§2, §7) rather than loading model artifacts directly inside the Flink job's process — this is the architectural fix for the cross-service version-mismatch class of bug (see `Guardrails.md`), and it also means the Flink job's own dependencies stay minimal and stable.
- **ML inference scales independently of Flink parallelism.** Flink's job/operator parallelism is tuned for stream-processing throughput; the model-serving services behind it (§2, §9) are scaled on their own axis — independent inference worker replicas per model, sized against inference queue depth/latency rather than Flink task-slot count — since classification, credibility, and embedding calls are a different (often GPU- or batch-friendlier) compute profile than the surrounding stream processing. Coupling the two scaling decisions would either starve inference under Flink's tuning or over-provision Flink to compensate for slow inference.
- State/checkpointing: use Flink's built-in checkpointing for failure recovery, covering the canonical-event keyed state above — do not build a custom checkpoint mechanism.

---

## 6. Real-Time Delivery Detail

**SSE, Phase 1 and Phase 2+:** even though SSE is simpler to implement than WebSocket, it must be a genuine push, not a disguised poll pretending to be real-time. Implementation requirement: the SSE endpoint should be triggered by a change-notification mechanism (e.g., Postgres `LISTEN`/`NOTIFY`, or a lightweight subscription to the post-enrichment Kafka topic) rather than a blind fixed-interval database re-query — a blind poll loop technically "works" but is not honestly real-time and should not be described as such. **Do not treat WebSocket as the default scaling answer**: a Kafka-fed SSE stream scales along the same axis (more consumers subscribing to the same changefeed) that a WebSocket gateway would, so scale alone is not a reason to switch. Reserve WebSocket for Phase 2+ *only if* the product grows a genuine need for client→server push or sub-second bidirectional messaging — neither of which the live dashboard currently requires — at which point it becomes a Kafka-fed WebSocket gateway per the original design.

**Progressive event updates [PHASE 1, strongly recommended]:** the dashboard should not wait for full enrichment before showing an event, nor should it silently overwrite what the analyst is looking at with no indication anything changed. Concretely:
1. On the priority dual-write fast path (§4.3), the SSE stream pushes the event as soon as the fast consumer's rule-based classification and initial DB write complete — marked with a status field (e.g., `enrichment_status: "preliminary"`).
2. When the fuller Flink enrichment write lands (embeddings, historical-context re-scoring, full credibility), the same event ID is pushed again as a patch/update — `enrichment_status: "complete"` — rather than a brand-new event, so the frontend updates the existing card/marker in place instead of duplicating it.
3. Non-priority events follow the same pattern with a single terminal update once Flink enrichment completes, since there is no separate fast write for them to preview first.
This keeps the "score without its reasoning field is a bug" rule (§7) intact — a preliminary event still carries whatever reasoning the fast path produced, even if abbreviated, and the UI should visibly distinguish preliminary from complete rather than presenting both identically.

---

## 7. API Contract [PHASE 1]

```
GET  /api/v1/events                 — paginated, filterable (date, category, severity,
                                       location, verification status, priority)
GET  /api/v1/events/stream          — real-time push (SSE, progressive updates, per §6)
GET  /api/v1/events/stats
GET  /api/v1/events/map             — geospatial-formatted
GET  /api/v1/events/{id}            — full detail incl. all reasoning/explanation fields
POST /api/v1/reports/citizen        — citizen report submission
GET  /api/v1/verification           — analyst review queue
POST /api/v1/verification/{id}/...  — verify/flag/duplicate/reject actions
GET  /api/v1/system/status
```

### Design rules
- The frontend never talks directly to Kafka/Flink/the database — always through this API layer, so internal infrastructure changes never require frontend changes.
- Every response model that includes a score (credibility, classification confidence) MUST include its accompanying reasoning field, and the frontend MUST render it — a backend field existing without a corresponding visible UI element is treated as a bug, not a minor gap (see `Guardrails.md`).
- Every event object (`/events`, `/events/{id}`, and `/events/stream` payloads) carries an `enrichment_status` field (`preliminary` | `complete`, §6) so the frontend can visually distinguish a fast-path event from a fully enriched one rather than presenting both identically.
- Authentication/authorization for write endpoints (citizen submission gets light rate-limiting/anti-abuse; verification actions require analyst/admin role) — full detail in `SecurityCompliance.md`; do not ship verification-action endpoints unauthenticated beyond an initial local-only Phase 1 dev environment.

---

## 8. Database Design Notes [PHASE 1 → PHASE 2+]

### 8.1 Table partitioning [PHASE 1]
The core events table is partitioned from the first migration, not retrofitted once it gets large — retrofitting partitioning onto a live, growing table is disproportionately more painful than declaring it up front:
- Partition by time range (e.g., monthly) as the primary axis, since almost every query (dashboard windows, verification queue, stats endpoints) is time-bounded, and this also makes historical-data archival (§3, historical/reference path) a matter of detaching old partitions rather than deleting rows out of a monolithic table.
- Consider a secondary partition or index strategy by region/geography if query patterns in practice concentrate around specific states/districts — decide from observed query patterns, not upfront, same as partition counts in Kafka (§4.2).
- PostGIS and pgvector indexes are created per-partition, not once globally, so query planning stays effective as partitions grow.

### 8.2 Read replicas and the pgvector → dedicated vector DB decision [PHASE 1 → PHASE 2+]
- Phase 1: single PostgreSQL instance is sufficient; Phase 2+ adds read replicas once dashboard/API read load genuinely contends with the Flink DB-writer job's write load — route read-heavy endpoints (`/events`, `/events/map`, `/events/stats`) to replicas, keep writes and the verification queue's read-after-write paths on the primary.
- The pgvector → Milvus/Qdrant migration (§2) is **benchmark-driven**: instrument actual vector similarity query latency and embedding write throughput against pgvector in production, and only migrate once measured numbers cross a defined threshold (e.g., p95 similarity query latency degrading past an acceptable bound for the corroboration lookup in §3) — not once embedding *count* alone looks large, since raw volume doesn't reliably predict query performance on its own.

---

## 9. Observability [PHASE 1]

Instrumentation is one standard (OpenTelemetry) across every service, not a per-component logging approach bolted on separately:
- **Tracing:** a single event's journey — source adapter → pre-triage → Kafka → Flink enrichment steps → DB write → API → SSE push — is traceable end to end via a shared trace/event ID, directly supporting the explainability principle (principle 2): if a score exists, its computation path should be inspectable, not just its output value.
- **Metrics:** per-service latency and error rates, Kafka consumer lag per topic/consumer group (§4.4), Flink checkpoint duration and backpressure, model-serving inference latency and queue depth per model (feeding the independent inference-scaling decisions in §5), and SSE connection counts.
- **ML inference worker scaling signal:** inference queue depth and p95 latency per model-serving service are the metrics that drive the independent scaling described in §5 — these are watched and alerted on separately from Flink's own throughput metrics, since the two do not move together.
- Exported to a standard OpenTelemetry collector; Phase 1 can back this with a lightweight local stack (e.g., a single Prometheus + Grafana pair in `docker-compose.yml`), with the same instrumentation carrying forward unchanged into a more production-grade backend at Phase 2+ (`SecurityCompliance.md`) — the instrumentation code never has to be rewritten, only where it's shipped to.

---

## 10. Deployment Topology

### Phase 1 — Docker Compose, single machine
Services: Kafka + Zookeeper (or a KRaft-mode Kafka without Zookeeper, which is the more modern default and worth using fresh rather than the older Zookeeper-dependent setup), PostgreSQL (with PostGIS + pgvector — build or select an image that includes both from day one rather than retrofitting), Flink JobManager + TaskManager, the model-serving service(s), the API service, the frontend, and a lightweight observability stack (Prometheus + Grafana, or an OpenTelemetry-collector-backed equivalent — §9).

**Model-serving service isolation (important, carried forward as a design principle, not old-stack baggage):** every ML model (classification, credibility fusion, embeddings, and later the multimodal models in `MLSpec.md`) runs in its own service/container with its own pinned dependency versions, called over a lightweight internal API (REST/gRPC) by the Flink jobs — never imported as a library directly into the stream-processing job's own runtime. This single decision prevents an entire category of cross-service dependency conflicts. Even at Phase 1's single-machine scale, each model-serving container can be given its own replica count independent of the Flink job's parallelism (§5), so the scaling axis is established from the start rather than introduced later.

### Phase 2+ — Kubernetes
- Kafka cluster (3+ brokers), Flink cluster with multiple task managers, managed/clustered PostgreSQL (with read replicas per §8.2), horizontally-scaled API pods, auto-scaling configured on real observed load, not speculative sizing.
- **Model-serving services scale as their own deployments/HPAs**, keyed on inference queue depth/latency (§5, §9) — never tied to Flink's task-manager count.

---

## 11. Directory / Service Structure [PHASE 1 — establish this shape from the first commit]

```
/
├── docker-compose.yml                # Phase 1 orchestration
├── infra/
│   └── k8s/                          # Phase 2+ manifests, added when needed, not upfront
├── services/
│   ├── ingestion/                    # source adapters + pre-triage + Kafka producers
│   │   └── adapters/                 # one file per source type
│   ├── stream-processing/            # Flink jobs
│   │   ├── enrichment-job/
│   │   └── db-writer-job/
│   ├── ml-services/                  # one subfolder per model-serving service — see MLSpec.md
│   │   ├── classification-service/
│   │   ├── credibility-service/
│   │   ├── embedding-service/
│   │   └── (multimodal services added per MLSpec.md's build order)
│   ├── api/                          # FastAPI
│   ├── frontend/                     # React + Vite
│   └── observability/                # OpenTelemetry collector config, dashboards — §9
├── db/
│   └── migrations/                   # schema migrations — see Schema.md for the schema itself;
│                                      # includes source-adapter cursor table (§3.1) and
│                                      # events table partitioning (§8.1)
└── data/
    ├── raw/ processed/ reference/    # training and historical data — see MLSpec.md
```

**Rule:** every new file goes in the location matching this shape's pattern — do not invent parallel structures.

---

## 12. Scaling Path Summary

```
PHASE 1 — Single machine, Docker Compose, modest Kafka/Flink footprint, SSE delivery
       ↓
PHASE 2 — Real government data sources fully connected, authentication live,
          expanded source coverage
       ↓
PHASE 3 — Multi-broker Kafka, multi-task-manager Flink, Kubernetes, independent
          inference-worker scaling, SSE continues to scale (WebSocket only if a genuine
          bidirectional need emerges — §6), multimodal ML services begin (MLSpec.md)
       ↓
PHASE 4 — Full production scale: managed database cluster, dedicated vector DB,
          learned evidence-fusion model, government-grade monitoring (SecurityCompliance.md)
```
