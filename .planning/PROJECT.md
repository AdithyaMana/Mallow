# Mallow

> *"Humanity survived because we shared our fire. We will save the future because we synced our calendars."*

## Vision

Mallow bridges the vacuum between deep work and real life. It is a dual-engine mobile app that solves **Temporal Isolation** — the modern condition where people grind in productive isolation until their social batteries die, with no mechanism to transition back to connection.

The core insight: **Work is the currency for Play.** Friends earn the right to gather by logging focus time. The app detects when the crew is collectively free and fires the signal. Everyone meets at the Campfire.

The name comes from the Outer Wilds campfire — the moment where the protagonist sits under the stars, eating marshmallows, with no agenda. Mallow is engineered to manufacture more of those moments.

## Core Value

**One thing that must work:** When enough of a crew finishes work at the same time, every member gets a push notification and knows it's time to gather — and they feel like they *earned* it.

## Inspirations

- **Outer Wilds** — Campfire aesthetic, Nomai-style UI elements, warmth, exploration, marshmallows
- **Project Hail Mary** — Astrophage as the focus currency, scientific discipline framework, mission-critical crew dynamics
- **Yeolpumta** — Live study accountability, but stripped of toxic "study-until-you-drop" culture
- **When2Meet / Focusmate / Forest** — Each solves half the problem. Mallow closes the loop.

## The Problem

| Trap | Symptom | Root Cause |
|------|---------|------------|
| YPT Trap | Grinding in isolation until social battery dies | No off-ramp from productivity |
| Calendar Trap | Scheduling hangouts with tools built for board meetings | Corporate scheduling ≠ friend coordination |
| Focus Trap | Apps that track work but never connect you to people | Half-loop — no reward for the grind |

## The Solution: Two Engines, One Loop

**Engine 1 — The Heatmap (Social Coordination)**
Friends mark their availability in broad strokes — no event names, no details. The app scans continuously for a Launch Window: when 80%+ of the crew's schedules open simultaneously, it fires a push notification. Privacy is default — friends see *when* you are busy, never *why*.

**Engine 2 — The Laboratory (Deep Work)**
When working, users enter The Lab and select a Science Discipline. Their avatar ship orbits on the group map. Focus time earns Astrophage — the closed-loop currency that fuels social coordination.

**The Bridge — The Astrophage Economy**
1 minute of focus = 1 mg of Astrophage. Crew members pool Astrophage to unlock hangout proposals made by the rotating crew leader. Spending Astrophage to unlock a Campfire transforms scheduling from obligation into reward.

## Target Users

Anyone with a crew who grinds — college students, young professionals, distributed friend groups. No age gate. If you have people you want to see more but keep missing, Mallow is for you.

## Platform & Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Mobile | Flutter (iOS + Android) | Single codebase, excellent animation performance for space UI |
| Backend | Custom server (Node.js or Go) | Full control over Astrophage economy logic and real-time polling |
| Real-time | Near real-time (30-60s polling) | Good enough for crew status; simpler than WebSockets |
| Push | FCM (Android) + APNs (iOS) | Launch Window ping is the core feature — must be native push |

## Team

Small team (2-3). Parallel workstreams possible.

## Key Features

### The Heatmap UI
- Tap time blocks to mark: **In the Lab** (Busy) or **Oxygen Flowing** (Free)
- No event titles, no descriptions — Ghost Matter Filter by default
- Friends see availability shape, never reason

### Launch Window Detection
- Continuous scan for 80%+ crew overlap
- Push notification: *"Stable Orbit Detected: 3 hours until the next Campfire."*
- Hangout Index formula calculates emotional + temporal availability

### The Laboratory
- Enter session → select Science Discipline → avatar ship orbits on group map
- **Astrogeology** (Math, Coding) → 1.5x Astrophage
- **Astrophysics** (Reading, Planning) → 1.0x Astrophage
- **Xenobiology** (Collaborative work) → 1.2x Astrophage
- Leave app → 5-minute grace period → session closes
- 6+ hours in Lab → ship flashes red (Radiation Warning) → friends notified to send "Return to Campfire"

### Social Mechanics
- **Fist-Bump**: Tap orbiting friend's ship → haptic thump-thump-thump on their phone
- **Positive FOMO**: *"Rocky and Grace are generating Astrophage. The ship needs your output."*
- **Crew Roles**: Rotating/elected leadership — leader proposes hangouts, crew unlocks them

### The Campfire
- When Launch Window fires → shared virtual Campfire room opens for the crew
- Crew leader proposes custom hangout ideas
- Group spends pooled Astrophage to unlock and activate a hangout
- Coordination happens in-app (Campfire room) and outside

### Cosmetics
- Ship/avatar skins
- Campfire environment themes
- Crew emblems and profile flair
- Unlocked via Astrophage spending and group milestones

### Onboarding (The Transmission)
- Screen 1: Black Void — *"Signal detected. Origin: Unknown."*
- Screen 2: The Transmission — Nomai-style UI locks in, friend's invite message appears
- Screen 3: The Choice — *"The Campfire is cold. Enter the Lab to begin syncing your timeline."*
- User completes 5-minute Calibration Focus → earns first Astrophage → instantly invested

## Lore & Aesthetic

- **Original IP** built on top of Outer Wilds and PHM inspiration
- Astrophage (term) retained — it's perfect
- Nomai-style UI elements retained as aesthetic direction
- Character names in the live app are original (not Rocky, Grace, Stratt)
- **Living lore**: The universe expands — notifications, events, and unlocks tell an ongoing story
- The cosmetic stakes: group cosmetic unlocks are tied to collective health — if the crew goes inactive, consequences ripple through the shared universe

## Social Model

- Invite-only circles — no discovery, pure trust networks
- Flexible crew size — no hard cap
- Privacy by default throughout

## Monetization

Free forever in v1. Build the habit loop first.

## Requirements

### Validated

*(None yet — ship to validate)*

### Active

- [ ] User can mark time blocks as Busy or Free on the Heatmap
- [ ] App detects Launch Window when 80%+ of crew is free simultaneously
- [ ] App sends push notification when Launch Window opens
- [ ] Friends see *when* user is busy, never *why* (Ghost Matter Filter)
- [ ] User can enter a Lab session and select a Science Discipline
- [ ] User earns Astrophage at discipline-specific rates during Lab sessions
- [ ] Lab session closes after 5-minute grace period if user leaves app
- [ ] User's avatar ship appears on crew's live orbit map during Lab session
- [ ] User can Fist-Bump an orbiting friend (haptic on recipient's device)
- [ ] Anti-Burnout Protocol fires after 6+ continuous Lab hours
- [ ] Crew leader can propose custom hangout ideas
- [ ] Crew can pool and spend Astrophage to unlock a hangout proposal
- [ ] Shared Campfire room opens when Launch Window activates
- [ ] Invite-only crew creation via invite link
- [ ] Rotating/elected crew leadership
- [ ] Narrative onboarding flow (Black Void → Transmission → Choice → Calibration)
- [ ] Cosmetic system (ship skins, campfire themes, crew emblems)

### Out of Scope (v1)

- Hangout Index formula ($H.I.$) — defer to v2 (simple overlap detection is sufficient for v1)
- Multiplier wheel variable reward — v2 habit loop enhancement
- System Health / Sun Supernova mechanic — v2 group stakes
- Hangout Tokens (earned by attending) — v2 reward layer
- Schrödinger's Campfire full implementation — v2
- Any monetization — free forever in v1

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Flutter over React Native | Animation-heavy space UI benefits from Flutter's rendering | — Pending |
| Custom backend over BaaS | Astrophage economy logic needs full control; BaaS limits long-term | — Pending |
| Near real-time (polling) over WebSockets | Simpler architecture; 30-60s is good enough for crew status | — Pending |
| Core loop MVP (earn, not spend) | Validates the fundamental habit loop before building economy | — Pending |
| Free forever (v1) | Grow the habit first; monetization follows retention | — Pending |
| Invite-only social graph | Trust networks over viral growth — protects the campfire feeling | — Pending |

---
*Last updated: 2026-04-20 after initialization*
