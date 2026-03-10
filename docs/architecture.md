# Factory Audit System – Architecture

## Problem Statement

A factory has 30 shared devices on the floor. Workers clock in and out of production jobs throughout a shift. Devices go offline regularly. Workers hand devices to each other mid-session. Supervisors need an accurate, auditable record of who worked on what and for how long, and that record must be trustworthy even when things went wrong.

## Core Design Principle

**The source of truth is the immutable event log, not a mutable "current state" table.**

Instead of storing:
```
worker 14 is currently on job 88
```

We store:
```
WORK_STARTED: worker 14, job 88, 09:03, device D-07
WORK_STOPPED: worker 14, job 88, 10:11, device D-07
WORK_STARTED: worker 14, job 91, 10:14, device D-11
```

This ensures:
- Immutability (past events cannot be altered)
- Auditability (every claim is preserved)
- Offline-first capability (events are captured locally first)
- Conflict visibility (contradictions are preserved, not hidden)

## System Architecture

```
┌──────────────────────────┐
│     Flutter App          │
│  (Worker & Supervisor)   │
└──────────┬───────────────┘
           │
           │ read/write
           ▼
┌──────────────────────────┐
│    Local SQLite DB       │
│  (Offline Event Queue)   │
└──────────┬───────────────┘
           │
           │ sync batch
           ▼
┌──────────────────────────┐
│    Backend API           │
│  (Event Ingestion)       │
└──────────┬───────────────┘
           │
           │ append-only
           ▼
┌──────────────────────────┐
│   Server Database        │
│  (Canonical Event Log)   │
└──────────────────────────┘
```

### Layer 1: Flutter App

**Responsibilities:**
- Worker selection
- Job selection
- Start/stop work
- Device handoff representation
- Sync trigger (manual "Sync Now" button)

**Not responsible for:**
- Conflict resolution
- Business logic beyond event creation
- Authentication (out of scope)

### Layer 2: Local SQLite Database

**Responsibilities:**
- Capture events locally (pending_sync = true)
- Persist events across app restarts
- Maintain sync queue
- Derive local "active session" state for UI

**Not responsible for:**
- Authoritative state
- Conflict detection (that's server-side)

### Layer 3: Backend API

**Responsibilities:**
- Accept event batches from devices
- Validate event structure (reject malformed)
- Deduplicate by event_id (idempotency)
- Append all valid events to immutable log
- Detect and flag conflicts post-sync
- Build read models (projections)

**Not responsible for:**
- Preventing conflicts (they're offline anyway)
- Auto-resolving conflicts
- Permission checks
- Worker identity verification

### Layer 4: Server Database

**Responsibilities:**
- Store events immutably (never update/delete)
- Store derived conflict records
- Store session projections
- Maintain audit trail

## Core Entities

### EventRecord

An immutable claim about what happened on a device.

```
event_id:              UUID
device_id:             which device
worker_id:             who performed the action
job_id:                which job (nullable for some events)
session_id:            groups start/stop pairs
event_type:            WORK_STARTED | WORK_STOPPED | DEVICE_HANDOFF
occurred_at:           device timestamp (user's local time)
recorded_locally_at:   when device recorded it
synced_at:             when server acknowledged it
payload:               JSON metadata
sync_status:           pending | inflight | synced | failed | duplicate
```

**Why this shape:**
- `event_id` ensures idempotency
- `occurred_at` + `recorded_locally_at` preserve device perspective
- `synced_at` proves server receipt
- `session_id` groups related events without needing a separate sessions table
- `sync_status` tracks delivery state locally

### ConflictRecord

Flagged when two or more events contradict each other.

```
conflict_id:           UUID
conflict_type:         WORKER_OVERLAP | DEVICE_OVERLAP | STOP_WITHOUT_START
worker_id:             worker involved
device_id:             device involved (if applicable)
related_event_ids:     which events caused this
detected_at:           when server detected it
status:                open | reviewed | resolved
details:               JSON explanation
```

**Why separate from events:**
- Conflicts are *derived* (read model), not source of truth
- Preserves original events unchanged
- Allows supervisor review workflow
- Makes conflict detection testable

### SessionProjection

Derived summary of start/stop pairs.

```
session_id:            UUID
worker_id:             who worked
job_id:                what they worked on
start_time:            when it began
end_time:              when it ended
duration_seconds:      calculated
state:                 open | closed | invalid
trust_status:          trusted | disputed
```

**Why derived:**
- No redundancy with raw events
- Recalculated on every projection build
- If conflict detection improves, projections automatically improve

## Event Types (MVP)

Only two required:

- `WORK_STARTED(worker_id, job_id, device_id, session_id, occurred_at)`
- `WORK_STOPPED(worker_id, job_id, device_id, session_id, occurred_at)`

Optional future:
- `DEVICE_HANDOFF_RECORDED` — explicit handoff event (but not needed for MVP; just stop + start)

**Why minimal:**
- Simpler to reason about
- Easier to test
- Fewer edge cases
- Can expand later

## Offline-First Behavior

### When Device Is Offline

1. User taps "Start Work"
2. App creates EventRecord locally
3. App inserts into SQLite with `sync_status = pending`
4. UI updates immediately from local data
5. No network call attempted
6. Event survives app restart (persisted in SQLite)

### When Device Comes Online

1. App detects connectivity
2. Collects all events where `sync_status = pending`
3. Sends POST /events/batch with event array
4. Server appends them to event log
5. Server returns `{ accepted: [...], duplicates: [...], rejected: [...] }`
6. App marks those rows as `sync_status = synced`
7. If a retry is needed, same event_id is idempotent (server ignores duplicate)

**Key principle:** Events are facts once created locally. Sync doesn't mutate them; it just broadcasts them.

## Sync Model

### Idempotency

By `event_id`, the sync is idempotent:

```
Device sends: [event_1, event_2, event_3]
Server receives, appends all three.
Response: { accepted: [uuid1, uuid2, uuid3], duplicates: [], rejected: [] }

Device retries same batch (network glitch):
Server receives same events again.
Response: { accepted: [], duplicates: [uuid1, uuid2, uuid3], rejected: [] }

App marks the duplicate event_ids as synced (idempotent).
```

### Sync Failure Handling

If POST /events/batch fails:
- Mark events as `sync_status = failed`
- Retry on next connectivity check
- Do NOT discard the events
- Do NOT mutate them

## Conflict Detection Strategy

### Rule 1: Worker Overlap

A worker should not have more than one active session simultaneously.

**Detection:**
```
For each worker:
  Sort events by occurred_at
  Track open sessions (start without matching stop)
  If a WORK_STARTED arrives while another session is open:
    Create CONFLICT of type WORKER_OVERLAP
    Mark both sessions as disputed
```

**Example:**
```
Device A (offline): Worker 5 WORK_STARTED Job 10 @ 09:00
Device B (offline): Worker 5 WORK_STARTED Job 11 @ 09:05
Neither device has WORK_STOPPED before the other start.

Both devices sync later.
Server detects overlap from 09:05 onward.
Creates CONFLICT: WORKER_OVERLAP with both events.
Supervisor sees both events and must review.
```

### Rule 2: Device Overlap

A device should not be used in overlapping sessions by different workers.

**Detection:**
```
For each device:
  Sort events by occurred_at
  Track open sessions
  If a WORK_STARTED for worker B arrives before worker A's session ends:
    Create CONFLICT of type DEVICE_OVERLAP
```

**Example:**
```
Device 1:
  Worker A starts Job 10 @ 09:00
  Worker B starts Job 11 @ 09:10 (before A stops)

Conflict: DEVICE_OVERLAP
Indicates device handoff went wrong (A didn't stop before B took device).
```

### Rule 3: Stop Without Start

A WORK_STOPPED event with no matching WORK_STARTED is suspicious.

**Detection:**
```
For each WORK_STOPPED:
  Check if a matching WORK_STARTED exists in same session
  If not, create CONFLICT of type STOP_WITHOUT_START
```

### Key Decision: Preserve, Don't Reject

**The system accepts all structurally valid events, even if they conflict logically.**

Why?
- Offline devices record truth as they see it
- Later, different truths may arrive from other devices
- Rejecting one device's claim destroys auditability
- Better to preserve both and flag the contradiction
- Supervisor can review and decide

**This is the hardest design decision.** Auto-resolving conflicts feels cleaner, but it silently overwrites device observations. For an audit system, that's a violation of trust.

## Database Design

### Local SQLite (Device)

```sql
CREATE TABLE workers (
  worker_id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  employee_code TEXT
);

CREATE TABLE jobs (
  job_id TEXT PRIMARY KEY,
  job_code TEXT NOT NULL,
  job_name TEXT NOT NULL
);

CREATE TABLE events_local (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  event_id TEXT NOT NULL UNIQUE,
  device_id TEXT NOT NULL,
  worker_id TEXT NOT NULL,
  job_id TEXT,
  session_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  occurred_at TEXT NOT NULL,
  recorded_locally_at TEXT NOT NULL,
  payload_json TEXT,
  sync_status TEXT NOT NULL DEFAULT 'pending',
  synced_at TEXT
);

CREATE INDEX idx_sync_status ON events_local(sync_status);
CREATE INDEX idx_worker_id ON events_local(worker_id);
CREATE INDEX idx_session_id ON events_local(session_id);
```

### Server Database (Postgres or equivalent)

```sql
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  event_id TEXT NOT NULL UNIQUE,
  device_id TEXT NOT NULL,
  worker_id TEXT NOT NULL,
  job_id TEXT,
  session_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  occurred_at TEXT NOT NULL,
  recorded_locally_at TEXT,
  recorded_at_server_time TEXT NOT NULL,
  payload_json TEXT,
  sync_batch_id TEXT
);

CREATE TABLE conflicts (
  conflict_id TEXT PRIMARY KEY,
  conflict_type TEXT NOT NULL,
  worker_id TEXT,
  device_id TEXT,
  related_event_ids TEXT NOT NULL,
  detected_at TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'open',
  details_json TEXT
);

CREATE TABLE session_projection (
  session_id TEXT PRIMARY KEY,
  worker_id TEXT NOT NULL,
  job_id TEXT,
  start_time TEXT,
  end_time TEXT,
  duration_seconds INTEGER,
  state TEXT NOT NULL,
  trust_status TEXT NOT NULL
);
```

## API Contract

### POST /api/events/batch

**Request:**
```json
{
  "events": [
    {
      "eventId": "uuid-1",
      "deviceId": "device-07",
      "workerId": "worker-12",
      "jobId": "job-88",
      "sessionId": "session-abc-123",
      "eventType": "WORK_STARTED",
      "occurredAt": "2026-03-09T08:15:00Z",
      "recordedLocallyAt": "2026-03-09T08:15:03Z",
      "payload": {}
    }
  ]
}
```

**Response:**
```json
{
  "accepted": ["uuid-1"],
  "duplicates": [],
  "rejected": [],
  "newConflicts": [
    {
      "conflictId": "conflict-xyz",
      "type": "WORKER_OVERLAP",
      "relatedEventIds": ["uuid-1", "uuid-2"]
    }
  ]
}
```

### GET /api/audit/worker/:workerId

Returns timeline of events for a worker.

### GET /api/audit/job/:jobId

Returns timeline of events for a job.

### GET /api/conflicts

Returns list of all flagged conflicts.

## Explicit Non-Goals (MVP Scope)

❌ User authentication
❌ Permission system
❌ Real-time synchronization (polling is fine)
❌ Supervisor conflict resolution workflows (just visibility)
❌ Automatic conflict resolution
❌ Device enrollment / device management
❌ Fancy UI or design system
❌ Background sync (manual "Sync Now" button is acceptable)

These can be added post-MVP without changing the core event model.

## Key Design Decisions

### Why Event Sourcing?

Event sourcing fits this problem because:
1. Spec explicitly requires immutability
2. Offline devices mean conflicting claims are expected
3. Audit trail is the primary requirement
4. Derived state (current worker, active job) is secondary

### Why Preserve Conflicts Instead of Prevent?

Offline reality: devices cannot talk to each other or the server in real-time. Any attempt to prevent conflicts (e.g., "reject starts if worker already active") would require perfect connectivity, which you don't have.

Better: preserve all claims, flag contradictions, let supervisors review.

### Why Separate Sessions From Events?

Sessions (start/stop pairs) are read model concerns. The source of truth is the two events. Storing a separate "session" table creates redundancy and the risk of desync.

Solution: derive sessions from event pairs during projection.

### Why Both Device Time and Server Time?

Device time reflects the worker's experience ("I really did start at 9:00 AM").
Server time reflects the audit order ("the server received this claim at 10:15 AM").

Both matter for different audit purposes.

### Why SQLite Locally, Not Firebase?

SQLite:
- Works offline without any setup
- Simple, predictable sync model
- Cheap to implement conflict detection
- No dependency on external service
- Good for teaching system thinking

Firebase Firestore:
- Easier to build on, but...
- Conflict model is less explicit
- Harder to demonstrate "immutable append-only" design
- More of a black box

For this interview, demonstrating clear architectural thinking matters more than speed.

## Assumptions

- Workers are identified via supervisor entry or PIN, not biometric identity.
- Device clocks may drift; both device time and server receipt time are stored.
- One worker should normally have at most one active session at a time.
- Conflicting events are preserved and flagged, not auto-resolved.
- Server time is the authoritative ordering for audit records.
- Sync is idempotent by event_id.
- Device handoff is represented as: previous worker stops, next worker starts.
- Supervisors review conflicts; system does not auto-resolve them.
- App target platforms: Android and iOS (mobile-first).

## Timeline

- Days 1–2: Design lock (you're here)
- Days 3–4: Local event capture
- Days 5: Backend event store
- Days 6: Sync engine
- Days 7: Conflict detection
- Days 8: Supervisor view
- Days 9: Testing
- Days 10: Documentation & recording

## Artifacts to Build

- `factory_audit/` — Flutter project
- `backend/` — API + server database
- `docs/` — architecture, assumptions, design decisions
- `demo_walkthrough.mp4` — recording

## Success Criteria

You will know this is working when:

1. ✅ Worker can start/stop job offline; events are captured locally
2. ✅ When online, events sync to backend without mutation
3. ✅ Backend appends events immutably
4. ✅ Overlapping sessions on same worker are detected and flagged
5. ✅ Overlapping sessions on same device are detected and flagged
6. ✅ Supervisor sees conflict list with original events preserved
7. ✅ You can explain the design tradeoffs clearly
8. ✅ You can articulate where a design suggestion (AI or otherwise) was wrong and how you caught it
