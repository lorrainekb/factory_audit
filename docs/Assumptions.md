# Assumptions & Scope Boundaries

This document clarifies what the system assumes and what it deliberately does NOT attempt to solve in this MVP.

## Core Assumptions

### 1. Worker Identity

**Assumption:** Workers are identified through supervisor-entered selection (dropdown) or PIN, not biometric authentication.

**Implication:** 
- No login system required
- UI is: "Select your worker from this list"
- Production version would add proper authentication

**Why:**
- Keeps MVP scope tight
- Avoids whole auth infrastructure
- Focuses on event capture, not identity verification

---

### 2. Device Identity

**Assumption:** Each device has a persistent device_id generated once on first app launch and stored locally.

**Implication:**
- Every event includes this device_id
- Server can trace events back to specific physical hardware
- Device identity is stable across sessions

**Why:**
- Essential for "device overlap" conflict detection
- Enables device-specific audit trails
- Allows supervisors to understand "which device caused this conflict?"

---

### 3. Clock Drift

**Assumption:** Device clocks may drift from server time by hours or days.

**Implication:**
- Both `occurred_at` (device time) and `recorded_at_server_time` (server time) are stored
- Server time is used for audit trail ordering, but device time is preserved for context
- Supervisor can see: "Device reported this at 09:00, but server didn't receive it until 14:00 (device was offline)"

**Why:**
- Reflects reality of factory floor devices
- Allows forensic understanding of what happened
- Prevents false certainty about event ordering when devices are offline

---

### 4. Session Cardinality

**Assumption:** A worker should normally have at most one active session at a time.

**Implication:**
- If two WORK_STARTED events exist for the same worker without an intervening WORK_STOPPED, it's flagged as conflict
- Normal flow: WORK_STARTED → WORK_STOPPED → WORK_STARTED
- Abnormal flow triggers conflict detection

**Why:**
- Reflects factory floor reality (one person, one job at a time)
- Makes conflict detection clear and testable
- Doesn't assume this is always true (it might not be in edge cases), but flags when it's violated

---

### 5. Sync Is Idempotent

**Assumption:** Sync is safe to retry. The same event sent twice = one event in the log.

**Implication:**
- Server never updates existing event_id
- Duplicate event_ids are silently ignored
- App can safely retry without risk of duplication
- Network glitches don't corrupt data

**Why:**
- Critical for offline-first reliability
- Makes sync simple and testable
- event_id is the synchronization anchor

---

### 6. Event Structure

**Assumption:** Every event has this shape:

```
{
  eventId,
  deviceId,
  workerId,
  jobId (nullable for some events),
  sessionId,
  eventType,
  occurredAt,
  recordedLocallyAt,
  payload (JSON)
}
```

**Implication:**
- All events are validated against this schema
- Malformed events are rejected, not conflicts
- payload is flexible for future extensibility

**Why:**
- Consistent structure makes detection and validation easy
- payload allows future event types without schema change

---

### 7. Dual-Perspective Timestamps

**Assumption:** Every event carries two timestamps:
- `occurred_at` (device's claim about when it happened)
- `recorded_at_server_time` (server's claim about when it learned about it)

**Implication:**
- **For audit ordering:** Server uses `recorded_at_server_time` (canonical ingestion order)
- **For conflict detection:** Server sorts by `recorded_at_server_time` to determine sequence
- **For supervisor view:** Both timestamps are shown, so supervisor sees both worker perspective and server perspective
- **For timeline reconstruction:** Supervisor can reconstruct what the worker *experienced* (via `occurred_at`) vs. what the server *recorded* (via `recorded_at_server_time`)

**Why:**
- Server time is authoritative for the canonical audit log ordering
- Device time is honest about what the worker/device *claimed* happened
- Together they enable forensic understanding (e.g., "device was offline for 18 hours")
- Allows detection of clock drift without rewriting history
- Supervisor can see both perspectives and make informed judgments

---

### 7.5. Timestamp Usage: Two Different Purposes

To remove any ambiguity, here's exactly how each timestamp is used:

**`occurred_at` (Device Time):**
- **Used for:** Reconstructing the claimed worker/device timeline
- **Used in:** Supervisor views (shows what worker experienced)
- **NOT used for:** Conflict detection ordering or audit log sequencing
- **Reason:** Allows supervisor to understand device perspective even if clock drifted

**`recorded_at_server_time` (Server Time):**
- **Used for:** Canonical audit log ordering
- **Used for:** Conflict detection (all sorts are by this field)
- **Used in:** Determining sequence of events in authoritative log
- **Reason:** Server is single source of truth; this proves when claim arrived

**In Supervisor UI, show both:**
```
WORK_STARTED | Worker: Alice | Job: Assembly12
  Worker's device said: 2024-03-08 @ 13:55
  Server recorded at: 2024-03-08 @ 14:10
  [Interpretation: Device was offline for ~15 minutes before sync]

[If timeline has conflict]:
WORK_STARTED | Worker: Bob | Job: Welding5
  Worker's device said: 2024-03-08 @ 13:57
  Server recorded at: 2024-03-08 @ 14:12
  
⚠️ CONFLICT: Alice and Bob both have overlapping sessions
  (determined by server receipt times 14:10 and 14:12)
  But check device times: Alice claims 13:55, Bob claims 13:57
  - If both correct, real overlap is ~2 minutes
  - If Alice's device was drifting, overlap might be different
  Supervisor must review.
```

This dual perspective is the whole point of storing both timestamps.

---

### 8. Conflicts Are Preserved, Not Resolved

**Assumption:** When two events conflict, both are preserved. The system does not attempt to choose "the true one."

**Implication:**
- WORKER_OVERLAP events create conflict records, but both underlying events remain
- DEVICE_OVERLAP events create conflict records, but both underlying events remain
- Supervisor must review and decide how to interpret the conflict
- System marks affected sessions as "disputed" in projections

**Why:**
- In offline-first systems, both devices may be equally right
- Choosing one would be silently overwriting history
- Audit integrity requires preserving all claims

---



**Assumption:** When device goes offline, all events created locally are persisted in SQLite and will eventually sync when connectivity returns.

**Implication:**
- Local SQLite is the safety net
- App restart doesn't lose pending events
- User doesn't need to know about sync status (but it's visible if they care)
- Sync can happen hours or days later; events are still valid

**Why:**
- Critical for offline-first guarantee
- Reflects real factory floor (devices go offline for hours)
- Prevents "oh no, my event was lost" scenarios

---

## What MVP Does NOT Solve

These are explicitly out of scope for this submission. They are good ideas for a real product, but not required for demonstrating the core design.

### ❌ Authentication & Authorization

**Out of scope:**
- User login
- Password management
- Permission levels (worker can't see supervisor data, etc.)
- Role-based access control

**Why:**
- Orthogonal to the core problem (offline + audit + conflicts)
- Would double the scope
- Interview is about event sourcing and offline-first, not security

**How to mention in demo:**
- "In production, workers would authenticate. For MVP, I focus on the audit system itself."

---

### ❌ Device Management

**Out of scope:**
- Device enrollment / registration
- Device revocation / unregistration
- Device assignment to workers
- Device fleet status (online/offline/battery/etc.)

**Why:**
- Adds administrative overhead
- Not required for demonstrating audit trail integrity
- Could be bolted on later

**How to mention in demo:**
- "Real system would have device inventory and assignment. For MVP, devices self-identify."

---

### ❌ Real-Time Synchronization

**Out of scope:**
- WebSocket connections
- Live conflict notification
- Real-time supervisor dashboard

**Why:**
- HTTP polling or manual sync is sufficient for demo
- Real-time adds infrastructure complexity (and cost)
- Doesn't affect the core design

**How to mention in demo:**
- "MVP uses manual 'Sync Now' button. Production would have background sync."

---

### ❌ Conflict Resolution Workflows

**Out of scope:**
- Supervisor UI for choosing which event is "true"
- Automatic conflict resolution rules
- Payroll impact calculation when conflicts exist
- Dispute escalation

**Why:**
- Conflicts are specific to the business context
- Supervisor judgment is required
- Out of scope for MVP

**How to mention in demo:**
- "System flags conflicts and shows them to supervisors. Resolving them is business logic, not system logic."

---

### ❌ Advanced Reporting

**Out of scope:**
- Charts and dashboards
- Time tracking analysis
- Worker productivity metrics
- Job costing integration

**Why:**
- Nice-to-have, not core
- Would bloat the scope
- Audit trail is sufficient for other systems to build on

---

### ❌ Mobile App Polish

**Out of scope:**
- Beautiful UI design system
- Accessibility features (WCAG compliance)
- Localization / i18n
- Offline indicator animations

**Why:**
- Interviewer cares about architecture, not design
- Would waste time better spent on core logic
- MVP UX can be functional

---

### ❌ Complex Sync Scenarios

**Out of scope:**
- Differential sync (only send changed events)
- Compression or encryption of sync payloads
- Resume on network interrupt mid-sync
- Selective sync (sync only certain jobs)

**Why:**
- Batch sync is sufficient
- HTTP handles retry automatically
- Scope boundary

---

### ❌ Testing Harness

**Out of scope:**
- Comprehensive unit test suite
- Integration tests for all edge cases
- Load testing
- Chaos engineering

**Why:**
- MVP needs to work, not be bulletproof
- A few key tests (sync idempotency, conflict detection) are enough
- Demo is the proof

---

## What IS Required for MVP

To be clear, these are mandatory:

✅ **Local Event Capture**
- Worker can start/stop work
- Events are saved locally immediately
- Survives app restart

✅ **Offline Guarantee**
- App works with no connectivity
- Events are not lost

✅ **Sync Mechanism**
- Manual button to sync pending events
- Device sends events to server
- Server appends them immutably
- App marks them as synced

✅ **Conflict Detection**
- WORKER_OVERLAP flagged
- DEVICE_OVERLAP flagged
- STOP_WITHOUT_START flagged
- Conflicts preserved (not auto-resolved)

✅ **Supervisor Visibility**
- Audit timeline view
- Conflict list view
- Both show original events

✅ **Clear Design Explanation**
- Architecture doc
- Design decisions doc
- Assumptions doc
- Demo walkthrough

---

## Tradeoff Matrix

To help you make decisions during build, here's a tradeoff table:

| Feature | Scope | Value | Cost | Decision |
|---------|-------|-------|------|----------|
| Worker dropdown | MVP | High | Low | ✅ Include |
| Full auth system | Post-MVP | Medium | High | ❌ Defer |
| Manual sync button | MVP | High | Low | ✅ Include |
| Background sync | Post-MVP | Medium | Medium | ❌ Defer |
| Conflict detection | MVP | Very High | Medium | ✅ Include |
| Conflict resolution UI | Post-MVP | Medium | High | ❌ Defer |
| SQLite local store | MVP | Very High | Low | ✅ Include |
| Device management | Post-MVP | Medium | High | ❌ Defer |
| Beautiful UI | Post-MVP | Low | Medium | ❌ Defer |
| Simple working UI | MVP | Medium | Medium | ✅ Include |

---

## How to Use This Document

**During the build:**
- Refer back when you're tempted to add something
- Ask: "Is this MVP or post-MVP?"
- If post-MVP, defer it or mention it in README as future work

**During the demo:**
- Lead with what you DID (MVP scope)
- Mention what you DIDN'T (post-MVP scope)
- Show this document as proof of intentional scope control

**During the interview discussion:**
- Interviewer will ask: "Why didn't you include X?"
- Answer: "X is valuable but orthogonal to the core audit design. For MVP, I focused on the immutable audit trail and offline-first guarantee."
- Shows judgment

---

## Realistic Scope for 10 Days

With these boundaries, 10 days breaks down like this:

**Days 1–2:** Design & setup
- Finalize architecture docs (you're doing this now)
- Create Flutter project structure
- Create backend skeleton
- Lock in database schema

**Days 3–4:** Core worker flow
- Event capture (start/stop)
- Local SQLite storage
- Basic UI (2–3 screens)

**Days 5–6:** Sync & backend
- Backend event ingestion
- Sync mechanism
- Idempotency logic

**Days 7–8:** Conflict detection & supervisor view
- Conflict detection logic
- Audit timeline UI
- Conflict list UI

**Days 9–10:** Polish & demo
- Documentation
- Demo walkthrough
- Recording

This is achievable if you stay within the scope in this document.

---

## Sign-Off

If you agree with these assumptions and scope boundaries, you're ready to start building. Share this document with your interviewers if they ask about decisions. It shows you thought clearly about what matters and what doesn't.
