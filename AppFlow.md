# AppFlow — User Flows Per Role
### SIH26069 — National Weather Big Data Analytics Platform
### See `PRD.md` for role definitions and document index. Screen-level detail for each step lives in `Design.md`. This file covers the sequence of actions each role takes, decision points, and edge cases.

**Tagging:** [PHASE 1] = build this flow first. [PHASE 2+] = flow requires capability not yet built (usually authentication — see `SecurityCompliance.md`).

---

## 1. Citizen Flow — Submitting a Report

```
Open report submission page
        ↓
See a short, clear consent notice (what data is collected, why) [PHASE 1]
        ↓
Fill report:
  - Event category (dropdown, from fixed taxonomy — Schema.md)
  - Free-text description
  - Location: auto-detect via device GPS (with permission) OR manual pin/search
  - Optional: attach photo(s)/video(s)
        ↓
    ┌─── location permission denied? ──→ fall back to manual location entry,
    │                                     do not block submission
    │
    ┌─── no network / poor connectivity? ──→ queue submission locally, retry on
    │                                         reconnect, show clear "pending
    │                                         upload" state — never silently
    │                                         lose a filled-in report
    ↓
Review screen — shows exactly what will be submitted
        ↓
Submit
        ↓
Confirmation shown immediately (report received) — NOT the same as "verified,"
this must be worded so the citizen understands their report is now in the
review pipeline, not already confirmed as a real event
        ↓
[PHASE 2+] — citizen can optionally check back (via a reference ID or, if
accounts exist later, a personal submission history) to see their report's
current verification status
```

### Design constraints this flow imposes
- Must work on a low-end mobile device with unreliable connectivity — this is the most common real-world condition for actual citizen reporting during a weather event.
- Must never require a login/account to submit a first report [PHASE 1] — friction here directly reduces real-world reporting during an actual emergency, when speed matters most. Optional accounts for tracking history are a [PHASE 2+] convenience, never a submission requirement.
- The distinction between "submitted" and "verified" must be visually and textually unambiguous at every step — this is a trust-critical wording decision, not just copy polish.

---

## 2. Public Viewer Flow — Browsing the Live Dashboard

```
Open dashboard (no login required — PHASE 1)
        ↓
Default view: live map + recent events list, most recent/highest-severity
surfaced first
        ↓
Apply filters (any combination):
  - Date/time range
  - Event category
  - Severity
  - Location (state/district/city, or map-area selection)
  - Verification status
        ↓
    ┌─── new event arrives while viewing? ──→ appears live without a manual
    │                                          refresh (real-time delivery,
    │                                          TechSpec.md §6)
    ↓
Click into a specific event
        ↓
See full detail:
  - Category, severity, location, timestamp
  - Credibility score AND its underlying reasons (not just the number)
  - Verification status AND its underlying reasons
  - Number of corroborating/contributing reports
  - Any attached media
        ↓
[PHASE 2+] — subscribe to alerts for a specific area (push notification /
email when a new verified event appears near a saved location)
```

### Design constraints
- Every score shown must have its reasoning one click away, never buried or omitted — directly enforces the explainability requirement from `PRD.md` §7.
- Must be legible and fast on a mobile browser — a public viewer checking conditions during an actual event is very likely on a phone, possibly on a degraded connection.

---

## 3. Analyst Flow — Reviewing & Verifying Events [PHASE 2+ for auth-gated access; the underlying review UI can be built and tested in PHASE 1 behind a placeholder/no-auth gate, then locked down once auth ships]

```
Log in with analyst credentials
        ↓
Land on Verification Queue — events with status pending/needs_review/suspicious,
sorted by priority (urgent events surfaced first, then by age — see aging term
in MLSpec.md's priority formula)
        ↓
Select an event to review
        ↓
See full evidence bundle:
  - All contributing source reports (not just the merged canonical summary)
  - Credibility score breakdown, per-factor
  - Duplicate/corroboration detail — which reports were merged and why
  - Any attached media
  - Cross-reference against official data (weather API agreement, official
    alerts if available)
        ↓
Decide:
  ┌── Verify ──────────→ status becomes "verified," visible as trusted on
  │                       public dashboard
  ├── Mark needs_review ─→ stays in queue, flagged for a second opinion
  │                       (e.g., insufficient evidence either way)
  ├── Mark suspicious ──→ status becomes "suspicious," still visible but
  │                       clearly flagged, not hidden
  ├── Mark duplicate ───→ merged into an existing canonical event, removed
  │                       from queue
  └── Reject ───────────→ status becomes "rejected," removed from public view
        ↓
Add an optional note (required if rejecting or marking suspicious — an
analyst's reasoning for a negative decision should not be silently absent)
        ↓
Action logged (who, what, when, why) — see Schema.md's verification_log
        ↓
Queue advances to next event
```

### Design constraints
- The queue's default sort must respect priority/urgency, not simple recency — an analyst working through a backlog during a real event must see the most urgent items first, not just the newest.
- The evidence bundle must show enough for an analyst to make a confident decision WITHOUT needing to leave the platform to cross-check manually, wherever the data to do so already exists in-system.
- A rejection or suspicious-mark without a note should be discouraged/blocked in the UI — decisions with real consequences need a recorded reason.

---

## 4. Admin Flow — System & Source Management [PHASE 2+, depends on auth]

```
Log in with admin credentials
        ↓
Dashboard: system health overview
  - Source adapter status (which sources are live/healthy vs erroring)
  - Basic pipeline health (ingestion rate, processing lag if available)
  - Recent verification activity summary
        ↓
Source management:
  - Enable/disable individual source adapters
  - View per-source trust configuration (feeds into credibility scoring —
    MLSpec.md)
  - View last-successful-fetch time per source (catches a silently-broken
    adapter before it causes a data gap)
        ↓
User/role management [PHASE 2+]:
  - Grant/revoke analyst access
  - View audit log of admin actions
        ↓
[PHASE 3+, ties to SecurityCompliance.md] — deeper observability (latency
percentiles, error rates, alert configuration) once the monitoring stack
described there is built
```

### Design constraints
- An admin must be able to tell, at a glance, if a source has silently stopped working (e.g., an API key expired, a feed URL changed) — a stale "last successful fetch" timestamp is the simplest reliable signal for this and must be prominent, not buried.
- Source enable/disable must take effect without requiring a full system restart.

---

## 5. Cross-Cutting Flow Notes

- **No role's flow should ever dead-end without explanation.** Every action (submit, verify, reject, filter with zero results) must show a clear resulting state, never a blank screen with no feedback.
- **Every flow that shows a score or automated decision must make the "why" reachable within one additional click/tap** — this is the single most repeated design constraint across all four roles above, because it's the core trust mechanism the entire product depends on (`PRD.md` §3, Goal 2).
- Detailed screen layouts implementing each flow above are specified in `Design.md`.
