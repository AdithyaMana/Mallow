# Requirements — Mallow v1.0 MVP

> Scoped requirements for the first milestone. All requirements map to exactly one roadmap phase.

---

## v1.0 Requirements

### Foundation

- [ ] **FOUND-01**: User can register with a display name
- [ ] **FOUND-02**: User can log in and receive JWT auth tokens (access + refresh)
- [ ] **FOUND-03**: App registers device push token with backend on login/token-refresh

### Crew

- [ ] **CREW-01**: User can create an invite-only crew
- [ ] **CREW-02**: User can join a crew via invite deep link
- [ ] **CREW-03**: Crew has a rotating/elected leader role
- [ ] **CREW-04**: Friends see *when* user is busy, never *why* (Ghost Matter Filter — privacy by default)

### Heatmap

- [ ] **HEAT-01**: User can mark 30-minute time blocks as Busy or Free on the Heatmap
- [ ] **HEAT-02**: User can view crew members' combined availability heatmap

### Lab

- [ ] **LAB-01**: User can start a Lab session and select a Science Discipline (Astrogeology / Astrophysics / Xenobiology)
- [ ] **LAB-02**: User's avatar ship appears on the crew orbit map during an active Lab session
- [ ] **LAB-03**: Lab session closes automatically after a 5-minute grace period when user leaves the app
- [ ] **LAB-04**: Anti-Burnout Protocol fires after 6+ continuous Lab hours (warning to user + gentle crew notification)

### Astrophage

- [ ] **APH-01**: User earns Astrophage at discipline-specific rates during Lab sessions (1.5× Astrogeology, 1.0× Astrophysics, 1.2× Xenobiology)
- [ ] **APH-02**: User can view personal Astrophage balance and crew's pooled total

### Notifications

- [ ] **NOTIF-01**: App detects a Launch Window when 80%+ of crew is free simultaneously (background job, 60s scan)
- [ ] **NOTIF-02**: App sends native push notification (FCM + APNs) when a Launch Window opens
- [ ] **NOTIF-03**: User can Fist-Bump an orbiting friend — recipient's device delivers haptic feedback (thump ×3)

### Campfire

- [ ] **CAMP-01**: Shared Campfire room opens automatically when a Launch Window activates
- [ ] **CAMP-02**: Crew leader can propose a custom hangout idea with an Astrophage cost
- [ ] **CAMP-03**: Crew members can contribute Astrophage to unlock a hangout proposal

### Onboarding

- [ ] **ONB-01**: Narrative onboarding flow — Black Void → Transmission → Choice → Calibration Focus screen sequence
- [ ] **ONB-02**: Calibration Focus earns user their first Astrophage via the normal Lab session flow

### Cosmetics

- [ ] **COS-01**: Cosmetic catalog exists (ship skins, campfire themes, crew emblems)
- [ ] **COS-02**: Cosmetics can be unlocked via Astrophage spending and group milestones

---

## Future Requirements (deferred to v2+)

- Hangout Index formula (H.I.) — complex availability scoring; simple overlap detection sufficient for v1
- Multiplier wheel variable reward — habit loop enhancement
- System Health / Sun Supernova mechanic — group stakes for inactive crews
- Hangout Tokens (earned by attending) — reward layer for completing hangouts
- Schrödinger's Campfire full implementation — uncertainty mechanic
- Any monetization

---

## Out of Scope (v1)

| Item | Reason |
|------|--------|
| Real-time WebSockets | 30-60s polling delivers identical UX with far less complexity |
| GraphQL / tRPC | Fixed screen data requirements make REST sufficient; tRPC adds no value for Dart clients |
| Mutable Astrophage balance column | Append-only ledger is correct; mutable columns cause race conditions |
| Client-side Astrophage computation | Server is authoritative; client exploitability unacceptable |
| User discovery / public profiles | Invite-only model — no discovery in v1 |

---

## Traceability

| REQ-ID | Phase | Status |
|--------|-------|--------|
| FOUND-01 | — | Pending |
| FOUND-02 | — | Pending |
| FOUND-03 | — | Pending |
| CREW-01 | — | Pending |
| CREW-02 | — | Pending |
| CREW-03 | — | Pending |
| CREW-04 | — | Pending |
| HEAT-01 | — | Pending |
| HEAT-02 | — | Pending |
| LAB-01 | — | Pending |
| LAB-02 | — | Pending |
| LAB-03 | — | Pending |
| LAB-04 | — | Pending |
| APH-01 | — | Pending |
| APH-02 | — | Pending |
| NOTIF-01 | — | Pending |
| NOTIF-02 | — | Pending |
| NOTIF-03 | — | Pending |
| CAMP-01 | — | Pending |
| CAMP-02 | — | Pending |
| CAMP-03 | — | Pending |
| ONB-01 | — | Pending |
| ONB-02 | — | Pending |
| COS-01 | — | Pending |
| COS-02 | — | Pending |
