# Mallow

![Mallow](Mallow.jpg)

You and your friends keep saying "we should hang out" and then never do. Not because you don't want to — because nobody knows when everyone's actually free, and by the time you figure it out, the moment's gone.

Mallow fixes that. It's a mobile app built around one idea: work earns the hangout. Log focus time, and when enough of the crew wraps up at the same time, everyone gets a notification. No scheduling, no group chats going nowhere, just a signal that it's time.

The name is from Outer Wilds — that campfire moment where you're sitting under the stars with your friends, eating marshmallows, nothing on the agenda. That's the feeling this app is trying to make happen more often.

## How it works

**The Heatmap** is where you mark your time as busy or free in broad strokes. No event titles, no details — your friends see when you're occupied, never why. When 80% or more of the crew's schedules line up, the app fires a Launch Window notification.

**The Lab** is where the work happens. Pick a Science Discipline when you start a session and your ship appears on the crew's orbit map. Focus time earns Astrophage (1 min = 1 mg), and the discipline you pick affects your multiplier — Astrogeology (coding, math) at 1.5×, Xenobiology (collab work) at 1.2×, and Astrophysics (reading, planning) at 1.0×.

**The Campfire** opens when a Launch Window fires. The crew leader proposes a hangout, everyone pools their Astrophage to fund it, and it happens. The grind paid for something real.

## Stack

Built with Flutter for iOS and Android from a single codebase, backed by a Node.js + Fastify REST API, PostgreSQL for persistent state, and BullMQ + Redis for background jobs. Astrophage balances are stored as an append-only ledger — never a mutable column. Push notifications run through Firebase Cloud Messaging for both platforms.

## Status

Currently in active development, building toward the v1 MVP — the full earn-and-gather loop from heatmap through lab, launch window, and funded hangout.

*by Adithya*
