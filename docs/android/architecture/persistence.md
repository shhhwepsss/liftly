# Local Persistence

**Decision: Room.**

SQLDelight produces cleaner SQL-first schemas and shares code with iOS, but Liftly is Android-only at MVP and Room's Kotlin coroutines + Flow integration is better documented and more widely understood. The hybrid approach (Room + DataStore) is the correct call: Room for relational data, DataStore (Proto) for user preferences and auth tokens. SQLite-backed Room handles the offline queue durability requirement reliably — if the process dies mid-workout, the queue survives.

## Tables / Entities

| Entity | Table | Purpose |
|---|---|---|
| `CachedRoutineEntity` | `cached_routines` | Serialized routine + exercise list from the `/me` bootstrap payload |
| `CachedExerciseConfigEntity` | `cached_exercise_configs` | Per-exercise Working Weight, Rep Range, Trigger/Deload Rule config, rest seconds |
| `CachedWeighInEntity` | `cached_weigh_ins` | Latest Weigh-in for bodyweight Exercise prefill |
| `WorkoutEntity` | `workouts` | All Workouts (in-progress and completed). Columns: id (client-UUID PK), routineId, startedAtUtc, completedAtUtc (nullable), startIdempotencyKey, completeIdempotencyKey, startFlushState (pending/flushed), completeFlushState (pending/flushed/n/a) |
| `QueuedSetEntity` | `queued_sets` | Logged Sets not yet flushed; idempotency key, exerciseId, reps, weightKg, addedWeightKg, bodyweightSnapshotKg, loggedAtUtc, flushState |
| `QueuedProgressionDecisionEntity` | `queued_progression_decisions` | Increase/Decrease Modal outcome per Exercise per Workout; direction, decision, proposedAfterKg, clientEvalTriggeredBySetId |
| `QueuedWeighInEntity` | `queued_weigh_ins` | Weigh-ins not yet flushed; same shape as `QueuedSetEntity` |
| `MirroredNotificationEntity` | `notifications` | Server Notification mirror; id, kind, payload (JSON string), createdAtUtc, readAtUtc |
| `SyncMetaEntity` | `sync_meta` | Bootstrap freshness: lastBootstrappedAtUtc, ttlExpiresAtUtc (single row keyed by user) |
| `LastFlushStateEntity` | `last_flush_state` | Per-Workout: last successful flush idempotency key, server-returned `updatedWorkingWeights` (used to recover after process death between flush success and merge commit) |

Note: there is no separate "active workout" table. `WorkoutEntity` with `completedAtUtc IS NULL AND startFlushState IN (pending, flushed)` identifies the in-progress Workout, of which there is at most one at a time. This is what the [Resume in-progress Workout](./sync.md#resume-in-progress-workout) flow keys off.

**Schema migration strategy:** Room `Migration` classes, version-incremented, checked into source control alongside the schema JSON export (`room.schemaLocation`). No auto-migrations at MVP — schema is still in flux. The CI pipeline compiles and runs a migration test against the exported schema on every PR (see [testing strategy](./testing.md)).

## DataStore usage

- `preferences_proto.pb`: theme, locale, display unit, notification opt-outs.
- `auth_proto.pb`: access token, refresh token, token expiry. Stored as Tink-encrypted bytes (AES-GCM) with the wrapping key held in the Android Keystore (key alias `liftly_auth_v1`, generated per install). See [auth & session](./auth.md) — this replaces `EncryptedSharedPreferences`, which Google deprecated in 2024 with no maintained successor.
