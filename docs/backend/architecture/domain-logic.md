# Domain Logic Placement

All domain logic lives in the backend. Specifically:

- **Trigger Rule / Deload Rule evaluation** → `progression/rule-evaluator.ts`, a pure stateless function `evaluateRules(sets: SetResult[], config: ExerciseConfig): RuleEvaluation`. No I/O, no NestJS dependencies. Invoked directly by use cases that need a rule decision (notably `flush-workout.use-case`), without going through a service.
- **Progression Event creation** → `progression/progression.service.ts` (`createProgressionEvent`, `updateWorkingWeight`). The service knows how to persist an event and adjust the Working Weight; it does **not** know about workouts or flushes. The call is initiated by `workout/use-cases/flush-workout.use-case.ts` after Sets are committed.
- **PR calculation** → `progression/pr.service.ts`, called by `workout/use-cases/complete-workout.use-case.ts` after a Workout is marked complete.
- **Streak math** → `progression/streak.service.ts`, called by `workout/use-cases/complete-workout.use-case.ts` after a Workout is marked complete.
- **Progression Notification creation** → `notification/notification.service.ts`, called by `workout/use-cases/flush-workout.use-case.ts` for each Progression Event that fires.

**Client-side rule duplication strategy: rules-as-config returned by the sync API.**

The client does not receive compiled Kotlin or a shared library. Instead, the sync payload ([cache & sync semantics](./sync.md)) includes the full `ExerciseConfig` for each Exercise in the Routine — Rep Range, Trigger Rule variant, set position evaluated (first/last), threshold. The Android client hard-codes the same evaluation logic against these config values.

**Why:** The Trigger Rule is intentionally simple (one comparison: `set[position].reps >= threshold`). Shipping it as config data keeps the server authoritative over thresholds and variants without requiring a shared runtime. The client re-implements one comparison expression. If the rule ever becomes complex enough that duplication becomes risky, the migration path is a `/evaluate` endpoint the client calls mid-workout — but that requires connectivity and defeats offline-tolerance. Config-driven duplication is the right call at this complexity level.

The server re-evaluates using the same `rule-evaluator.ts` pure function during flush. If the client sent a Progression Event that the server's re-evaluation does not agree with, the server **drops the client's decision and recomputes**, returning a `reconciliation` block in the [flush response](./api.md#offline-flush--reconcile) that tells the client what was overridden.
