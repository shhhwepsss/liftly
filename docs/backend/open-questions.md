# Open Questions

These require product input before the relevant module is implemented:

1. **Routine exercise polymorphism.** `routine_exercises.exercise_id` needs to reference either `exercises` (presets) or `custom_exercises` (per-user). Options: a nullable dual FK, a single `exercises` table with a `user_id` nullable column, or a polymorphic type column. Decision affects the schema and the sync payload query. **Lean: merge into a single `exercises` table with `user_id = NULL` for presets and a `is_preset` flag.**

2. **Working Weight canonical location.** The PRD sketch has `UserExerciseConfig.workingWeightKg`. Two valid approaches: (a) store it directly and update it on each Progression Event; (b) derive it by replaying Progression Events (pure audit log). Approach (a) is simpler and fast to read. Approach (b) gives a full audit trail but requires a reduce on read. **Lean: store directly in `user_exercise_configs`, keep `progression_events` as the audit log.**

3. **Starter Routine seeding.** The PRD says a "Full Body Starter Routine" is pre-loaded on account creation. Is this seeded from a `starter_routines` table (cloned per user at sign-up) or templated in code? Does it vary by user locale/unit preference? If the exercise list or default weights ever change, should existing users get updates? This needs a decision before `account.service.ts` is written.

4. **Progression Event — user declined.** The PRD says "No → no change, no record beyond the user's choice." If the server re-evaluates and also concludes the rule was met, but the client sent `decision: rejected`, the server currently respects that decision (see the [flush endpoint](./api.md#offline-flush--reconcile)). Confirm: a user can indefinitely decline a triggered Increase Modal with no server-side consequence. If so, is there a future "overdue progression" notification type?

5. **PR definition edge case.** The UL defines PR as "heaviest Working Weight completed at the upper bound of the Rep Range." Does this mean the Set must have `reps >= repRangeUpper`, or `reps == repRangeUpper`? And does weight refer to `weightKg` (total) or `addedWeightKg` for Bodyweight Exercises? The PR query in `pr.service.ts` depends on this.

6. **FCM token rotation.** Android clients rotate FCM tokens. The current model collects all tokens per user in `fcm_tokens`. Should stale tokens be pruned (e.g., on FCM delivery failure returning `registration-token-not-registered`)? This is a reliability concern worth deciding before the FCM dispatcher is written.

7. **Reminder cadence configurability.** PRD §9 leans toward 3 days. Is this hardcoded or a per-user setting? If it becomes a setting, it needs a column in `users` and a field in `GET /me`.
