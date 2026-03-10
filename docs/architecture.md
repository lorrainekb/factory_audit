# Design Decisions

This document explains the "why" behind key architectural choices.

## Decision 1: Event Sourcing Over Mutable State

**Choice:** Store immutable events; derive current state from them.

**Not:** Store a "current_worker_session" table with mutable state.

### Why This Decision?

The spec explicitly requires:
> "The audit trail must be immutable: past events cannot be altered, only appended to."

Event sourcing makes this a first-class architectural pattern, not an afterthought.

### Tradeoff

**Pros:**
- Immutability is baked in
- Audit trail is the source of truth
- Replay is possible (for debugging)
- No accidental mutation of history

**Cons:**
- Slightly more code for projections
- Current state must be calculated each time
- Slightly more complex to reason about initially

**Verdict:** For an audit system, the pros massively outweigh the cons. Current state is secondary; audit trail is primary.

---

## Decision 2: Preserve Conflicts Instead of Preventing Them

**Choice:** Accept all structurally valid events, even if they conflict logically.

**Not:** Reject events that would create conflicts.

### Why This Decision?

In an offline-first system, you cannot prevent conflicts. Here's why:

```
Device A (offline):  Worker 5 starts Job 10 @ 09:00
Device B (offline):  Worker 5 starts Job 11 @ 09:05

At this moment:
- Device A has no way to know about Device B's start
- Device B has no way to know about Device A's start
- Both devices are offline; no server to ask

Both devices are right, from their perspective.
Both events are true.
```

Attempting to prevent this would require:
- Real-time connectivity (not guaranteed)
- A central authority checking state before every action (offline-unfriendly)
- A false sense of security (which contradicts the spec)

### Tradeoff

**Pros:**
- Offline-first design is clean and simple
- Every device observation is preserved
- Auditability is maintained even under failure
- Supervisors see actual reality (not sanitized version)

**Cons:**
- Some conflicts make it to the supervisor inbox
- Requires supervisor review workflow
- Feels less "clean" than auto-resolution

**Verdict:** For a *trustworthy* audit system, preserving conflicts is the only honest choice. The alternative is false certainty.

---

## Decision 3: Sessions Are Derived, Not Stored

**Choice:** A session is just a pair of WORK_STARTED / WORK_STOPPED events sharing a session_id.

**Not:** A separate "sessions" table with mutable state.

### Why This Decision?

Storing sessions separately creates redundancy:

```
Option A (Separate Table):
  events table: WORK_STARTED
  events table: WORK_STOPPED
  sessions table: { session_id, worker_id, job_id, start_time, end_time, duration }
  ^ This duplicates information already in events

Option B (Derived):
  events table: WORK_STARTED, WORK_STOPPED
  ^ Calculate session details from these on read
```

With Option A, you must keep two tables in sync. If your conflict detection updates a session's state, now you have mutable audit data (violates spec).

With Option B, there's one source of truth: the events. Sessions are read models, not write models.

### Tradeoff

**Pros:**
- Single source of truth
- No sync burden between tables
- If conflict detection improves, all projections automatically improve
- Simpler to reason about

**Cons:**
- Must calculate sessions on every read (minor performance cost)
- Requires more code for projections

**Verdict:** For an immutable audit system, derived state is the correct pattern.

---

## Decision 4: Two Timestamps Per Event

**Choice:** Store both `occurred_at` (device time) and `recorded_at_server_time` (server receipt).

**Not:** Just use server time.

### Why This Decision?

These timestamps serve different audit purposes:

- `occurred_at` = what the worker experienced ("I started at 9:00 AM")
- `recorded_at_server_time` = when the audit trail registered the claim ("server received this at 10:15 AM")

Example conflict analysis:

```
Worker claims they started Job A at 09:00 on Device 1.
Worker also claims they started Job B at 09:05 on Device 2.
Device 2 was offline until 10:30, then synced.

Timeline (device times):
  09:00 Job A starts
  09:05 Job B starts

Timeline (server times):
  09:15 Job A claim received
  10:30 Job B claim received

With only server times, you lose the information that both starts happened ~5 minutes apart in the worker's experience.
With both timestamps, you preserve the full picture.
```

This matters for supervisor review. A supervisor might think: "Device 2 was offline at 09:05. Device 1 was online. That explains why we didn't prevent the conflict."

### Tradeoff

**Pros:**
- Richer audit picture
- Better supervisor understanding
- Good for forensics

**Cons:**
- Slightly more complex timestamp handling
- Must account for clock drift

**Verdict:** Worth the complexity. Audit trails benefit from preserving multiple perspectives.

---

## Decision 5: Three Conflict Types (Not One Catch-All)

**Choice:** Define specific conflict types: WORKER_OVERLAP, DEVICE_OVERLAP, STOP_WITHOUT_START.

**Not:** Generic "conflict" flag.

### Why This Decision?

Different conflicts have different meanings:

**WORKER_OVERLAP:**
- Same worker, two simultaneous jobs
- Suggests: worker forgot to stop first job, or was cloned across devices

**DEVICE_OVERLAP:**
- Same device, two different workers simultaneously
- Suggests: device handoff went wrong; previous worker didn't stop before handing over

**STOP_WITHOUT_START:**
- Stop event with no matching start
- Suggests: data corruption or app crash during start

Each type points to a different type of failure, and supervisors need to know which.

### Tradeoff

**Pros:**
- Supervisor gets actionable diagnosis
- Easier to write rules per type
- Extensible for future conflict types

**Cons:**
- Slightly more complex detection logic
- More things to test

**Verdict:** Small complexity cost for much better observability.

---

## Decision 6: Manual Sync ("Sync Now" Button) Over Background Sync

**Choice:** App has explicit "Sync Now" button. No background sync (for MVP).

**Not:** Automatic background sync on connectivity.

### Why This Decision?

Background sync is seductive but adds complexity:

```
Background sync logic:
  - Detect connectivity changes
  - Debounce rapid changes
  - Handle sync failure without UI
  - Retry logic
  - User notification of conflict detection post-sync
```

For an MVP focused on demonstrating trustworthiness, a manual sync button is much clearer:

```
Manual sync logic:
  - User taps "Sync Now"
  - Events are sent
  - Response is shown (conflicts, errors, accepted count)
  - Done
```

The demo walkthrough becomes much easier to control and explain.

### Tradeoff

**Pros:**
- Explicit and testable
- User has visibility and control
- Easier to demo conflict detection
- Fewer edge cases

**Cons:**
- Requires user action (less convenient)
- Possible to forget to sync

**Verdict:** For MVP, explicit beats implicit. Real product could add background sync later without changing the core design.

---

## Decision 7: SQLite (Not Firebase Firestore)

**Choice:** SQLite locally on device. Custom backend for server-side event store.

**Not:** Firebase / Firestore for everything.

### Why This Decision?

Firebase is tempting because it handles sync automatically. But for this problem:

**SQLite Pros:**
- Explicit, simple sync model (batch HTTP)
- Easy to implement idempotency (by event_id)
- No vendor lock-in
- Clear data model (relational)
- Easy to demonstrate "immutable append-only" pattern
- Good for showing architectural thinking

**Firestore Cons:**
- Sync model is less visible (automatic)
- Conflict resolution is implicit (last-write-wins or custom rules)
- Harder to demonstrate "events are immutable"
- More of a black box; harder to explain to interviewer

For an interview focused on architectural judgment, demonstrating a clear, explicit sync model beats speed.

### Tradeoff

**Pros:**
- Shows deep architectural thinking
- Full control over sync and conflict logic
- Easier to explain in walkthrough
- Good for teaching

**Cons:**
- More code to write
- Sync is manual, not automatic
- Slightly slower to build

**Verdict:** Right choice for this interview context. Different decision if this were a real product.

---

## Decision 8: No Authentication (MVP)

**Choice:** Worker selection is a simple dropdown. No login.

**Not:** Full auth system.

### Why This Decision?

Auth is a distraction from the core problem.

The spec focuses on:
- Offline capability ✓
- Immutable audit trail ✓
- Conflict handling ✓

Auth would add:
- Database schema for users
- Password management or SSO integration
- Permissions logic
- Session management

None of that affects your ability to demonstrate offline-first, event-sourced, conflict-aware design.

### Tradeoff

**Pros:**
- Focus on core problem
- Faster build
- Cleaner demo

**Cons:**
- Less production-like
- Anyone can select any worker (data integrity risk)

**Verdict:** Acceptable for MVP. Mention in assumptions: "Workers are selected via dropdown; production would add auth."

---

## Decision 9: Simple Backend Over Fancy Architecture

**Choice:** Straightforward Node.js + Express or similar. Simple sync endpoint.

**Not:** CQRS, Event Store library, complex async patterns.

### Why This Decision?

Fancy patterns can look smart but add noise:

```
Option A (Fancy):
  - Event store library (EventStoreDB, Axon, etc.)
  - CQRS pattern
  - Event bus / pub-sub
  - Saga patterns
  - Complex deploy setup

Option B (Simple):
  - One "events" table
  - One "conflicts" table
  - Simple endpoint that appends and detects
  - Standard relational DB
```

Option B is more humble, more testable, and clearer to explain.

### Tradeoff

**Pros:**
- Faster to build
- Easier to debug
- Easier to explain
- Fewer dependencies

**Cons:**
- Doesn't show knowledge of fancy patterns
- Less scalable (but not required for MVP)

**Verdict:** Humility scores higher than cleverness in interviews. Show you can build something that works and you understand why.

---

## Decision 10: Supervisor Visibility Over Conflict Resolution

**Choice:** Show conflicts. Do not attempt to auto-resolve or provide workflow for resolution.

**Not:** Implement full conflict resolution workflow.

### Why This Decision?

Full resolution would require:
- Rules for choosing one event over another (which rule is right?)
- Supervisor review and approval
- Audit trail of the resolution itself
- Rollback capability

That's a whole system inside the system.

For MVP, the core claim is: "The system preserves truth and flags contradiction."

Resolving contradiction is the supervisor's job, not the system's.

### Tradeoff

**Pros:**
- Simpler scope
- Clearer responsibility boundary
- Honest about what system can do

**Cons:**
- Supervisor must manually handle conflicts
- Not a complete "solution"

**Verdict:** This is a good scope boundary. You show you know where the hard problem is (resolution) and deliberately don't solve it for MVP.

---

## Where AI Led Me Wrong (Honest Reflection)

During design exploration, I considered:

### Suggestion 1: "Maybe you could auto-resolve overlaps by keeping the first event?"

**Why it sounded good:**
- Feels clean
- Gives the appearance of a complete system
- Less work for supervisors

**Why it was wrong:**
- Silently overwrites device observation
- Violates "immutable audit trail"
- If the rule is wrong, you've corrupted history
- Can't be undone

**How I caught it:**
- Compared against the spec: "past events cannot be altered"
- Realized auto-resolution is a form of alteration
- Decided: preservation is more trustworthy than automatic resolution

### Suggestion 2: "You could store 'current_active_session' as a mutable record for performance"

**Why it sounded good:**
- Faster queries
- Easier to check "is worker currently active?"
- Feels more like traditional DB design

**Why it was wrong:**
- Creates redundancy with events
- Introduces mutation into audit data
- Sync burden between two tables
- If they diverge, which is truth?

**How I caught it:**
- Realized this is derivable state, not source of truth
- Recognized the redundancy risk
- Decided: derive from events, accept the query cost

---

## Summary

These decisions are tied together:

1. **Events are immutable** (spec requirement)
2. **Therefore conflicts are preserved** (offline reality)
3. **Therefore sessions are derived** (no mutation)
4. **Therefore timestamps are dual** (preserve context)
5. **Therefore sync is explicit** (show the process)
6. **Therefore backend is simple** (show the thinking)
7. **Therefore no auth** (scope focus)
8. **Therefore supervisor reviews conflicts** (honest boundary)

This coherence is what makes the system defensible, not the individual choices.
