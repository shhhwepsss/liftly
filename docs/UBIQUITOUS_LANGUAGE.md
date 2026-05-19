# Ubiquitous Language

Shared vocabulary for Liftly. Every term used in product, code, UI copy, and team conversation should match the definitions here. Definitions describe **what each concept *is***, not how it is implemented, defaulted, or displayed. If a definition needs to change, update this file first.

## Workout Domain

- **Exercise** — A named movement (e.g., "Bench Press"). Either a *Preset Exercise* or a *Custom Exercise*.
- **Preset Exercise** — A globally curated Exercise available to all users.
- **Custom Exercise** — An Exercise defined by a single user and visible only to them.
- **Routine** — A reusable, ordered template of Exercises.
- **Starter Routine** — A Routine that exists in a user's account without the user having created it.
- **Workout** *(also: Session)* — A single instance of a user executing a Routine. Has a start time, an end time, and an ordered list of Sets.
- **Set** — One bout of reps at a given weight on a specific Exercise within a Workout. The atomic unit of logged work.
- **Working Weight** — The weight currently programmed for a user on a given Exercise.

## Progression Domain

- **Rep Range** — A target range of reps for an Exercise, defined by a lower and an upper bound.
- **Trigger Rule** — The condition under which a user has earned an increase in *Working Weight* for an Exercise.
- **Deload Rule** — The condition under which a user's *Working Weight* for an Exercise should be reduced.
- **Increment** — The weight delta applied when *Working Weight* changes.
- **Progression Event** — A change to *Working Weight* for an Exercise, recorded with a direction (increase or decrease) and applied to subsequent Workouts.
- **Increase Modal** — The user-facing prompt asking whether to apply an increase to *Working Weight*.
- **Decrease Modal** — The user-facing prompt asking whether to apply a decrease to *Working Weight*.

## Bodyweight Domain

- **Bodyweight** — A user's self-mass.
- **Weigh-in** — A timestamped Bodyweight measurement.
- **Bodyweight Exercise** — An Exercise whose load is the user's own body (e.g., pull-ups, dips).
- **Added Weight** — External load on a *Bodyweight Exercise* (e.g., weight on a dip belt).

## Notification Domain

- **Notification** — A persisted record of a noteworthy event in the user's training.
- **Progression Notification** — A Notification that records a *Progression Event*.
- **Reminder** — A Notification nudging the user to work out.
- **Achievement Notification** — A Notification marking a *Streak* or *PR* milestone.
- **Streak** — A count of consecutive weeks in which the user completed at least one Workout.
- **PR (Personal Record)** — The heaviest *Working Weight* a user has completed at the upper bound of an Exercise's *Rep Range*.

## Account & Settings

- **Display Unit** — A user's preferred unit for weight: kilograms or pounds.
- **Theme** — A user's preferred visual mode: light or dark.
- **Locale** — A user's preferred language for the application.

## Domain Invariants

- A *Workout* always belongs to a *Routine*.
- A *Set* always belongs to a *Workout* and references an *Exercise*.
- *Working Weight* is a per-user property of an Exercise, not a property of the Exercise itself.
- A *Progression Event* applies to subsequent Workouts, never to the Workout that produced it.
- Each Set on a *Bodyweight Exercise* is associated with the user's *Bodyweight* at the time the Set was logged.
