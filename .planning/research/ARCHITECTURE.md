# Architecture Patterns: Mallow

**Domain:** Social focus mobile app — dual-engine (coordination + deep work)
**Researched:** 2026-04-20
**Confidence:** HIGH (stack patterns well-established; Mallow-specific economy logic is original design)

---

## Recommended Architecture

Mallow is a **client-server mobile app** with five distinct system components. There is no peer-to-peer layer — all state flows through the server. The design is intentionally boring in infrastructure so the product complexity can live in the domain logic (Astrophage economy, Launch Window detection, anti-burnout rules).

```
┌─────────────────────────────────────────────┐
│                Flutter App                  │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │ Heatmap  │ │   Lab    │ │  Campfire   │ │
│  │  Screen  │ │  Screen  │ │    Room     │ │
│  └────┬─────┘ └────┬─────┘ └──────┬──────┘ │
│       └────────────┴──────────────┘         │
│              Riverpod State Layer            │
│         (polling loop + local cache)         │
└──────────────────┬──────────────────────────┘
                   │  HTTPS REST (30-60s poll)
                   │
┌──────────────────▼──────────────────────────┐
│              API Server (Node.js)            │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │  Auth    │ │  Crew    │ │  Economy    │ │
│  │ /sessions│ │ /heatmap │ │ /astrophage │ │
│  └──────────┘ └──────────┘ └─────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │   Lab    │ │Campfire  │ │ Notification│ │
│  │/sessions │ │  /rooms  │ │   Service   │ │
│  └──────────┘ └──────────┘ └──────┬──────┘ │
└──────────────────┬─────────────────┼────────┘
                   │                 │ FCM / APNs
┌──────────────────▼──────────────────────────┐
│               PostgreSQL                     │
│  users │ crews │ memberships │ heatmap_slots │
│  lab_sessions │ astrophage_ledger │ hangouts │
│  campfire_rooms │ push_tokens │ invites      │
└─────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│           Background Job Runner              │
│           (BullMQ + Redis)                   │
│  ┌─────────────────┐  ┌────────────────────┐│
│  │ Launch Window   │  │ Anti-Burnout       ││
│  │ Scanner (60s)   │  │ Checker (5min)     ││
│  └─────────────────┘  └────────────────────┘│
│  ┌─────────────────┐  ┌────────────────────┐│
│  │ Session Grace   │  │ Astrophage          ││
│  │ Period Enforcer │  │ Balance Snapshot    ││
│  └─────────────────┘  └────────────────────┘│
└─────────────────────────────────────────────┘
```

---

## Component Boundaries

| Component | Responsibility | Communicates With | State Owned |
|-----------|---------------|-------------------|-------------|
| Flutter App | All UI, local state, polling loop, haptic triggers | API Server (HTTPS), FCM (receive only) | Session-local UI state, cached crew snapshot |
| API Server | Business logic, auth, economy rules, crew management | PostgreSQL, BullMQ, FCM Admin SDK | None — stateless between requests |
| PostgreSQL | Source of truth for all persistent state | API Server, Background Jobs | All durable data |
| BullMQ + Redis | Scheduled background jobs, job persistence across restarts | API Server (enqueue), PostgreSQL (read), FCM Admin SDK (send) | Job queues, schedule metadata |
| FCM / APNs | Delivery of push notifications to devices | API Server and BullMQ Workers (send), Flutter App (receive) | Device token routing (managed by Firebase/Apple) |

**Critical boundary rule:** The Flutter app never computes Astrophage balances. It requests them. All economy logic runs server-side and is authoritative. The app renders what the server says.

---

## Data Flow: Astrophage from Focus to Hangout Unlock

This is the core loop. Every step is explicit.

```
1. USER STARTS LAB SESSION
   Flutter → POST /lab/sessions
   Server → inserts lab_sessions row {user_id, crew_id, discipline, started_at, status: 'active'}
   Server → BullMQ enqueues grace-period watchdog job (delay: 5min)

2. USER IS IN LAB (polling)
   Flutter polls GET /crew/{id}/status every 30s
   Server → returns crew snapshot: {who is in Lab, current Astrophage balances, heatmap state}
   Flutter renders orbiting ships on crew map

3. USER LEAVES APP (background)
   Flutter detects AppLifecycleState.paused → sends POST /lab/sessions/{id}/pause
   Server → records paused_at timestamp
   BullMQ grace-period job fires after 5 min if no resume received

4. GRACE PERIOD EXPIRES
   BullMQ worker → reads lab_sessions where status='paused' and paused_at < now()-5min
   Worker → computes earned Astrophage: floor(duration_minutes × discipline_multiplier)
   Worker → writes to astrophage_ledger (append-only, never UPDATE balances directly)
     INSERT astrophage_ledger {user_id, crew_id, amount, reason: 'lab_session', session_id, created_at}
   Worker → updates lab_sessions row: {status: 'closed', ended_at, astrophage_earned}

5. FIST-BUMP (haptic trigger)
   Flutter → POST /crew/{id}/fistbump {target_user_id}
   Server → validates both users are in same crew and both active in Lab
   Server → looks up target_user's push_token
   Server → sends FCM data-only message {type: 'fistbump', from_user_id} to target device
   Target Flutter app → receives FCM data message in background handler
   Target Flutter app → triggers HapticFeedback.heavyImpact() ×3

6. LAUNCH WINDOW DETECTION (background job, every 60s)
   BullMQ repeatable job → reads heatmap_slots for all crew members for next 3 hours
   Job → calculates overlap: what % of crew has 'free' blocks in same time window
   If overlap >= 80%:
     Job → inserts launch_windows row {crew_id, window_start, window_end, triggered_at}
     Job → sends FCM multicast to all crew push_tokens
       Notification: "Stable Orbit Detected — X hours until you can gather"
     Job → creates campfire_rooms row {crew_id, launch_window_id, status: 'open'}

7. CREW POOLS ASTROPHAGE TO UNLOCK HANGOUT
   Leader → POST /campfire/{room_id}/proposals {description, astrophage_cost}
   Server → inserts hangout_proposals row
   Members → POST /campfire/{room_id}/proposals/{id}/contribute {amount}
   Server → validates user has sufficient balance (SUM astrophage_ledger for user)
   Server → appends DEBIT to astrophage_ledger {amount: -N, reason: 'hangout_unlock', proposal_id}
   Server → checks if proposal fully funded → marks hangout as 'unlocked'
   Server → sends FCM multicast to crew: "Hangout unlocked — the Campfire is lit"
```

---

## Database Schema

### Principles
- All Astrophage balance queries derive from `astrophage_ledger` (append-only, double-entry style). Never store a mutable `balance` column.
- All timestamps in UTC, stored as `TIMESTAMPTZ`.
- Soft deletes only — no hard DELETE on user or crew data.
- Use PostgreSQL `BIGINT` for IDs generated server-side (not UUIDs in URLs — use short IDs or slugs for invite links).

### Tables

```sql
-- Identity
CREATE TABLE users (
  id            BIGSERIAL PRIMARY KEY,
  display_name  TEXT NOT NULL,
  avatar_seed   TEXT,                    -- used to derive avatar appearance
  created_at    TIMESTAMPTZ DEFAULT now(),
  deleted_at    TIMESTAMPTZ              -- soft delete
);

CREATE TABLE push_tokens (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT REFERENCES users(id) NOT NULL,
  token       TEXT NOT NULL,
  platform    TEXT CHECK (platform IN ('ios','android')) NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT now(),
  last_seen   TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, platform)              -- one active token per platform per user
);

-- Crew & Social Graph
CREATE TABLE crews (
  id           BIGSERIAL PRIMARY KEY,
  name         TEXT NOT NULL,
  emblem_id    TEXT,                     -- cosmetic, references cosmetics catalog
  created_at   TIMESTAMPTZ DEFAULT now(),
  deleted_at   TIMESTAMPTZ
);

CREATE TABLE crew_memberships (
  id         BIGSERIAL PRIMARY KEY,
  crew_id    BIGINT REFERENCES crews(id) NOT NULL,
  user_id    BIGINT REFERENCES users(id) NOT NULL,
  role       TEXT CHECK (role IN ('leader','member')) DEFAULT 'member',
  joined_at  TIMESTAMPTZ DEFAULT now(),
  left_at    TIMESTAMPTZ,               -- soft remove
  UNIQUE(crew_id, user_id)
);

CREATE TABLE crew_invites (
  id          BIGSERIAL PRIMARY KEY,
  crew_id     BIGINT REFERENCES crews(id) NOT NULL,
  invite_code TEXT UNIQUE NOT NULL,     -- short random slug, used in deep link
  created_by  BIGINT REFERENCES users(id) NOT NULL,
  expires_at  TIMESTAMPTZ,
  used_at     TIMESTAMPTZ,
  used_by     BIGINT REFERENCES users(id)
);

-- Heatmap (availability)
-- Slots are 30-minute buckets. slot_start is always on the :00 or :30.
-- Storing explicit free/busy rather than inferring avoids complex absence logic.
CREATE TABLE heatmap_slots (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT REFERENCES users(id) NOT NULL,
  crew_id     BIGINT REFERENCES crews(id) NOT NULL,
  slot_start  TIMESTAMPTZ NOT NULL,
  status      TEXT CHECK (status IN ('free','busy')) NOT NULL,
  updated_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, crew_id, slot_start)  -- upsert target
);

-- Launch Windows (detected by background job)
CREATE TABLE launch_windows (
  id               BIGSERIAL PRIMARY KEY,
  crew_id          BIGINT REFERENCES crews(id) NOT NULL,
  window_start     TIMESTAMPTZ NOT NULL,
  window_end       TIMESTAMPTZ NOT NULL,
  overlap_pct      NUMERIC(5,2),        -- e.g. 85.00
  triggered_at     TIMESTAMPTZ DEFAULT now(),
  notification_sent BOOLEAN DEFAULT false
);

-- Lab Sessions (deep work)
CREATE TABLE lab_sessions (
  id                BIGSERIAL PRIMARY KEY,
  user_id           BIGINT REFERENCES users(id) NOT NULL,
  crew_id           BIGINT REFERENCES crews(id) NOT NULL,
  discipline        TEXT CHECK (discipline IN ('astrogeology','astrophysics','xenobiology')) NOT NULL,
  multiplier        NUMERIC(4,2) NOT NULL,    -- snapshot at session start
  started_at        TIMESTAMPTZ DEFAULT now(),
  paused_at         TIMESTAMPTZ,
  ended_at          TIMESTAMPTZ,
  status            TEXT CHECK (status IN ('active','paused','closed')) DEFAULT 'active',
  astrophage_earned BIGINT,                   -- populated on close
  burnout_flag      BOOLEAN DEFAULT false     -- set if 6hr limit triggered
);

-- Astrophage Economy (append-only ledger)
CREATE TABLE astrophage_ledger (
  id           BIGSERIAL PRIMARY KEY,
  user_id      BIGINT REFERENCES users(id) NOT NULL,
  crew_id      BIGINT REFERENCES crews(id) NOT NULL,
  amount       BIGINT NOT NULL,          -- positive = earn, negative = spend (in mg)
  reason       TEXT NOT NULL,            -- 'lab_session' | 'hangout_unlock' | 'calibration_bonus'
  reference_id BIGINT,                   -- lab_sessions.id or hangout_proposals.id
  created_at   TIMESTAMPTZ DEFAULT now()
);
-- Balance query: SELECT COALESCE(SUM(amount), 0) FROM astrophage_ledger WHERE user_id=$1 AND crew_id=$2

-- Campfire & Hangouts
CREATE TABLE campfire_rooms (
  id               BIGSERIAL PRIMARY KEY,
  crew_id          BIGINT REFERENCES crews(id) NOT NULL,
  launch_window_id BIGINT REFERENCES launch_windows(id),
  status           TEXT CHECK (status IN ('open','closed')) DEFAULT 'open',
  opened_at        TIMESTAMPTZ DEFAULT now(),
  closed_at        TIMESTAMPTZ
);

CREATE TABLE hangout_proposals (
  id                  BIGSERIAL PRIMARY KEY,
  campfire_room_id    BIGINT REFERENCES campfire_rooms(id) NOT NULL,
  proposed_by         BIGINT REFERENCES users(id) NOT NULL,   -- must be crew leader
  description         TEXT NOT NULL,
  astrophage_cost     BIGINT NOT NULL,                         -- total cost in mg
  astrophage_pledged  BIGINT DEFAULT 0,                        -- running total
  status              TEXT CHECK (status IN ('open','funded','cancelled')) DEFAULT 'open',
  created_at          TIMESTAMPTZ DEFAULT now(),
  funded_at           TIMESTAMPTZ
);
```

### Indexes (performance-critical queries)

```sql
-- Polling: crew status snapshot
CREATE INDEX idx_lab_sessions_crew_active ON lab_sessions(crew_id, status) WHERE status = 'active';
CREATE INDEX idx_heatmap_slots_crew_time ON heatmap_slots(crew_id, slot_start);

-- Launch Window scan
CREATE INDEX idx_heatmap_slots_user_crew_time ON heatmap_slots(user_id, crew_id, slot_start);

-- Astrophage balance
CREATE INDEX idx_astrophage_ledger_user_crew ON astrophage_ledger(user_id, crew_id);

-- Grace period enforcer
CREATE INDEX idx_lab_sessions_paused ON lab_sessions(paused_at) WHERE status = 'paused';
```

---

## API Design

**Verdict: REST (not tRPC, not GraphQL).**

Rationale:
- Flutter is not TypeScript — tRPC's primary benefit (end-to-end type sharing) does not apply to a Dart client. tRPC would add complexity with no payoff.
- GraphQL is justified when clients need flexible field selection across complex graphs. Mallow's screens have fixed data requirements — crew status, heatmap, Astrophage balance. Over-fetching is not a real problem here.
- REST is universally understood, toolable (Postman, curl, OpenAPI), and the 2-3 person team can move fast without a schema layer.
- At Mallow's scale (hundreds to low thousands of users for v1), REST performance concerns do not apply.

**Pattern: RESTful resources with versioned prefix `/api/v1/`**

```
Auth
  POST   /api/v1/auth/register
  POST   /api/v1/auth/login
  DELETE /api/v1/auth/session        (logout — invalidates token)
  PUT    /api/v1/auth/push-token     (upsert FCM/APNs token)

Crews
  POST   /api/v1/crews               (create)
  GET    /api/v1/crews/:id           (detail + my membership)
  GET    /api/v1/crews/:id/status    (POLLING ENDPOINT — crew snapshot)
  POST   /api/v1/crews/:id/invites   (generate invite link)
  POST   /api/v1/invites/:code/accept

Heatmap
  GET    /api/v1/crews/:id/heatmap              (full crew heatmap, 7-day window)
  PUT    /api/v1/crews/:id/heatmap/me           (bulk upsert my slots)

Lab
  POST   /api/v1/lab/sessions                   (start session)
  POST   /api/v1/lab/sessions/:id/pause         (app goes background)
  POST   /api/v1/lab/sessions/:id/resume        (app returns to foreground)
  POST   /api/v1/lab/sessions/:id/end           (user manually ends)

Social
  POST   /api/v1/crews/:id/fistbump             (body: {target_user_id})

Campfire
  GET    /api/v1/crews/:id/campfire             (active room, if any)
  POST   /api/v1/campfire/:room_id/proposals
  POST   /api/v1/campfire/:room_id/proposals/:id/contribute  (body: {amount})

Economy
  GET    /api/v1/crews/:id/astrophage/me        (my balance in this crew)
  GET    /api/v1/crews/:id/astrophage/ledger    (recent transactions, paginated)
```

**Auth:** JWT Bearer tokens. Refresh token pattern (short-lived access token, longer-lived refresh). Store push tokens on login/token-refresh.

**Polling endpoint contract:** `GET /api/v1/crews/:id/status` returns a single JSON object with all fields the app needs to render the crew map and Astrophage bar. This avoids N+1 polling calls from the client.

```json
{
  "crew_id": 42,
  "members": [
    {
      "user_id": 7,
      "display_name": "Yuki",
      "in_lab": true,
      "discipline": "astrogeology",
      "lab_started_at": "2026-04-20T14:00:00Z",
      "heatmap_status_now": "busy"
    }
  ],
  "crew_astrophage_pool": 4320,
  "my_astrophage_balance": 810,
  "active_launch_window": null,
  "active_campfire_room_id": null,
  "fetched_at": "2026-04-20T14:03:22Z"
}
```

---

## Background Job Needs

Use **BullMQ** backed by Redis. BullMQ supports repeatable jobs with cron expressions, persists jobs across server restarts (critical — cron memory loss is a known footgun), and has mature TypeScript support.

| Job | Schedule | What It Does |
|-----|----------|--------------|
| `launch-window-scanner` | Every 60s | Reads heatmap_slots for all active crews for next 3hr window. If any crew hits 80%+ free overlap, creates launch_windows row, sends FCM multicast, creates campfire_rooms row. Idempotent — checks if window already triggered for this time block. |
| `grace-period-enforcer` | Every 30s | Reads lab_sessions WHERE status='paused' AND paused_at < now()-5min. Closes each session, computes Astrophage earned (floor(minutes × multiplier)), appends to astrophage_ledger. |
| `anti-burnout-checker` | Every 5min | Reads lab_sessions WHERE status='active' AND started_at < now()-6hr AND burnout_flag=false. Sets burnout_flag=true, sends FCM to the overworking user + a gentle notification to crew. |
| `push-token-cleanup` | Daily (03:00 UTC) | Removes push_tokens where last_seen > 90 days. FCM/APNs will return errors for stale tokens — handle those errors in-line too. |

**Grace period correctness note:** The client sends `/pause` when going background; it does not rely on the server detecting absence. This makes the 5-minute grace period explicit and auditable in the database (`paused_at` column), rather than inferred from polling gaps.

---

## Suggested Build Order

Dependencies drive this order. Each layer must exist before the next can be tested.

```
Phase 1 — Foundation (nothing works without this)
  ├── PostgreSQL schema + migrations (Flyway or db-migrate)
  ├── API server skeleton (Express or Fastify + JWT auth)
  ├── User registration + login
  ├── Push token registration endpoint
  └── Flutter app scaffold (Riverpod, routing, auth screens)

Phase 2 — Crew & Social Graph (required before any crew features)
  ├── Crew create + invite link generation
  ├── Invite accept flow (deep link → join crew)
  ├── Crew membership queries
  └── Flutter: invite flow + crew home screen

Phase 3 — Heatmap & Polling (core coordination engine)
  ├── Heatmap CRUD endpoints
  ├── Crew status polling endpoint (the compound snapshot)
  ├── Flutter: heatmap UI + 30s polling loop (Riverpod timer)
  └── Flutter: crew map with member presence

Phase 4 — Lab & Astrophage Economy (earn loop)
  ├── Lab session start/pause/resume/end endpoints
  ├── Astrophage ledger (append-only writes + balance query)
  ├── BullMQ + Redis setup
  ├── Grace period enforcer job
  ├── Anti-burnout checker job
  └── Flutter: Lab screen + discipline selector + session timer

Phase 5 — Push Notifications & Launch Window (the money feature)
  ├── FCM Admin SDK integration (Node.js firebase-admin)
  ├── APNs certificate setup
  ├── Launch Window scanner job
  ├── Fist-bump haptic push endpoint
  └── Flutter: FCM receive handler + haptic trigger + launch window UI

Phase 6 — Campfire & Hangout Unlock (spend loop)
  ├── Campfire room lifecycle endpoints
  ├── Hangout proposal + Astrophage contribute endpoints
  ├── Crew leader role enforcement
  └── Flutter: Campfire room screen + proposal UI

Phase 7 — Cosmetics & Onboarding (retention layer)
  ├── Cosmetics catalog (static JSON to start, no DB needed)
  ├── Onboarding flow (Black Void → Transmission → Calibration Focus)
  └── Calibration Focus → earns first Astrophage via normal lab session flow
```

**Why this order matters:**
- Push notifications (Phase 5) need FCM tokens which need auth (Phase 1).
- The Astrophage economy (Phase 4) must be stable before the spend side (Phase 6) can be built.
- Launch Window (Phase 5) requires heatmap data (Phase 3) to exist first.
- Fist-bump is in Phase 5 because it uses the FCM infrastructure set up there, not because it is complex.
- Onboarding (Phase 7) is last because it wraps Phase 4 (first Astrophage earn) — implementing it earlier means rewriting it as features land.

---

## Flutter App Architecture

**Use Riverpod (not BLoC) for this project.**

Rationale: Mallow has a polling loop (timer-driven async state), not a stream-heavy event bus. Riverpod's `AsyncNotifier` and `StreamProvider` handle polling elegantly. BLoC adds boilerplate discipline that a 2-3 person team does not need for this scope. Riverpod is the 2026 default for mid-scale Flutter apps.

```
lib/
├── main.dart
├── app/
│   ├── router.dart                 (go_router)
│   └── theme.dart
├── features/
│   ├── auth/
│   │   ├── data/          (API calls)
│   │   ├── domain/        (User model)
│   │   └── presentation/  (screens + providers)
│   ├── crew/
│   │   ├── data/          (crew_repository.dart — wraps polling endpoint)
│   │   ├── domain/        (CrewStatus, Member models)
│   │   └── presentation/  (crew_map_screen, heatmap_screen)
│   ├── lab/
│   │   ├── data/
│   │   ├── domain/        (LabSession, Discipline, AstrophageBalance)
│   │   └── presentation/  (lab_screen, discipline_selector)
│   ├── campfire/
│   │   ├── data/
│   │   ├── domain/        (CampfireRoom, HangoutProposal)
│   │   └── presentation/  (campfire_screen)
│   └── notifications/
│       └── fcm_handler.dart       (receive FCM, trigger haptics, route to screen)
└── shared/
    ├── api_client.dart            (Dio + auth interceptor)
    ├── polling_provider.dart      (shared 30s timer logic)
    └── widgets/
```

**Polling implementation:** A single `crewStatusProvider` (Riverpod `AsyncNotifier`) holds the crew snapshot and refreshes via a periodic timer (30s). All crew-dependent screens watch this one provider. This avoids duplicated polling timers.

**Background lifecycle:** Listen to `AppLifecycleState` via `WidgetsBindingObserver`. On `paused` → call `/lab/sessions/:id/pause`. On `resumed` → call `/lab/sessions/:id/resume` and restart polling timer.

**FCM data-only messages** (used for fist-bump haptic): Configure `FirebaseMessaging.onBackgroundMessage` handler. When `type == 'fistbump'`, call `HapticFeedback.heavyImpact()` three times with 150ms gaps. No notification shown — pure tactile signal.

---

## Node.js vs Go

**Use Node.js for v1.**

Rationale:
- Node.js with TypeScript (Fastify or Express) and BullMQ is a well-understood, cohesive stack. The FCM Admin SDK is first-party on Node.js.
- Go offers better raw performance but the team is small (2-3) and Go's ecosystem for rapid product iteration (ORMs, queue libraries, admin SDKs) is thinner. The performance difference does not matter at Mallow's scale.
- Node.js allows the same language across backend logic and BullMQ worker processes.
- Migrate to Go later if profiling reveals a real bottleneck — v1 will not have this problem.

**Recommended Node.js stack:**
- Fastify (faster than Express, built-in schema validation with JSON Schema)
- Drizzle ORM (TypeScript-native, lightweight, no magic — good fit for an append-only ledger)
- BullMQ + Redis (background jobs)
- firebase-admin (FCM multicast)
- node-apn or firebase-admin (APNs — FCM handles both platforms via a single API in HTTP v1)
- Zod (runtime validation at API boundaries)

---

## Scalability Considerations

| Concern | At 100 users (v1) | At 10K users | At 100K users |
|---------|-------------------|--------------|---------------|
| Crew status polling | Single API server, no cache needed | Add Redis cache for crew snapshot (TTL 25s) | CDN edge caching, horizontal API scaling |
| Launch Window scan | Full table scan on heatmap_slots is fine | Partition heatmap_slots by crew_id | Dedicated scan worker pool, per-crew scheduling |
| Astrophage ledger | SUM query on small table is instant | Add materialized balance cache updated on each ledger write | Event-driven balance updates with optimistic locking |
| Push delivery | Firebase handles delivery; backend just sends | Same | Rate limit outbound FCM calls, batch multicast |
| Background jobs | Single BullMQ worker process | Multiple workers with concurrency config | Job partitioning by crew shard |

For v1: deploy API server and BullMQ worker as two separate processes on the same machine (or two small containers). Separate them so the worker can be scaled independently later.

---

## Architecture Anti-Patterns to Avoid

### Anti-Pattern 1: Mutable Astrophage Balance Column
**What:** Adding a `balance` column to `users` or `crew_memberships` and UPDATEing it.
**Why bad:** Race conditions on concurrent lab session closes corrupt balances silently. Impossible to audit. Cannot reconstruct history.
**Instead:** Append-only ledger. Balance = `SELECT SUM(amount) FROM astrophage_ledger WHERE user_id=$1 AND crew_id=$2`. Cache the result, never store it.

### Anti-Pattern 2: Client-Side Economy Computation
**What:** Flutter computing Astrophage earned from session duration.
**Why bad:** Trivially exploitable. Clocks differ between client and server. Grace period logic belongs to the server.
**Instead:** Server closes sessions and appends ledger entries. Client only displays what server returns.

### Anti-Pattern 3: WebSockets for Crew Status
**What:** Opening a persistent WebSocket connection per crew member for live status.
**Why bad:** Significantly higher operational complexity, connection state management, reconnect logic, server statefulness. For 30-60s freshness requirements, polling is strictly simpler with identical UX.
**Instead:** REST polling. The crew status endpoint is cheap and stateless.

### Anti-Pattern 4: In-Memory Cron for Background Jobs
**What:** Using `setInterval` or node-cron without persistence.
**Why bad:** Server restart wipes all scheduled jobs. Grace periods silently never fire. Launch Windows are missed.
**Instead:** BullMQ repeatable jobs with Redis persistence. Jobs survive server restart.

### Anti-Pattern 5: Storing Full Heatmap as JSON Blob
**What:** Saving a week's availability as `{user_id, schedule_json}` in a single row.
**Why bad:** Impossible to query ("who is free Tuesday at 3pm across all crew members") without loading and parsing all blobs. Launch Window scanner becomes O(N) application-layer computation.
**Instead:** Normalized `heatmap_slots` rows with indexed `(crew_id, slot_start)`. Launch Window scan is a SQL aggregate query.

---

## Sources

- Flutter architecture: [codewithandrea.com — Flutter App Architecture with Riverpod](https://codewithandrea.com/articles/flutter-app-architecture-riverpod-introduction/)
- Flutter state management 2026: [samioda.com — Flutter State Management in 2026](https://samioda.com/en/blog/flutter-state-management-2026)
- Flutter background lifecycle: [Flutter API — AppLifecycleListener](https://api.flutter.dev/flutter/widgets/AppLifecycleListener-class.html)
- BullMQ background jobs: [BullMQ docs](https://docs.bullmq.io/) | [Better Stack — BullMQ scheduled tasks](https://betterstack.com/community/guides/scaling-nodejs/bullmq-scheduled-tasks/)
- FCM Node.js: [Firebase — Receive messages in Flutter](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages) | [Medium — FCM HTTP V1 Node.js 2026](https://medium.com/@rhythm6194/send-fcm-push-notification-in-node-js-using-firebase-cloud-messaging-fcm-http-v1-2024-448c0d921fff)
- Append-only ledger: [architecture-weekly.com — Building your own Ledger Database](https://www.architecture-weekly.com/p/building-your-own-ledger-database) | [Medium — Append-Only Ledger in NoSQL](https://ahmed-waseem.medium.com/building-an-append-only-ledger-in-nosql-caebd3786193)
- REST vs tRPC vs GraphQL 2026: [dev.to — tRPC vs REST vs GraphQL SaaS Builder's Take](https://dev.to/whoffagents/trpc-vs-rest-vs-graphql-in-2026-a-saas-builders-honest-take-459k) | [pockit.tools — Definitive Guide](https://pockit.tools/blog/rest-graphql-trpc-grpc-api-comparison-2026/)
- Node.js vs Go: [uptech.team — Node.js vs Go](https://www.uptech.team/blog/nodejs-vs-golang-comparison)
- haptic + FCM data-only: [Firebase Flutter docs — Receive messages](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages)
