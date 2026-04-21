# Mallow
![Mallow](Mallow.jpg)

We all have those friends we keep meaning to hang out with but never actually do. It is not a lack of interest. The real issue is that nobody knows when everyone is actually free, and by the time you finally figure out a plan, the momentum is gone.

Mallow is our solution to this problem. The core concept is simple: your focus time earns you a hangout. You just log your work hours, and when enough of your friend group finishes up at the same time, everyone gets an alert. There is no tedious scheduling or endless group chats. You just get a signal that it is time to meet up.

We named it after that iconic campfire moment in the game Outer Wilds. You are sitting under the stars with your friends, roasting marshmallows, and having absolutely nothing on the agenda. We want this app to help recreate that exact feeling more often in real life.

## How Mallow Works

* **The Heatmap:** This is where you block out your busy and free time in broad strokes. Privacy is built right in. You do not need to add event titles or details. Your friends only see when you are occupied, but they never see why. Once 80% of your group's schedules align, the app triggers a Launch Window notification.
* **The Lab:** This is your focus zone. When you start a work session, you pick a Science Discipline and your ship pops up on your crew's shared orbit map. Working earns you an internal currency called Astrophage at a base rate of one milligram per minute. The discipline you choose applies a multiplier to your earnings. For example, Astrogeology for coding and math gives you 1.5x. Xenobiology for collaborative work gives you 1.2x. Astrophysics for reading and planning gives you 1.0x.
* **The Campfire:** This unlocks as soon as a Launch Window fires. The group leader suggests a hangout idea, everyone pools the Astrophage they earned during the week to fund it, and the meetup happens. Your hard work directly translates into actual offline plans.

## The Tech Stack

We are building the mobile app using Flutter so it runs naturally on both iOS and Android from a single codebase. The backend is a Node.js and Fastify REST API. For data storage, we are using PostgreSQL, and background jobs are handled by BullMQ and Redis. 

To keep things secure and accurate, Astrophage balances function as an append only ledger rather than just an editable database column. Finally, we rely on Firebase Cloud Messaging to handle push notifications across both operating systems.

## Current Status

Mallow is currently in active development by 3 friends. We are putting all our effort into building the version one MVP. The primary goal right now is to perfect the complete user loop from blocking time on the heatmap to earning Astrophage in the lab and finally triggering a funded hangout at the campfire.

*Work by Flazer,Akira and Deadshot