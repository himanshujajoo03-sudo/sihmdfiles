# Schema — Data Schema Reference
### SIH26069 — National Weather Big Data Analytics Platform
### See `PRD.md` for document index, `TechSpec.md` for the architecture this schema serves, `AppFlow.md`/`Design.md` for the Public and Admin module flows that read/write these tables.

**Tagging:** [PHASE 1] = needed from the first build. [PHASE 2+] = shape defined now so no painful migration later, but not enforced/fully used until the corresponding feature ships.

**Naming note (fresh, deliberately simple):** `reports` = individual raw incoming submissions (from any source). `events` = the merged, canonical, authoritative record shown on the dashboard. This matches the product language already used in `AppFlow.md` ("submit a report") and `Design.md` ("Live Events" dashboard) — a citizen submits a **report**; the platform surfaces an **event**.

---

## 1. Fixed Taxonomies

These are domain requirements driven by the official problem statement, not implementation choices — reused consistently across every table, the classification model (`MLSpec.md`), and every screen (`Design.md`).

### 1.1 Event categories [PHASE 1]
```
rainfall, heavy_rainfall, flood, thunderstorm, lightning,
heatwave, fog, dust_storm, strong_wind, hailstorm, cyclone, other
```

### 1.2 Severity levels [PHASE 1]
```
low, moderate, high, extreme
```

### 1.3 Verification statuses [PHASE 1]
```
pending, verified, needs_review, suspicious, duplicate, rejected
```

### 1.4 Source types [PHASE 1]
```
citizen, social, news, weather_api, government, other
```

### 1.5 Admin/Analyst roles [PHASE 2+ — shape defined now, single shared admin login used at Phase 1 per `AppFlow.md` §4]
```
admin, analyst
```

---

## 2. Core Tables

### 2.1 `reports` — raw incoming submissions [PHASE 1]

One row per individual report, regardless of source.

```sql
report_id                  UUID PRIMARY KEY
source_type                 TEXT NOT NULL CHECK (source_type IN taxonomy §1.4)
source_name                 TEXT NOT NULL         -- e.g. "citizen_portal", "gdacs_rss", "open_meteo"
source_id                   TEXT                  -- source's own identifier for this item, if any
source_url                  TEXT
source_trust_score          NUMERIC(4,3)          -- configured baseline, feeds credibility scoring (MLSpec.md)

reported_at                 TIMESTAMPTZ NOT NULL   -- when the event was said to occur
ingested_at                 TIMESTAMPTZ NOT NULL DEFAULT NOW()

latitude, longitude         NUMERIC
geom                        GEOMETRY(Point, 4326)  -- PostGIS
city, district, state       TEXT
country                     TEXT DEFAULT 'India'

category                    TEXT NOT NULL CHECK (category IN taxonomy §1.1)   -- as submitted/detected pre-classification
severity                    TEXT CHECK (severity IN taxonomy §1.2)
description                 TEXT CHECK (char_length(description) <= 2000)
hashtags                    TEXT[]                 -- #IMD and similar tags, per PS requirement
author_id                   TEXT
platform                    TEXT

photo_urls                  TEXT[]
video_urls                  TEXT[]

classified_category         TEXT CHECK (classified_category IN taxonomy §1.1)  -- model output
classification_confidence   NUMERIC(4,3)
duplicate_score             NUMERIC(4,3)
credibility_score           NUMERIC(4,3)
credibility_reasons         TEXT[]                 -- MUST be populated, never left empty — see Guardrails.md

event_id                    UUID REFERENCES events(event_id)   -- which canonical event this merged into
priority                    VARCHAR(20) DEFAULT 'normal'        -- 'normal' | 'critical'

verification_status         TEXT DEFAULT 'pending' CHECK (verification_status IN taxonomy §1.3)

created_at, updated_at      TIMESTAMPTZ
```

### 2.2 `events` — canonical merged record [PHASE 1]

The authoritative, deduplicated, dashboard-facing record. One row per real-world event, built from one or more `reports`.

```sql
event_id                    UUID PRIMARY KEY
category                    TEXT NOT NULL CHECK (category IN taxonomy §1.1)
severity                    TEXT CHECK (severity IN taxonomy §1.2)
description                 TEXT                    -- representative/merged description

latitude, longitude         NUMERIC
geom                        GEOMETRY(Point, 4326)
city, district, state       TEXT
country                     TEXT DEFAULT 'India'

first_reported_at           TIMESTAMPTZ NOT NULL
last_reported_at            TIMESTAMPTZ NOT NULL
report_count                INTEGER DEFAULT 1
source_count                INTEGER DEFAULT 1       -- DISTINCT source types contributing — see MLSpec.md
                                                       -- corroboration note: count distinct source types,
                                                       -- not raw report count, to resist coordinated reposts
contributing_source_types   TEXT[]

classified_category         TEXT CHECK (classified_category IN taxonomy §1.1)
classification_confidence   NUMERIC(4,3)
credibility_score           NUMERIC(4,3)
credibility_reasons         TEXT[]                  -- MUST be populated on every write path, no exceptions
verification_status         TEXT DEFAULT 'pending' CHECK (verification_status IN taxonomy §1.3)
verification_reasons        TEXT[]
verified_by                 UUID REFERENCES users(user_id)   -- [PHASE 2+] nullable until auth ships
verified_at                 TIMESTAMPTZ

priority                    VARCHAR(20) DEFAULT 'normal'

processed_at                TIMESTAMPTZ             -- stream-processing completion timestamp, latency tracking
written_at                  TIMESTAMPTZ             -- database write timestamp, latency tracking

embedding                   VECTOR(384)              -- nullable; see Schema §4 for extension requirement

media_urls                  TEXT[]                  -- representative media across contributing reports

created_at, updated_at      TIMESTAMPTZ
```

### 2.3 `verification_log` — audit trail [PHASE 1 shape / PHASE 2+ populated once auth exists]

```sql
log_id                      UUID PRIMARY KEY
event_id                    UUID NOT NULL REFERENCES events(event_id) ON DELETE CASCADE
action                      TEXT NOT NULL CHECK (action IN
                               ('verified','needs_review','suspicious','duplicate','rejected'))
performed_by                UUID REFERENCES users(user_id)     -- nullable at Phase 1 (shared login, no per-user identity yet)
notes                       TEXT                                -- REQUIRED at the application layer for
                                                                   -- suspicious/rejected actions — see AppFlow.md §3
performed_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

### 2.4 `sources` — source registry [PHASE 1]

```sql
source_name                 TEXT PRIMARY KEY
source_type                 TEXT NOT NULL CHECK (source_type IN taxonomy §1.4)
trust_baseline               NUMERIC(4,3) NOT NULL
enabled                     BOOLEAN NOT NULL DEFAULT true
last_successful_fetch_at    TIMESTAMPTZ              -- surfaced prominently in Admin module, Design.md §2.3
created_at, updated_at      TIMESTAMPTZ
```

### 2.5 `users` — admin/analyst accounts [PHASE 2+ shape defined now, not enforced until auth ships]

```sql
user_id                     UUID PRIMARY KEY
email                       TEXT UNIQUE NOT NULL
password_hash               TEXT NOT NULL             -- never store plaintext, ever, even at prototype stage
role                         TEXT NOT NULL CHECK (role IN taxonomy §1.5)
is_active                   BOOLEAN NOT NULL DEFAULT true
created_at, updated_at      TIMESTAMPTZ
```

---

## 3. Schema Design Rules

1. **`credibility_reasons` and `verification_reasons` are never optional in practice**, even though nullable/empty-array in SQL — the application layer must never write an empty array here on a real, scored event. This is the single most important carried-forward lesson from prior evaluation of this design pattern (see `Guardrails.md`).
2. **`event_id` on `reports` is the merge link** — a report starts with `event_id = NULL`, gets assigned once the matching/clustering step (`MLSpec.md`) either matches it to an existing event or creates a new one.
3. **Corroboration counting uses `contributing_source_types` (distinct types), not raw `report_count`** — this resists a coordinated set of reposts from one source type artificially inflating trust (`MLSpec.md` credibility formula).
4. **`embedding` must be nullable** — the system must function correctly with this column unpopulated if the embedding service (`TechSpec.md` §2, §8) is unavailable, per the graceful-degradation principle.
5. **No table stores plaintext credentials.** `users.password_hash` only, via a proper hashing library — this rule applies from the very first version of this table, even before auth is actively enforced.
6. **Every schema change is a proper migration**, applied in order, tracked — no hand-edited "just fix it in prod" schema drift. Migration tooling choice belongs in `ImplementationPlan.md`; the rule itself belongs here as a schema-integrity requirement.

---

## 4. Required Database Extensions [PHASE 1]

- **PostGIS** — for `geom` columns and geospatial queries (proximity search, map-area filtering).
- **pgvector** — for the `embedding` column and similarity search (`MLSpec.md`). Must be added via a migration that gracefully no-ops if the extension is genuinely unavailable in a given environment, rather than hard-failing the whole schema setup — see `Guardrails.md` for the general graceful-degradation rule this follows.

---

## 5. What's Deliberately Simplified vs. a More Complex Prior Design

For a small team building fresh, this schema intentionally omits some structures that a larger system might add later, to avoid premature complexity:

- **No separate `event_clusters` table** — clustering membership is expressed directly via `reports.event_id`; a dedicated clustering-metadata table can be added later ([PHASE 2+]) if cluster-level analytics beyond what `events` already captures become necessary.
- **No `verification_outbox`/transactional-publishing table at Phase 1** — add this ([PHASE 2+]) only once a genuine downstream consumer of verification-state-change events (e.g., a push-notification service) exists; building it earlier would be speculative infrastructure with no current consumer.
