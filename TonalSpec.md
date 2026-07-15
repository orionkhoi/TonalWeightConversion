# TonalPlan — Product Spec

*Upgrades BRIEF.md. The brief described a tool; this spec describes the coach that tool grows into — and draws a hard line around the small slice that ships first.*

**Status:** Live converter, My Lifts, accounts + sync, 315-move library, and the goal-driven Workout Creator (with editing + export) are already built. This spec adds the first proactive coaching loop on top of that foundation.

---

## 1. Problem & user

**The user.** Someone who trains on a Tonal home gym and wants to keep showing up. They fall on a spectrum: some need pushing, some just need a cheerleader — but all of them need *something* that keeps the habit alive between good intentions and actual sessions.

**The problems, in order of depth:**

1. *Surface (already solved):* Tonal's digital resistance feels heavier than the same number in free weights, and building equipment-appropriate workouts by hand is tedious.
2. *Deeper (the real opportunity):* a tool waits to be opened. Motivation doesn't. The person who most needs a workout is the one least likely to launch an app and ask for one. The gap isn't information — it's initiation. Nobody starts the conversation on the hard days.

TonalPlan's job is to close that second gap: to be the thing that reaches out first, reads how you're actually doing, and gets you to train anyway — in the voice that works on *you*.

---

## 2. The Dream (north star)

**TonalPlan is a coach that lives where your data lives and starts the conversation.**

The north-star behaviors, each a principle we found by working through the ideal:

- **It talks first.** Proactive, not reactive. It messages you before you open anything.
- **It persuades, it doesn't just inform.** When it overrides your plan it gives a reason built to *move* you, and keeps the "why" rare so it stays powerful.
- **It learns which lever works on you.** Pride, guilt, streak, or a photo — it watches what actually gets you training and leans into it.
- **It rewards honesty over perfection.** Telling it "I'm hurt" or "not today" is the winning move, not a penalty. A streak that survives a skipped day is worth more than one that punishes the truth.
- **It acknowledges the bad day.** It names that you're off and carries you through with a lighter session, rather than going silent or getting tough.
- **It can see you.** Progress photos are the coach's eyes — catching plateaus your mirror hides, flagging imbalances before they hurt, and using your own before/after as the argument no words beat.
- **It becomes the right coach for each person.** One engine, many bedside manners; warm for you, sharper for someone who needs a drill sergeant.
- **It only uses what's already in front of it.** Lean on first-party, low-creep signals (your lifts above all); don't reach into your calendar or bedroom.

North-star test for any future feature: *does it help the coach start a better conversation, or is it just another screen the user has to open?*

---

## 3. System map (inputs → decisions → outputs)

An **event-driven loop**, not a screen funnel. Every output feeds back and sharpens the next read.

**Triggers (what starts it):** morning · a Tonal session logged · your reply · inaction · you asking directly.

**Inputs (what flows in):**

- *You (typed):* goals, lift logs, progress photos, replies to the coach.
- *Tonal* ★ *(crown jewel):* lift quality — bar speed, eccentric control, rep decay. Always available, never creepy.
- *Screen time:* phone usage as a rough-day tell.
- *Golf app:* real-world drive distance, to close the loop on that goal.
- *Declined by design:* wearable sleep (coverage — not everyone has the gear) and calendar (felt like snooping).

**Decisions (where it combines + chooses):**

1. **Readiness read** — lift trend + screen time + reply tone → good day / off day.
2. **Keep, lighten, swap, or rest** — planned session vs. readiness.
3. **Pick the motivation lever** — the message style that historically moves this user.
4. **Persuade or just inform** — how hard to push, and whether to spend a "why."
5. **Honesty handling** — "injured / not feeling it" → recovery, no penalty, warm reframe.
6. **Generate workout** — goal + owned equipment + history → warmup → blocks → finisher → cooldown, loads from My Lifts.
7. **Progress read** — current photos/lifts vs. baseline → plateau, imbalance, regression → choose a warm reframe.
8. **Personality adaptation** — slowly tune tone to the user's dialect of motivation.
9. **Channel + timing** — reach out now or stay quiet, and through which channel.

**Outputs (channels, escalating to match stakes):** in-app screens · push (default) · text/SMS (high-stakes) · email (weekly digest). Then it **loops back** — every response teaches it more about you.

---

## 4. Data model

Core entities (v1-relevant unless marked *Later*):

- **user** — id, email, auth, created_at, coaching_tone_pref (warm | neutral | sharp | learn), notification_prefs.
- **equipment** — user_id, accessory (bar, rope, handles, bench, roller…), owned (bool). Drives the "only my equipment" filter.
- **exercise** *(catalog)* — id, name, primary_muscle, region, movement_pattern, required_accessories[]. The 315-move library. Read-only reference data.
- **my_lift** — user_id, exercise_id, tonal_weight, felt_equivalent_freeweight, created_at. Averaged into a **personal_multiplier** per exercise for the converter and suggested loads.
- **workout** — id, user_id, name, description, goal_text, source (generated | edited), created_at.
- **workout_movement** — workout_id, exercise_id, order, block (warmup/block/finisher/cooldown), sets, reps, suggested_load. *(The "next up" per-movement customization from the brief lands here.)*
- **session** — id, user_id, workout_id, scheduled_for, status (planned | completed | skipped | recovery), felt_rating.
- **session_movement_log** — session_id, exercise_id, actual_load, actual_reps, felt, (velocity / eccentric_control *Later*, from Tonal).
- **readiness_read** — user_id, date, signals_json, verdict (good | off), chosen_action.
- **coach_message** — user_id, sent_at, trigger, channel, tone, body, why_shown (bool), user_response.
- **goal** — user_id, text, created_at, active. Kept so the coach can quote your past words back at you.
- **progress_photo** *(Later)* — user_id, taken_at, image_ref, derived_notes.
- **motivation_signal** *(Later)* — user_id, lever, message_variant, outcome. The learning substrate for lever selection + personality.

---

## 5. External Interfaces

Each outside service, direction, and mechanism. **v1** interfaces are the minimum to ship the first loop; the rest are staged.

| Service | Direction | Mechanism | Purpose | Stage |
|---|---|---|---|---|
| Supabase (Postgres, Auth, RLS) | both | SDK | Accounts, sync, per-user data isolation | **v1** (built) |
| Push (APNs / FCM) | out | provider SDK | The proactive nudge — default channel | **v1** |
| Tonal Custom Workouts | out | "Copy for Tonal" build sheet | Recreate a workout on the Tonal | **v1** (built) |
| Tonal account/device ★ | in | API / connector | Lift quality: speed, eccentric control, rep decay | Later |
| Apple Screen Time / Android Digital Wellbeing | in | device API | Rough-day signal | Later |
| Camera / Photo library | in | device API | Progress photos as the coach's eyes | Later |
| Golf apps (Arccos, Garmin Golf, 18Birdies) | in | API | Correlate training to real drive distance | Later |
| SMS (Twilio or similar) | out | API | High-stakes persuade / app gone unopened | Later |
| Email (Postmark / SendGrid) | out | API | Weekly progress digest | Later |

---

## 6. Screens list

- **Auth / onboarding** — sign in; capture goals, owned equipment, tone preference.
- **Home / Today** — the coach's message + today's session (confirm, swap, or rest). The new center of gravity.
- **Converter** — Tonal ↔ free-weight, both directions. *(built)*
- **My Lifts** — log felt-equivalents; view personal multipliers. *(built)*
- **Movement Library** — 315 moves, searchable, "only my equipment" filter. *(built)*
- **Workout Creator** — goal in, full session out. *(built)*
- **Workout detail / editor** — rename, edit, swap movements; per-movement sets/reps/load. *(editing built; per-movement customization = next)*
- **Coach message / "why" card** — the override and its reason; reply path ("injured / not today").
- **Settings** — equipment, tone, notification channels.
- **Export sheet** — "Copy for Tonal." *(built)*
- **Progress timeline** *(Later)* — photos + strength curves + before/after.

---

## 7. v1 scope — the first slice (ruthless)

**Thesis:** prove the north star — *a coach that reads you and reaches out* — using **only data already inside TonalPlan**, with **zero new external integrations**. If the proactive loop doesn't change behavior with in-app data, no amount of Tonal-API or computer-vision magic will save it.

**v1 ships:**

1. Everything already built stays: converter, My Lifts, library, Workout Creator + editing + export, accounts + sync.
2. **Per-movement customization** (sets, reps, load) — the "next up" from the brief, needed so sessions are real enough to swap and log.
3. **In-app readiness read** — computed purely from logged workout trends the app already owns (loads dragging, sessions skipped). No Tonal API, no screen time.
4. **One proactive channel: push** — a morning message that confirms the planned session or swaps it, with a single-line "why."
5. **Honesty handling** — a reply path ("injured" / "not feeling it") that adjusts the session, logs it as recovery, and applies **no penalty**; warm acknowledgment tone throughout.

**Ruthlessly cut from v1** (not gone — see Later): Tonal API, screen-time and golf signals, progress photos + computer vision, motivation-lever A/B learning, per-user personality adaptation, SMS + email channels, and any streak mechanic. One channel, one honest signal source, one warm voice. That's the test.

**v1 success looks like:** users who get the morning push train more often than those who don't — and skipping honestly, without penalty, keeps them coming back rather than churning.

---

## 8. Later (preserved, not deleted)

Staged roughly by how much each deepens the north star per unit of new complexity:

- **Tonal API integration ★** — the crown-jewel signal. Upgrades readiness from "did you skip?" to "are your reps decaying?" The single highest-leverage Later item.
- **Motivation-lever learning** — track which message styles actually get *you* training; start choosing them.
- **Progress photos as the coach's eyes** — timeline, plateau detection, imbalance flags, before/after as persuasion.
- **Personality adaptation** — one engine, per-user bedside manner.
- **Multi-channel escalation** — SMS for high-stakes moments, weekly email digest.
- **Outside-world signals** — screen time (rough-day tell), golf apps (close the drive-distance loop).
- **Honesty-first streak redesign** — a streak that rewards truth over the appearance of perfection, so nobody fakes an injury to protect a number.

**Open questions to resolve before building the Later items** (carried from the dream session):

- *A/B-ing the user:* how much experimentation on which lever moves you is fair before it feels like being played?
- *When to release you:* how does the coach know the day you're right to skip — and grant permission that actually means something?
- *The receipts question:* should it quote your past goals back at you when you're lazy — powerful, but where's the line into resentment?
- *Mood vs. trust on the worst day:* when the news is genuinely bad, which does it protect? That single choice is the product's whole personality.
- *The streak's fans:* killing the punishing streak rewards honesty — but some users need that dumb little number. What replaces it for them?
