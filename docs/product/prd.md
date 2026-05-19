# Liftly — Product Requirements Document

**Status:** MVP scope locked
**Audience:** A small private group (the author + friends), used as testers.
**Companion doc:** [Ubiquitous Language](./ubiquitous-language.md) — all bolded terms below are defined there.

---

## 1. Overview

Liftly is a **weightlifting tracker for casual lifters** that removes the single hardest decision a beginner faces: *"when should I add weight?"*

Most lifting apps are passive — they log what you did. Liftly is active: it watches your reps against a configured **Rep Range** and prompts you, mid-workout, to add (or drop) weight on the next session the moment you've earned it. The progression rule is **double progression** — hit the top of the range, the weight goes up next time.

The user opens a **Routine**, logs **Sets**, and the app handles the math. Both barbell/dumbbell lifts and bodyweight movements (pull-ups, push-ups, dips) are first-class — the same progression mechanic applies to both.

## 2. Goals & Non-Goals

### Goals
- Make logging a set take **≤ 5 seconds** of tapping in the gym.
- Remove the "when do I add weight?" decision for casual users.
- Persist progression history reliably across phone wipes and reinstalls.
- Work in low-signal gym environments without losing data.
- Ship in dark/light theme and at least two languages.

### Non-Goals (MVP)
- Powerlifting features (RPE, RIR, 1RM estimation, percentage-based programming).
- Pre-built training programs (5/3/1, PPL, etc.). *Planned post-MVP.*
- iOS, web, or any non-Android client.
- Freestyle workouts without a Routine.
- Monetization (free for now; revenue model deferred).
- Analytics dashboards or charts.
- Public sharing, social features, leaderboards.
- Coach / multi-user account features.
- Warm-up tracking (users warm up on their own).

## 3. Target User

**One persona: the casual trainee.**

Trains for general strength and fitness 1–4× per week, by any of the following means — often mixed in the same Routine:
- Barbell and dumbbell lifts at a gym.
- Machine-based work.
- **Bodyweight training** (pull-ups, push-ups, dips, etc.), at home, in a park, or in a gym.

Other characteristics:
- Knows the basic movements but doesn't follow a programmed cycle.
- Wants to "get stronger" without thinking about programming theory.
- Will not configure rep ranges, RPE, or periodization on day one.
- Will abandon onboarding longer than ~60 seconds.

Bodyweight Exercises are treated as a first-class case, not an afterthought: the same Rep Range, Trigger Rule, and Progression Event flow apply to a pull-up as to a bench press. For pull-ups and similar, progression operates on **Added Weight** (e.g., dip-belt load), while the user's Bodyweight is shown alongside it.

Power users (powerlifters, programmed intermediates, coaches) are **out of scope.** The product should feel slightly under-featured to them — that's correct.

## 4. Core Mechanic: Double Progression

The mechanic is the product. Everything else exists to support it.

### Default configuration
- **Rep Range:** `2 sets × 12–15 reps`
- **Increment:** `+5 kg` (or `+5 lb` for lb users)
- **Trigger Rule:** reps on the **first Set** ≥ `upperLimit` → fire **Increase Modal**
- **Deload Rule:** reps on the **last Set** < `lowerLimit` → fire **Decrease Modal**

All of these are configurable globally and overridable per Exercise.

### Modal behavior
- **Reactive, mid-workout.** The Increase Modal fires the moment the trigger Set is logged. The Decrease Modal fires the moment a Set falls short of the lower limit.
- **Single decision:** "Increase to 65 kg next time?" → **Yes** / **No**.
- **Yes** → records a **Progression Event**, persists a **Progression Notification**, and updates **Working Weight** for the *next* Workout (never the current one).
- **No** → no change, no record beyond the user's choice.
- **At most one Increase Modal *and* one Decrease Modal per Exercise per Workout.** The two cannot both fire for the same Exercise in the same Workout (logically impossible given the trigger conditions).

### Why "first Set" for the Trigger Rule
The first Set is the freshest and easiest to clear, so the bar is intentionally low — the app errs toward eagerness on progression because (a) casual users need fast positive feedback, and (b) a one-session over-shoot self-corrects on the next session.

### Why "last Set" for the Deload Rule
The last Set is the truest fatigue signal. Asymmetric on purpose: be eager to reward, cautious to take away.

## 5. MVP Feature List

### 5.1 Authentication
- **Google sign-in only.** No email/password.
- Account is required to use the app (cloud sync is from day 1).

### 5.2 Onboarding
1. Google sign-in.
2. Account created with **"Full Body" Starter Routine** pre-loaded.
3. User lands on the home screen with one tap to start the routine.
4. No mandatory unit selection — defaults to `kg`, changeable in settings. *(Optional: detect from locale.)*
5. No mandatory Weigh-in. First Bodyweight Exercise triggers a one-time Bodyweight prompt.

**Target:** install → first completed Workout in **≤ 90 seconds.**

### 5.3 Routines
- Routine-based only. No freestyle.
- Starter Routines pre-loaded; user can edit, duplicate, or create new.
- A Routine has a name and an ordered list of Exercises, each with their per-user **Rep Range**, **Increment**, and current **Working Weight**.

### 5.4 Exercises
- Preset library: **~80 exercises** at launch, covering chest, back, legs, shoulders, arms, core.
- Each exercise carries: `name`, `primaryMuscleGroup`, `equipment` (barbell, dumbbell, machine, bodyweight, cable).
- Users can create **Custom Exercises** with the same fields.
- Presets are read-only; users may not edit or delete them.

### 5.5 Workout Flow
1. User taps a Routine to start a **Workout**.
2. App displays each Exercise in order; user logs Sets.
3. After each Set: **Rest Timer** auto-starts at 90 s (default; per-exercise override allowed).
4. Rest screen has in-moment add buttons: **+15 s / +30 s / +90 s**, plus skip.
5. When a Set's reps meet the **Trigger Rule** or **Deload Rule**, the corresponding Modal fires immediately.
6. User can finish the Workout at any time.
7. Workout summary is saved to History.

### 5.6 Set Logging
Each Set records:
- **reps** (integer)
- **weight** (canonical kg, with separate `addedWeight` field for Bodyweight Exercises)
- **timestamp** (UTC, regardless of user timezone)
- **exercise** (reference)

Not stored: per-set notes, RPE, RIR, warm-up flag.

### 5.7 Bodyweight Tracking
- Users can log a **Weigh-in** at any time from a dedicated screen.
- Weigh-in history is retained indefinitely.
- For **Bodyweight Exercises**, the Set logger shows two fields:
  - **Bodyweight** (auto-filled from latest Weigh-in, editable)
  - **Added Weight** (default 0)
  - Display total = Bodyweight + Added Weight.
- Each Set on a Bodyweight Exercise **snapshots Bodyweight at time of logging**, so historical accuracy is preserved as the user's Bodyweight changes.
- **Stale Bodyweight warning:** if the latest Weigh-in is >30 days old, surface a soft prompt at workout start. Non-blocking.
- Progression on Bodyweight Exercises operates on **Added Weight only.**

### 5.8 Rest Timer
- Default 90 s, auto-starts after Set completion.
- In-moment add buttons: **+15 s, +30 s, +90 s.**
- Skip and reset available.
- Per-exercise default override in settings.

### 5.9 History
- A chronological list of completed Workouts.
- Per-Workout view shows: date/time, Exercises performed, all Sets (reps × weight).
- Per-Exercise history view: chronological list of that Exercise's Sets across all Workouts.
- No charts in MVP — text-based history only.

### 5.10 Notifications
- **Three kinds**, each individually opt-out-able in settings:
  - **Progression Notification** — issued on every Progression Event ("Bench: 60 → 65 kg next session").
  - **Reminder** — workout-cadence nudge (e.g., after 3+ days without a Workout). Server-scheduled.
  - **Achievement Notification** — Streak milestones and PRs.
- **Delivery:** push (via FCM) **and** in-app, simultaneously.
- **Persistence:** every Notification is saved to the user's Notification Center and remains readable indefinitely (not just a push toast).

### 5.11 Settings
- Theme: light / dark / system.
- Locale: device default or override.
- Display Unit: kg / lb.
- Default Rep Range (global).
- Default Increment.
- Default Trigger Rule variant.
- Per-notification-type opt-out toggles.

### 5.12 Theming & i18n
- **Dark and light themes** required at launch.
- **i18n** required at launch; ship at minimum the author's language(s). Add more as needed.
- All user-facing strings externalized — no hardcoded text.

## 6. Architecture & Sync Behavior

### 6.1 Thin client, fat backend
- Business logic — Rep Range evaluation, Progression Event creation, PR calculation, Streak math — lives on the **backend**. The mobile app is a UI shell.
- Reasoning: future iOS / web clients will not need to reimplement logic.

### 6.2 Offline behavior (workout durability)
Casual users will hit gym dead zones. The app must complete a Workout end-to-end with **zero connectivity** mid-session.

- **On app open (online):** download the user's Routines, Exercises, Working Weights, and Progression rules into a local **TTL Cache** (~24h expiry).
- **On workout start:** all needed data is read from cache; the workout runs entirely client-side.
- **During workout (offline-tolerant):** Sets, Progression decisions, and Weigh-ins are written locally and queued.
- **On reconnect:** queued writes flush to the backend. Backend is **Source of Truth** and reconciles any conflicts.
- **Trigger Rule evaluation runs on the client during the workout** for instant Modal response. This is a documented duplication of the server-side evaluator. Both must stay in sync; the server result wins on disagreement.

### 6.3 Data persistence guarantees
- All user data is cloud-backed from the first Workout.
- A phone wipe / device change must not lose data — sign in with Google on a new device, all history returns.

## 7. Data Model Sketch (informal)

> This is a sketch, not a schema. Engineering owns the final shape.

- **User**: googleId, displayUnit, locale, theme, notificationPrefs, defaults (repRange, increment, triggerRule).
- **Exercise** *(global presets)*: id, name, primaryMuscleGroup, equipment.
- **CustomExercise** *(per user)*: id, userId, name, primaryMuscleGroup, equipment.
- **Routine** *(per user)*: id, userId, name, orderedExerciseRefs.
- **UserExerciseConfig** *(per user × exercise)*: workingWeightKg, repRange, increment, triggerRule, restSeconds.
- **Workout**: id, userId, routineId, startedAtUtc, completedAtUtc.
- **Set**: id, workoutId, exerciseRef, reps, weightKg, addedWeightKg, bodyweightSnapshotKg, timestampUtc.
- **WeighIn**: id, userId, bodyweightKg, timestampUtc.
- **ProgressionEvent**: id, userId, exerciseRef, direction (`up`/`down`), beforeKg, afterKg, workoutId, timestampUtc.
- **Notification**: id, userId, kind (`progression`/`reminder`/`achievement`), payload, createdAtUtc, readAtUtc.

## 8. Out of Scope (deferred)

- Pre-built programs (PPL, 5/3/1, nSuns, etc.).
- iOS and web clients.
- Freestyle (no-routine) workouts.
- Charts, analytics, volume-by-muscle-group breakdowns.
- Monetization.
- Public profiles, social features, leaderboards, sharing.
- Warm-up set tracking.
- RPE / RIR / 1RM estimation.
- Per-set notes.
- Multi-user coaching / shared accounts.
- Anonymous (no-account) usage.
- Custom plate calculator.
- Exercise videos / images.

## 9. Open Questions

- **Onboarding unit selection:** ship with `kg` default, or detect from device locale? (Lean: detect locale, default to `kg` on failure.)
- **Reminder cadence:** what gap triggers a "you haven't worked out" Reminder? Lean: 3 days since last Workout.
- **Custom Exercise hard cap:** none, or some safety limit? Lean: no cap for MVP, this is a friends-only app.
- **Routine cap:** none, or limit? Lean: no cap for MVP.
- **Notification Center retention:** keep forever, or auto-purge old entries? Lean: keep forever.
- **Languages at launch:** which two (or more)? Author to specify.

These should be resolved before implementation but do not block PRD sign-off.
