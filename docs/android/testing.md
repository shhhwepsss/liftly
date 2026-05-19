# Testing Strategy

## Mandatory at MVP

**Unit tests (JVM, no Android dependencies):**
- `TriggerRuleEvaluatorTest` — parameterized test covering all rule variants, edge cases, and the at-most-one-modal invariant. This is the highest-value test in the project because it guards the core mechanic.
- `FlushRequestBuilderTest` — verifies the `FlushRequest` payload is assembled correctly from queued entities (correct ordering, idempotency keys propagated, Bodyweight Exercise fields populated).
- `SyncEngineTest` — verifies TTL invalidation logic, stale-cache decision, Working Weight merge from `updatedWorkingWeights`, and reconciliation handling.
- `SettingsViewModelTest` — locale and theme preference flow.
- `WeightFormatterTest` / `TimestampFormatterTest` — kg↔lb conversion at unit-of-display boundary cases; UTC↔local across DST transitions.

**Integration tests (Android JVM with Robolectric or on-device with `@SmallTest`):**
- `WorkoutRepositoryTest` — in-memory Room database, verifies Set queue insertion, flush-state transitions, and that `QueuedSetEntity` survives a simulated process restart (Room close + reopen).
- `WorkoutLifecycleOfflineTest` — start a Workout offline, log Sets offline, complete offline, then unblock the network and verify FlushWorker executes start → flush → complete in order with correct idempotency keys.
- `NotificationRepositoryTest` — upsert idempotency, unread count derivation, offline mark-as-read reconciliation on next GET.

**UI tests (Compose test + Espresso, on-device or emulator):**
- `SetLoggingFlowTest` — open Routine → start Workout → log Set at trigger threshold → assert Increase Modal appears → tap Yes → assert modal dismissed and progression decision queued.
- `OfflineQueueTest` — start Workout while `FakeNetworkInterceptor` blocks all calls → log Set → assert Set appears in UI → unblock network → assert FlushWorker runs and clears queue end-to-end.
- `ResumeWorkoutTest` — start Workout, log a Set, simulate process kill, re-launch, assert auto-resume into the same Workout with Set list intact.

## Deferred (post-MVP)

- Screenshot tests (Paparazzi or Roborazzi) for theme/dark-light consistency.
- End-to-end tests against a test backend environment.
- Performance tests for the ≤ 5-second Set-log target (currently architectural — no network on the hot path).
- Rule evaluator fuzz tests (random Set sequences against known rule configs).
