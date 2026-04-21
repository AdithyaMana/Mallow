# State — Mallow v1.0 MVP

> Project memory. Updated at every phase transition and plan completion.

---

## Project Reference

**Core value:** When enough of a crew finishes work at the same time, every member gets a push notification and knows it's time to gather — and they feel like they *earned* it.

**Milestone:** v1.0 MVP

**Current focus:** —

---

## Current Position

| Field | Value |
|-------|-------|
| Phase | Not started |
| Plan | — |
| Status | Roadmap created — ready to plan Phase 1 |
| Progress | 0 of 7 phases complete |

```
[                                        ] 0%
Phase 1 ─────────────────────────────── Phase 7
```

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases defined | 7 |
| Requirements mapped | 25 / 25 |
| Plans complete | 0 |
| Phases complete | 0 |

---

## Accumulated Context

### Decisions locked in
- Flutter + Riverpod (not BLoC) for all UI and state
- Node.js + Fastify + Drizzle ORM (not Go, not Express)
- BullMQ + Redis for all background jobs (no in-memory cron)
- REST polling every 30-60s (not WebSockets)
- Append-only `astrophage_ledger` (never a mutable balance column)
- Server-authoritative Astrophage economy (client never computes balances)
- FCM HTTP v1 via firebase-admin handles both iOS and Android push (no separate APNs cert)
- Cosmetics catalog served as static JSON in v1 (no DB table needed)

### Architecture constraints to enforce
- Flutter app never computes Astrophage balances — requests them from server
- All heatmap data stored as normalized `heatmap_slots` rows (never JSON blob)
- BullMQ jobs must be repeatable with Redis persistence (survive server restart)
- Grace period is explicit (`paused_at` column), not inferred from polling gaps
- Astrophage debits in contribute endpoint must be atomic (DB transaction)

### Todos
- Plan Phase 1 using `/gsd:plan-phase 1`

### Blockers
- None

---

## Session Continuity

**Last activity:** 2026-04-21 — Roadmap created (7 phases, 25 requirements)

**Next action:** `/gsd:plan-phase 1` — Foundation
