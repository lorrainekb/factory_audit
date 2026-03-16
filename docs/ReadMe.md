# Factory Audit System

A Flutter + Node.js application demonstrating offline-first architecture, immutable audit trails, and conflict detection in a distributed work-tracking system.

## Overview

Factory Audit simulates a real-world factory environment where workers log production activities across 30 shared devices with unreliable connectivity. The system captures work sessions locally, syncs when available, and detects contradictions while preserving an immutable audit trail.

**Key Features:**
- ✅ Offline-first architecture (works without connectivity)
- ✅ Immutable event log (events never updated, only appended)
- ✅ Conflict detection (three types: worker overlap, device overlap, stop without start)
- ✅ Device-specific sessions (worker can have concurrent sessions on different devices)
- ✅ Dual timestamps (device time + server receipt time)
- ✅ Supervisor dashboard (complete audit trail with conflict flags)
- ✅ Idempotent sync (safe retry on network failures)

---

## Architecture at a Glance

```
Flutter App (Worker & Supervisor)
         ↓ (read/write)
    Local SQLite DB (pending events)
         ↓ (sync batch)
   Node.js Backend API
         ↓ (append-only)
 Server SQLite Database (canonical event log)
```

**Core Principle:** The device records local claims. The backend reconciles them into an auditable history.

For detailed architecture, see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Quick Start

### Prerequisites

- **Flutter** 3.0+ ([Install](https://flutter.dev/docs/get-started/install))
- **Node.js** 18+ ([Install](https://nodejs.org/))
- **npm** (comes with Node.js)

### 1. Backend Setup

```bash
cd backend
npm install
node src/server.js
```

Server runs on `http://localhost:3000`. Database created automatically.

### 2. Configure Flutter Backend URL

Update in two places:
- `lib/main.dart`
- `lib/features/work/presentation/supervisor_view.dart`

Use the appropriate URL for your environment:

| Environment | URL |
|---|---|
| **Android Emulator** | `http://10.0.2.2:3000` |
| **iOS Simulator** | `http://127.0.0.1:3000` |
| **Physical Device** | `http://<your-computer-ip>:3000` |

For physical device (same Wi-Fi), test connectivity:
```
http://<your-computer-ip>:3000/api/sessions
```

Should return: `{"sessions":[]}`

### 3. Flutter Setup

```bash
cd ..
flutter pub get
flutter run
```

Or run on specific device:
```bash
flutter run -d <device_id>
```

---

## Demo Scenarios

### Scenario 1: Clean Session (Single Worker, Single Device)

1. Select "Worker Alpha" → "Default Device" → "Assemble Pump Unit"
2. Click "Start Work"
3. Click "Stop Work" after a few seconds
4. Click "Sync Now"
5. Switch to Supervisor view
6. ✅ See one closed session with **TRUSTED** status

### Scenario 2: Device Overlap Conflict (Device Handoff Gone Wrong)

1. Select "Worker Alpha" → "Station 001" → "Assemble Pump Unit"
2. Click "Start Work"
3. Click "Sync Now"
4. Switch worker to "Worker Beta" (keep same device)
5. Select "Pack Finished Goods"
6. Click "Start Work"
7. Click "Sync Now"
8. Switch to Supervisor view
9. ✅ See both sessions marked **DISPUTED** with "⚠️ Device Overlap" conflict

### Scenario 3: Worker Overlap Conflict (Same Worker, Two Jobs)

1. Select "Worker Alpha" → "Default Device" → "Assemble Pump Unit"
2. Click "Start Work"
3. Click "Sync Now"
4. Without stopping, select different job "Pack Finished Goods"
5. Click "Start Work"
6. Click "Sync Now"
7. Switch to Supervisor view
8. ✅ See sessions marked **DISPUTED** with "⚠️ Worker Overlap" conflict

---

## Features

### Worker Interface

- **Select Worker** - Choose from Alpha, Beta, Gamma, or Supervisor
- **Select Device** - Three simulated factory stations
- **Select Job** - Three production jobs
- **Start Work** - Creates local event immediately (offline-safe)
- **Stop Work** - Closes session on current device only
- **Sync Now** - Uploads pending events, receives conflict detection results
- **View Local Events** - See all pending/synced events
- **Clear DB** - Reset for demo restart

### Supervisor Dashboard

- **Sessions** - All work sessions with:
  - Worker name
  - Trust status (🟢 TRUSTED / 🔴 DISPUTED)
  - Device time (worker's perspective)
  - Server received time (audit order)
  - Duration in readable format
  - Session state (open/closed)

- **Conflicts** - All detected conflicts with:
  - Type (Device Overlap, Worker Overlap, Stop Without Start)
  - Worker involved
  - Status
  - Related sessions

- **Color Coding:**
  - 🟢 Green border = Session is trusted
  - 🔴 Red border = Session has conflict
  - Badge colors match conflict severity

---

## Data Flow

### When Worker Starts Work

```
1. App creates WORK_STARTED event locally
2. Stores in SQLite with sync_status = "pending"
3. UI updates immediately from local event
4. Device usable offline
```

### When Device Syncs

```
1. User taps "Sync Now"
2. App collects pending events
3. HTTP POST to /api/events/batch
4. Backend validates structure
5. Backend deduplicates by event_id
6. Backend appends to immutable log
7. Backend detects conflicts
8. Response: { accepted, duplicates, rejected, newConflicts }
9. App marks events as synced
```

### Conflict Detection

Backend detects three types:

1. **WORKER_OVERLAP** - Same worker, 2+ simultaneous jobs
2. **DEVICE_OVERLAP** - Same device, 2+ different workers
3. **STOP_WITHOUT_START** - Stop event with no matching start

All conflicts are **preserved** (flagged, not hidden).

---

## Project Structure

```
factory_audit/
├── README.md                    ← You are here
├── ARCHITECTURE.md              ← System design & event sourcing
├── DESIGN_DECISIONS.md          ← Key choices & tradeoffs
├── ASSUMPTIONS.md               ← MVP scope & constraints
│
├── lib/                         ← Flutter frontend
│   ├── main.dart                # App entry & router
│   ├── core/
│   │   ├── db/                  # SQLite interface
│   │   ├── models/              # Data models
│   │   ├── repositories/        # Data access
│   │   └── services/            # Sync & event logic
│   ├── features/
│   │   └── work/
│   │       └── presentation/    # Supervisor dashboard
│   └── ui/
│       └── screens/             # Worker screen
│
├── backend/                     ← Node.js backend
│   ├── server.js                # Express API
│   ├── db.js                    # Database connection
│   ├── schema.js                # Table creation
│   ├── validateEvent.js         # Event validation
│   ├── package.json
│   └── data/
│       └── factory_audit_server.db  # SQLite (auto-created)
│
├── pubspec.yaml                 ← Flutter dependencies
└── package.json                 ← Node.js dependencies
```

---

## API Contract

### POST /api/events/batch

**Upload pending events for sync.**

Request:
```json
{
  "events": [
    {
      "eventId": "uuid",
      "deviceId": "device-001",
      "workerId": "worker-alpha",
      "jobId": "job-001",
      "sessionId": "session-xyz",
      "eventType": "WORK_STARTED",
      "occurredAt": "2026-03-15T10:30:00Z",
      "recordedLocallyAt": "2026-03-15T10:30:01Z",
      "payload": {}
    }
  ]
}
```

Response:
```json
{
  "accepted": ["event-id-1", "event-id-2"],
  "duplicates": ["event-id-3"],
  "rejected": [
    {
      "eventId": "event-id-4",
      "reasons": ["occurred_at must be valid ISO timestamp"]
    }
  ],
  "newConflicts": [
    {
      "conflictId": "conflict-xyz",
      "type": "WORKER_OVERLAP",
      "relatedEventIds": ["event-id-1", "event-id-2"]
    }
  ]
}
```

### GET /api/sessions

**Fetch all work sessions for supervisor view.**

Response:
```json
{
  "sessions": [
    {
      "sessionId": "session-xyz",
      "workerId": "worker-alpha",
      "jobId": "job-001",
      "startTime": "2026-03-15T10:30:00Z",
      "endTime": "2026-03-15T11:15:00Z",
      "durationSeconds": 2700,
      "state": "closed",
      "trustStatus": "trusted",
      "serverReceivedTime": "2026-03-15T10:35:00Z"
    }
  ]
}
```

### GET /api/conflicts

**Fetch all detected conflicts.**

Response:
```json
{
  "conflicts": [
    {
      "conflictId": "conflict-xyz",
      "type": "DEVICE_OVERLAP",
      "workerId": "worker-beta",
      "deviceId": "device-001",
      "relatedEventIds": ["event-id-5", "event-id-6"],
      "detectedAt": "2026-03-15T10:40:00Z",
      "status": "open",
      "details": "Device-001 has concurrent sessions from different workers"
    }
  ]
}
```

---

## Testing Checklist

- [ ] App launches without errors
- [ ] Workers dropdown shows all options
- [ ] Start work creates local event immediately
- [ ] Local events list updates
- [ ] Sync Now uploads events
- [ ] Supervisor dashboard loads
- [ ] Sessions display with correct data
- [ ] Color coding works (green/red borders)
- [ ] Conflicts are detected and displayed
- [ ] Device overlap scenario creates conflict
- [ ] Worker overlap scenario creates conflict
- [ ] Duplicate sync handled correctly
- [ ] Clear DB button resets local data

---

## Troubleshooting

### Backend Won't Connect

**Error:** "Error loading data: Connection refused"

**Solution:**
1. Check backend is running: `node src/server.js`
2. Verify correct backend URL in `lib/main.dart`
3. For physical device: ensure phone and computer on same Wi-Fi
4. Test connectivity: open `http://<your-computer-ip>:3000/api/sessions` in phone browser

### No Data in Supervisor View

**Error:** Sessions list is empty

**Solution:**
1. Create work sessions first (go back to worker screen)
2. Click "Sync Now" to upload events
3. Refresh supervisor view
4. Check backend console for errors

### SQLite Database Locked

**Error:** "database is locked"

**Solution:**
1. Wait for sync to complete
2. Close app and reopen
3. Restart backend: `node src/server.js`
4. Delete database file: `backend/data/factory_audit_server.db*`

### Flutter Import Errors

**Error:** "File not found" for imports

**Solution:**
1. Run `flutter pub get`
2. Verify imports match your file structure (case-sensitive)
3. Restart IDE

---

## Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - System design, event sourcing, conflict detection rules, database schema, API contract
- **[DESIGN_DECISIONS.md](DESIGN_DECISIONS.md)** - Key architectural choices, tradeoffs, and where AI led you wrong
- **[ASSUMPTIONS.md](ASSUMPTIONS.md)** - MVP scope, constraints, and why they matter

---

## Deployment Notes

### For Demo

1. Backend: `cd backend && node src/server.js`
2. Configure backend URL in Flutter (see Quick Start)
3. Reset data:
   - Stop backend
   - Delete: `backend/data/factory_audit_server.db*`
   - Restart backend
   - Use "Clear DB" button in app

### For Production

1. **Authentication** - Add user login/verification
2. **Database** - Migrate from SQLite to PostgreSQL
3. **Security** - Add HTTPS, API tokens, rate limiting
4. **Scaling** - Load balancer, database replicas
5. **Monitoring** - Logging, error tracking, metrics

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed production considerations.

---

## Learning Outcomes

This project demonstrates:

**Flutter Concepts**
- State management and lifting state
- Navigation patterns (Router)
- Local database (SQLite)
- HTTP networking
- Error handling
- UI patterns (cards, lists)

**Architecture Concepts**
- Offline-first design
- Eventual consistency
- Conflict detection
- Event sourcing
- Immutable audit records
- Device handoff models

**Backend Concepts**
- Express.js API design
- Validation and error handling
- Complex business logic (conflict detection)
- Database design (append-only logs)
- Idempotent operations

**DevOps Concepts**
- Frontend-backend communication
- Environment configuration
- Testing and debugging
- Database migrations

**Relevant to:**
- Flutter Certified Associate
- Google Cloud Associate
- Firebase Certified exams

---

## License

This project is provided as-is for educational and demonstration purposes.

---

**Questions?** See the documentation files above or review the code comments.
