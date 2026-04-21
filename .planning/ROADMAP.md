# Roadmap — Mallow v1.0 MVP

> 7 phases | 25 requirements | 3-person team

---

## Phase Overview

| # | Phase | Goal | Reqs | Effort |
|---|-------|------|------|--------|
| 1 | Foundation | Auth, DB schema, API skeleton, and Flutter scaffold are in place so every subsequent phase has something to build on | FOUND-01, FOUND-02, FOUND-03 | 4 days |
| 2 | Crew & Social Graph | Users can create crews and invite friends so the social layer exists before any crew-dependent features are built | CREW-01, CREW-02, CREW-03, CREW-04 | 4 days |
| 3 | Heatmap & Polling | Users can mark availability and see their crew's combined schedule in real time | HEAT-01, HEAT-02 | 4 days |
| 4 | Lab & Astrophage Economy | Users can run focus sessions that earn Astrophage, with background jobs enforcing grace periods and anti-burnout rules | LAB-01, LAB-02, LAB-03, LAB-04, APH-01, APH-02 | 6 days |
| 5 | Push Notifications & Launch Window | The app detects when the crew is free and fires a native push notification, and friends can fist-bump each other via haptic | NOTIF-01, NOTIF-02, NOTIF-03 | 5 days |
| 6 | Campfire & Hangout Unlock | A Campfire room opens on a Launch Window and the crew can propose and fund hangouts with Astrophage | CAMP-01, CAMP-02, CAMP-03 | 5 days |
| 7 | Cosmetics & Onboarding | New users land in the narrative onboarding flow and earn their first Astrophage; a cosmetics catalog lets the crew spend together | ONB-01, ONB-02, COS-01, COS-02 | 4 days |

**Total estimated parallel effort: ~32 days (3 people working in parallel)**

---

## Phase 1: Foundation

**Goal:** Auth, DB schema, API skeleton, and Flutter scaffold are in place so every subsequent phase has something to build on.
**Requirements:** FOUND-01, FOUND-02, FOUND-03
**Estimated effort:** 4 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Initialize Flutter project with Riverpod, go_router, and Dio dependencies
- [ ] Set up `app/router.dart` with placeholder routes for all major screens
- [ ] Set up `app/theme.dart` with space/Nomai color palette and text styles
- [ ] Build register screen (display name input, submit button)
- [ ] Build login screen (display name / password, submit)
- [ ] Implement `AuthNotifier` (Riverpod) that stores JWT access + refresh tokens in secure storage
- [ ] Implement `api_client.dart` (Dio) with auth interceptor that attaches Bearer token to every request
- [ ] On login success: call push-token registration endpoint (stub until FCM wired in Phase 5)
- [ ] Wire auth state to router guard (redirect to login if unauthenticated)

**Person B — Backend API**
- [ ] Initialize Fastify + TypeScript project with Zod validation and Drizzle ORM
- [ ] Write and apply initial DB migration: `users`, `push_tokens` tables (schema from ARCHITECTURE.md)
- [ ] `POST /api/v1/auth/register` — creates user row, returns JWT pair
- [ ] `POST /api/v1/auth/login` — validates credentials, returns JWT pair
- [ ] `DELETE /api/v1/auth/session` — logout, invalidates refresh token
- [ ] `PUT /api/v1/auth/push-token` — upserts push_tokens row
- [ ] JWT middleware (verify access token, attach `req.user` to context)
- [ ] Global error handler and health-check route (`GET /health`)

**Person C — Infra/Jobs** *(less job work this phase — assist backend)*
- [ ] Set up PostgreSQL instance (local Docker Compose for dev, note prod target)
- [ ] Set up Redis instance in Docker Compose (ready for BullMQ in Phase 4)
- [ ] Configure Drizzle migration runner and confirm schema applies cleanly
- [ ] Set up environment variable schema (dotenv + Zod) shared by API and future worker process
- [ ] Write `docker-compose.yml` covering Postgres, Redis, API server
- [ ] Set up CI pipeline: lint, type-check, and migration dry-run on every push

### Success Criteria
1. A new user can register with a display name and receive a valid JWT pair from the API.
2. A registered user can log in, and the Flutter app persists the token across cold restarts.
3. The Flutter app redirects unauthenticated users to the login screen and authenticated users past it.
4. The API returns a structured error (not a 500) for invalid credentials.
5. `docker-compose up` starts Postgres, Redis, and the API server with migrations applied automatically.

---

## Phase 2: Crew & Social Graph

**Goal:** Users can create crews and invite friends so the social layer exists before any crew-dependent features are built.
**Requirements:** CREW-01, CREW-02, CREW-03, CREW-04
**Estimated effort:** 4 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Build crew creation screen (crew name input, submit)
- [ ] Build crew home screen showing member list, roles (leader badge), and placeholder sections for Heatmap and Lab
- [ ] Implement deep-link handling (go_router) for invite URLs (`mallow://invite/:code`)
- [ ] Build invite accept screen — shows crew name and "Join Crew" CTA, shown when deep link opens
- [ ] Show each member's privacy-respecting availability shape (busy/free indicator, no reason shown — Ghost Matter Filter)
- [ ] Implement crew membership Riverpod provider polling `GET /api/v1/crews/:id`

**Person B — Backend API**
- [ ] Write and apply DB migration: `crews`, `crew_memberships`, `crew_invites` tables
- [ ] `POST /api/v1/crews` — create crew, auto-assign creator as leader
- [ ] `GET /api/v1/crews/:id` — crew detail + calling user's membership info
- [ ] `POST /api/v1/crews/:id/invites` — generate short invite code, return deep-link URL
- [ ] `POST /api/v1/invites/:code/accept` — validate code, add user as member
- [ ] Leader role enforcement middleware — validates `req.user` is leader before leader-only actions
- [ ] `GET /api/v1/crews/:id/status` — initial skeleton (returns members list; heatmap + Astrophage fields stubbed as null for now)

**Person C — Infra/Jobs** *(less job work this phase — assist Flutter)*
- [ ] Implement Flutter deep-link configuration for iOS (`Associated Domains`) and Android (`App Links` intent filter)
- [ ] Set up Flutter flavor/environment config (dev vs prod base URL)
- [ ] Help Person A wire the invite accept flow end-to-end (link received → screen shown → API call → crew home)
- [ ] Validate deep-link routing works on both iOS simulator and Android emulator

### Success Criteria
1. A user can create a crew and see themselves listed as leader on the crew home screen.
2. Sharing an invite link and opening it on another device lands the second user in the invite accept screen.
3. After accepting, the new member appears in the crew member list.
4. Crew members can see each other's busy/free status but never any reason or event title (Ghost Matter Filter enforced at API and UI layers).
5. Crew leader is visually distinguished; non-leaders cannot access leader-only actions.

---

## Phase 3: Heatmap & Polling

**Goal:** Users can mark availability and see their crew's combined schedule in real time.
**Requirements:** HEAT-01, HEAT-02
**Estimated effort:** 4 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Build Heatmap screen: 7-day grid of 30-minute time blocks, scrollable
- [ ] Tap interaction: toggle a block between Busy (In the Lab) and Free (Oxygen Flowing) — optimistic UI update, then API write
- [ ] Render crew overlay: show aggregated crew availability shape per time block using color intensity
- [ ] Implement `crewStatusProvider` — Riverpod `AsyncNotifier` with 30-second periodic refresh of `GET /api/v1/crews/:id/status`
- [ ] Crew map screen: display orbiting ships for members currently in Lab (placeholder ships using discipline color; real session data comes in Phase 4)
- [ ] Handle polling errors gracefully: show stale-data banner if last refresh > 90 seconds ago

**Person B — Backend API**
- [ ] Write and apply DB migration: `heatmap_slots` table + `idx_heatmap_slots_crew_time` index
- [ ] `PUT /api/v1/crews/:id/heatmap/me` — bulk upsert caller's heatmap slots (accepts array of `{slot_start, status}`)
- [ ] `GET /api/v1/crews/:id/heatmap` — return full crew heatmap for a 7-day window (grouped by user, respecting Ghost Matter Filter: status only, no context)
- [ ] Expand `GET /api/v1/crews/:id/status` to populate `members[].heatmap_status_now` from heatmap_slots

**Person C — Infra/Jobs** *(less job work this phase — assist backend)*
- [ ] Add database indexes: `idx_heatmap_slots_crew_time`, `idx_heatmap_slots_user_crew_time`
- [ ] Write integration test covering bulk upsert idempotency (same slot_start written twice → one row, not two)
- [ ] Confirm polling endpoint latency is under 200ms at 50 concurrent simulated requests
- [ ] Help Person B with heatmap aggregation query logic (SQL window over crew members)

### Success Criteria
1. A user can tap a time block on their Heatmap and the block toggles state; the change persists after app restart.
2. After a crew member marks themselves Busy or Free, other members see the updated availability shape within 30 seconds (one polling interval).
3. The crew heatmap shows aggregated availability but never reveals the reason any member is busy.
4. The crew map screen shows which members are active (in Lab placeholder) with the 30-second polling loop running in the background.

---

## Phase 4: Lab & Astrophage Economy

**Goal:** Users can run focus sessions that earn Astrophage, with background jobs enforcing grace periods and anti-burnout rules.
**Requirements:** LAB-01, LAB-02, LAB-03, LAB-04, APH-01, APH-02
**Estimated effort:** 6 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Build Lab screen: discipline selector (Astrogeology / Astrophysics / Xenobiology) with multiplier labels
- [ ] Session state: "Start Lab" → active timer view showing elapsed time and earned Astrophage (server-authoritative, polled)
- [ ] Implement `AppLifecycleObserver` (WidgetsBindingObserver): on `paused` → call `POST /lab/sessions/:id/pause`; on `resumed` → call `POST /lab/sessions/:id/resume` and reset polling timer
- [ ] Show user's orbiting ship on crew map when session is active (discipline-colored, animated orbit)
- [ ] Show other members' orbiting ships (from `crewStatusProvider` — `in_lab`, `discipline` fields)
- [ ] Astrophage balance widget: personal balance + crew pooled total, updated via polling
- [ ] Anti-Burnout UI: if `burnout_flag` returned in crew status, flash ship red and show "Return to Campfire" prompt
- [ ] "End Session" button → calls `POST /lab/sessions/:id/end` → shows Astrophage earned summary screen

**Person B — Backend API**
- [ ] Write and apply DB migration: `lab_sessions`, `astrophage_ledger` tables + all indexes from ARCHITECTURE.md
- [ ] `POST /api/v1/lab/sessions` — start session: insert row, enqueue grace-period BullMQ job
- [ ] `POST /api/v1/lab/sessions/:id/pause` — record `paused_at`
- [ ] `POST /api/v1/lab/sessions/:id/resume` — clear `paused_at`, re-enqueue grace-period job
- [ ] `POST /api/v1/lab/sessions/:id/end` — close session, compute Astrophage, append to ledger
- [ ] `GET /api/v1/crews/:id/astrophage/me` — return `SUM(amount)` from ledger for caller in crew
- [ ] `GET /api/v1/crews/:id/astrophage/ledger` — paginated recent transactions
- [ ] Expand `GET /api/v1/crews/:id/status` to populate `crew_astrophage_pool`, `my_astrophage_balance`, `members[].in_lab`, `members[].discipline`, `members[].lab_started_at`
- [ ] Enforce: only the server appends to `astrophage_ledger` — no client-side balance writes

**Person C — Infra/Jobs**
- [ ] Set up BullMQ worker process as a separate entrypoint (`worker.ts`) sharing DB and Redis config
- [ ] Implement `grace-period-enforcer` job (every 30s): close paused sessions older than 5 min, compute `floor(duration_minutes × multiplier)`, append to astrophage_ledger
- [ ] Implement `anti-burnout-checker` job (every 5 min): find active sessions started > 6hr ago with `burnout_flag=false`, set flag, enqueue FCM notification (FCM stub for now — real send in Phase 5)
- [ ] Add BullMQ repeatable job definitions with cron expressions and confirm they survive a worker restart
- [ ] Add `idx_lab_sessions_crew_active` and `idx_lab_sessions_paused` indexes
- [ ] Write worker integration test: start session, simulate 5-min pause gap, confirm ledger entry created

### Success Criteria
1. A user can start a Lab session, select Astrogeology, and see their ship appear on the crew map within one polling cycle.
2. If the user leaves the app for more than 5 minutes, the session closes automatically and Astrophage is credited to their ledger (visible on next poll).
3. Astrophage earned reflects the correct discipline multiplier (1.5× for Astrogeology, 1.0× for Astrophysics, 1.2× for Xenobiology).
4. After 6+ continuous Lab hours, the user's ship flashes red and crew members see a gentle notification prompt.
5. Astrophage balance is always server-computed from the ledger; the client only displays what the server returns.

---

## Phase 5: Push Notifications & Launch Window

**Goal:** The app detects when the crew is free and fires a native push notification, and friends can fist-bump each other via haptic.
**Requirements:** NOTIF-01, NOTIF-02, NOTIF-03
**Estimated effort:** 5 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Integrate `firebase_messaging` Flutter package; request notification permissions on iOS/Android
- [ ] Implement `fcm_handler.dart`: `FirebaseMessaging.onBackgroundMessage` and foreground message handler
- [ ] Fist-bump handler: when FCM data message `{type: 'fistbump'}` arrives, call `HapticFeedback.heavyImpact()` three times with 150ms gaps (no visible notification)
- [ ] Launch Window handler: when FCM notification `{type: 'launch_window'}` arrives, navigate user to Campfire room screen
- [ ] Build Fist-Bump button on crew map: tapping an orbiting friend's ship calls `POST /api/v1/crews/:id/fistbump`
- [ ] Show Launch Window banner in crew home screen when `active_launch_window` is non-null in crew status response
- [ ] Wire up real push-token registration: on login and token-refresh, send device FCM token to `PUT /api/v1/auth/push-token`

**Person B — Backend API**
- [ ] Integrate `firebase-admin` Node.js SDK; configure service account credentials
- [ ] `POST /api/v1/crews/:id/fistbump` — validate both users are in same crew and active in Lab; look up target's push_token; send FCM data-only message `{type: 'fistbump', from_user_id}`
- [ ] Notification service module: `sendToUser(userId, message)` and `sendToCrewMulticast(crewId, message)` abstractions used by both API routes and BullMQ workers
- [ ] Write and apply DB migration: `launch_windows` table
- [ ] Expand `GET /api/v1/crews/:id/status` to return `active_launch_window` object when one exists

**Person C — Infra/Jobs**
- [ ] Implement `launch-window-scanner` repeatable job (every 60s): read `heatmap_slots` for all active crews for next 3-hour window; calculate per-crew free overlap %; if any crew >= 80%: insert `launch_windows` row (idempotent check), send FCM multicast to all crew `push_tokens`, create `campfire_rooms` row with `status: 'open'`
- [ ] Implement `push-token-cleanup` daily job (03:00 UTC): remove `push_tokens` where `last_seen > 90 days`
- [ ] Handle FCM delivery errors inline: on `messaging/registration-token-not-registered` response, delete that push_token row
- [ ] Configure APNs via FCM HTTP v1 (single API handles both iOS and Android — no separate APNs cert needed if using firebase-admin)
- [ ] End-to-end test: seed heatmap_slots so crew overlap >= 80%, run scanner, verify `launch_windows` row created and FCM stub called

### Success Criteria
1. When 80% or more of a crew's heatmap shows free blocks in an overlapping window, a push notification is delivered to all crew members' devices within 60 seconds of the scan.
2. The notification text reads in the spirit of "Stable Orbit Detected — gather time incoming."
3. Tapping an orbiting friend's ship in the app delivers a triple haptic pulse on the recipient's device with no visible notification shown.
4. Fist-bump fails gracefully with a clear error if the target user is not currently in a Lab session.
5. The Launch Window banner appears on the crew home screen as soon as the active_launch_window field populates in the polling response.

---

## Phase 6: Campfire & Hangout Unlock

**Goal:** A Campfire room opens on a Launch Window and the crew can propose and fund hangouts with Astrophage.
**Requirements:** CAMP-01, CAMP-02, CAMP-03
**Estimated effort:** 5 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Build Campfire room screen: shows open room, active hangout proposals list, crew Astrophage pool balance
- [ ] Proposal card component: description, total cost, pledged so far, progress bar, "Contribute" button
- [ ] Contribute bottom sheet: number input for Astrophage amount, confirm CTA, calls `POST /campfire/:room_id/proposals/:id/contribute`
- [ ] Leader-only "Propose Hangout" button: text input for description, Astrophage cost input, calls `POST /campfire/:room_id/proposals`
- [ ] Hide "Propose Hangout" button for non-leaders (check membership role from crew status provider)
- [ ] Funded proposal state: show "Unlocked" badge and celebration animation when `status == 'funded'`
- [ ] Navigate to Campfire room automatically when `active_campfire_room_id` appears in crew status polling response

**Person B — Backend API**
- [ ] Write and apply DB migration: `campfire_rooms`, `hangout_proposals` tables
- [ ] `GET /api/v1/crews/:id/campfire` — return active campfire room (if any) with proposals list
- [ ] `POST /api/v1/campfire/:room_id/proposals` — leader-only: insert hangout_proposals row; enforce leader role via middleware
- [ ] `POST /api/v1/campfire/:room_id/proposals/:id/contribute` — validate user balance sufficient (SUM ledger); append DEBIT entry to astrophage_ledger; update `astrophage_pledged` on proposal; if `astrophage_pledged >= astrophage_cost` → mark proposal `funded`, set `funded_at`, send FCM multicast to crew
- [ ] `GET /api/v1/crews/:id/status` — populate `active_campfire_room_id` field
- [ ] Guard: users cannot contribute more Astrophage than their current balance (server enforces, not client)

**Person C — Infra/Jobs**
- [ ] Extend `launch-window-scanner` to confirm campfire_rooms row is only created once per launch_window (idempotent insert using `ON CONFLICT DO NOTHING` or pre-check)
- [ ] Add campfire room auto-close job: close rooms where `launch_window.window_end < now()` and `status = 'open'` (runs every 5 min)
- [ ] Wire FCM multicast call from the contribute endpoint into the shared notification service (already built in Phase 5) — fire "Hangout unlocked — the Campfire is lit" message on funding
- [ ] Load test the contribute endpoint: simulate concurrent contributions from 10 members and verify ledger entries are correct with no double-spend (balance check + debit must be atomic within a DB transaction)

### Success Criteria
1. When a Launch Window fires, a Campfire room screen is accessible from the crew home within one polling cycle (no manual navigation required).
2. The crew leader can post a hangout proposal with a description and Astrophage cost; non-leaders cannot see or access the propose action.
3. Any crew member can contribute Astrophage toward a proposal; the contribution is rejected if the member's balance is insufficient.
4. When total contributions reach the cost, the proposal is marked funded and all crew members receive a push notification.
5. Astrophage debits are server-authoritative and atomic — concurrent contributions never result in a negative balance or double-spend.

---

## Phase 7: Cosmetics & Onboarding

**Goal:** New users land in the narrative onboarding flow and earn their first Astrophage; a cosmetics catalog lets the crew spend together.
**Requirements:** ONB-01, ONB-02, COS-01, COS-02
**Estimated effort:** 4 days parallel

### Workstreams

**Person A — Flutter/Dart**
- [ ] Build onboarding screen 1 — Black Void: full-screen dark background, animated signal pulse, "Signal detected. Origin: Unknown."
- [ ] Build onboarding screen 2 — The Transmission: Nomai-style UI locks in, invite message from friend appears, narrative reveal animation
- [ ] Build onboarding screen 3 — The Choice: "The Campfire is cold. Enter the Lab to begin syncing your timeline." CTA button
- [ ] Calibration Focus screen: 5-minute countdown Lab session using existing Lab session flow (start → complete → ledger entry) — first Astrophage earned
- [ ] Guard: show onboarding only on first login; store completion flag in secure storage and skip on subsequent launches
- [ ] Build Cosmetics catalog screen: grid of ship skins, campfire themes, and crew emblems; show locked/unlocked state
- [ ] Cosmetic detail sheet: name, description, cost in Astrophage or milestone condition, "Unlock" CTA
- [ ] Unlock flow: call cosmetics unlock endpoint, deduct Astrophage via ledger, update UI state
- [ ] Apply unlocked ship skin to user's orbit ship on crew map

**Person B — Backend API**
- [ ] Serve cosmetics catalog as a static JSON file (no DB table needed for v1): `GET /api/v1/cosmetics` returns full catalog
- [ ] `POST /api/v1/cosmetics/:id/unlock` — validate user has sufficient Astrophage balance; append DEBIT to astrophage_ledger with `reason: 'cosmetic_unlock'`; store unlocked cosmetic reference on user row (add `unlocked_cosmetics JSONB` column to `users` via migration)
- [ ] `GET /api/v1/users/me/cosmetics` — return list of unlocked cosmetic IDs for caller
- [ ] Milestone-based unlock: add server-side check for group milestone condition (e.g., crew pooled Astrophage > threshold) before granting milestone cosmetics
- [ ] Update `GET /api/v1/users/me` to include `active_skin_id` so crew map renders correct ship skin

**Person C — Infra/Jobs**
- [ ] Write the cosmetics catalog JSON file (`cosmetics.json`) with initial set: 3 ship skins, 2 campfire themes, 2 crew emblems — each with id, name, description, cost, unlock_condition
- [ ] Add `unlocked_cosmetics` JSONB column migration for `users` table
- [ ] Configure production deployment: two-process deploy (API server + BullMQ worker), environment secrets, health-check endpoint monitored
- [ ] Set up error alerting (e.g., Sentry or equivalent) for both API server and worker process
- [ ] Final smoke test: run full earn-and-gather loop end-to-end on staging — register → crew → heatmap → lab → Astrophage earned → launch window → campfire → hangout funded

### Success Criteria
1. A brand-new user opening the app for the first time sees the Black Void → Transmission → Choice → Calibration Focus sequence before reaching the crew home screen.
2. Completing the Calibration Focus earns the user their first Astrophage via the normal Lab session flow (ledger entry created, balance reflects it).
3. The onboarding sequence is skipped for returning users (completion flag persists across cold restarts).
4. The cosmetics catalog screen shows all available items with their Astrophage cost and locked/unlocked state.
5. A user with sufficient Astrophage balance can unlock a cosmetic; the unlock is reflected immediately in the crew map ship skin.

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/3 | Not started | - |
| 2. Crew & Social Graph | 0/3 | Not started | - |
| 3. Heatmap & Polling | 0/3 | Not started | - |
| 4. Lab & Astrophage Economy | 0/3 | Not started | - |
| 5. Push Notifications & Launch Window | 0/3 | Not started | - |
| 6. Campfire & Hangout Unlock | 0/3 | Not started | - |
| 7. Cosmetics & Onboarding | 0/3 | Not started | - |
