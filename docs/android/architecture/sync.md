# Sync Engine

The sync engine lives in `core/sync/`. It is the most critical piece of the client and is designed to survive process death, network flakiness, and server reconciliation disagreements.

## Bootstrap on sign-in / app open

The Android client uses a **single bootstrap endpoint**: `GET /me`. The backend's `/me` response returns the user profile plus everything the client needs to render the home screen and execute any Workout offline:

- User profile + settings (Display Unit, locale, theme, notification prefs, defaults)
- The user's full Routine list with per-Exercise configs (Working Weight, Rep Range, Trigger/Deload Rule, increment, rest seconds)
- Latest Weigh-in + `weighInStaleDays`
- Current Streak count and per-Exercise PR map
- `syncedAtUtc`

This is a cross-bounded-context payload (Account + Workout + Bodyweight + Progression data in one response), accepted as a deliberate trade for cold-start simplicity: one round-trip on app open, no fan-out, no first-time-vs-warm-cache divergence.

**Flow on foreground resume (via `ProcessLifecycleOwner`):**

1. Read `SyncMetaEntity.ttlExpiresAtUtc`. If `< now` (or row missing), call `GET /me`.
2. Write response into `CachedRoutineEntity`, `CachedExerciseConfigEntity`, `CachedWeighInEntity`, and the in-memory account/streak/PR caches in a single Room transaction. Set `ttlExpiresAtUtc = now + 24h`.
3. If offline: serve stale cache. Stale cache is always preferable to blocking the UI. A `SyncStatus.Stale` flag in the ViewModel surfaces a soft banner ("Working from cached data") — non-blocking.

The call is also forced (ignoring TTL) **only on cold start** — `updatedWorkingWeights` returned from a flush is merged directly into `CachedExerciseConfigEntity` without an additional refetch, so the next session opens with current Working Weights without a second network call.

## Mid-workout offline queue

Every write during a Workout goes to Room immediately:

- **Workout started** → insert `WorkoutEntity` with `startFlushState = PENDING` and a client-generated `startIdempotencyKey`.
- **Set logged** → insert `QueuedSetEntity` with `flushState = PENDING`, client-generated UUID PK, separate `idempotencyKey`.
- **Modal decision** → insert `QueuedProgressionDecisionEntity`.
- **Weigh-in** → insert `QueuedWeighInEntity`.
- **Workout completed** → set `WorkoutEntity.completedAtUtc` and `completeFlushState = PENDING`, with a separate `completeIdempotencyKey`.

The queue persists across process death because it is in Room, not in memory.

No network call happens during the workout. The UI reads exclusively from the local queue and cache.

## Resume in-progress Workout

On launch, the app checks for a `WorkoutEntity` with `completedAtUtc IS NULL`. If one exists, navigation **auto-resumes** directly into `ActiveWorkoutScreen` for that Workout, restoring the Set list from `QueuedSetEntity` rows and the Rest Timer state from `RestTimerController` (see [Rest Timer](#rest-timer) below). No prompt, no confirmation dialog — the user is dropped back where they were. This matches the gym pattern where the OS frequently evicts the app between sets.

A "Discard workout" action is reachable from the active workout's overflow menu (rare path).

## Flush on reconnect

**Triggers:**
- `ConnectivityManager.NetworkCallback.onAvailable()` — network regained while app is foregrounded.
- App comes to foreground and has pending queue items.
- WorkManager `FlushWorker` (see background sync below).

**Flush sequence** (`core/sync/FlushWorker.kt`) per Workout, executed strictly in order:

1. **Workout start.** If `WorkoutEntity.startFlushState = PENDING`, POST `POST /workouts { id, routineId, startedAtUtc, idempotencyKey: startIdempotencyKey }`. On 200, mark `startFlushState = FLUSHED`. On 4xx (other than idempotency conflict), surface error and abort the flush for this Workout. The remaining steps require the Workout to exist server-side.
2. **Flush mid-workout writes.** POST `POST /workouts/:id/flush` with the body assembled from `QueuedSetEntity` rows (ordered by `loggedAtUtc`), `QueuedProgressionDecisionEntity` rows (in the same Workout), and `QueuedWeighInEntity` rows. Within the request body each array is independently ordered by timestamp — the server processes types in fixed order (sets → weigh-ins → re-evaluate → progression decisions), so cross-type interleaving in the queue is irrelevant.
3. **Merge reconciliation response into local cache** (Room transaction):
   - Mark all flushed Set / Weigh-in entities `flushState = FLUSHED`.
   - For each entry in `reconciliation[]` where `clientDecision = accepted` and `serverDecision = rejected`: remove the corresponding `QueuedProgressionDecisionEntity`. The Working Weight is NOT touched here — if the server overrode to rejected, no Progression Event was created, and `updatedWorkingWeights` for this Exercise will be absent.
   - Apply `updatedWorkingWeights` map to `CachedExerciseConfigEntity` rows. This is the only authoritative source of post-flush Working Weight changes.
   - Persist `LastFlushStateEntity` with the flush `idempotencyKey` and the server response, so a crash between the network success and the merge commit can replay the merge on next launch.
4. **Workout complete.** If `WorkoutEntity.completeFlushState = PENDING`, POST `POST /workouts/:id/complete { completedAtUtc, idempotencyKey: completeIdempotencyKey }`. On 200, mark `completeFlushState = FLUSHED`.
5. Emit `FlushResult` (success/partial/conflict) to the `SyncEngine` flow, which the active ViewModel consumes.
6. On HTTP 4xx (non-idempotency): surface error to user, do not clear queue.
7. On HTTP 5xx or network error: let WorkManager retry with exponential backoff.

**Conflict UX:** if `reconciliation[]` contains `clientDecision = accepted, serverDecision = rejected` entries, the app shows a non-blocking snackbar: "Weight update adjusted by server after sync." The snackbar includes the Exercise name and the server's final Working Weight. Silent reconcile is wrong here because the user consciously tapped "Yes" on the Increase Modal; they deserve to know the server disagreed.

The inverse case (server creates a Progression Event the client never proposed — e.g., from an unknown rule variant on an outdated client) has **no inline surface** at MVP. The server sends a Progression FCM Notification which lands in the Notification Center; the `updatedWorkingWeights` map silently updates the cache. The plan to prevent the underlying divergence is force-update of the Android client, scheduled post-MVP.

## Online-only writes

Routine CRUD (create, rename, reorder Exercises, delete) and Custom Exercise creation are **online-only at MVP**. These are not on the gym hot path; users typically configure Routines at home with connectivity.

- The action button is disabled with a hint ("You're offline") when `ConnectivityManager` reports no network.
- When online, the UI updates optimistically. On HTTP failure, the optimistic change is rolled back and a snackbar surfaces the error.
- There is no `queued_routine_writes` table — adding one would double the sync surface and reconciliation logic for negligible benefit.

## Background sync

`FlushWorker` is a `CoroutineWorker` registered with WorkManager:

- **Constraints:** `NetworkType.CONNECTED`. No wifi-only requirement (gym LTE is fine). No battery-not-low constraint — losing flush data to a low-battery deferral is worse than the marginal battery cost.
- **Trigger:** enqueued as `OneTimeWorkRequest` whenever items enter the queue and app goes to background, and on app open if the queue is non-empty.
- **Retry:** `BackoffPolicy.EXPONENTIAL`, initial delay 30 s, max 5 attempts.
- **Uniqueness policy:** `ExistingWorkPolicy.KEEP` per workout ID — prevents duplicate concurrent flushes.

WorkManager persists the work request across process death, handling the case where the user completes a Workout, kills the app, and reconnects hours later.

---

## Rest Timer

PRD §5.8: 90 s default, auto-starts after Set completion, in-moment +15 / +30 / +90 buttons, skip, per-exercise override.

**Decision: ViewModel coroutine for in-screen countdown, AlarmManager for the completion cue when backgrounded.**

Implementation lives in `workout/timer/RestTimerController`:

- **Foreground (app visible):** a coroutine inside `ActiveWorkoutViewModel` ticks every second and updates `UiState.restRemainingSeconds`. Compose recomposes the in-screen countdown. +15 / +30 / +90 actions mutate the deadline; Skip cancels the coroutine.
- **Backgrounded:** when the workout screen goes to STOPPED, the controller schedules a one-shot **AlarmManager** alarm at the timer's deadline. The alarm receiver posts a high-priority system notification ("Rest done — next set") with vibration + sound. AlarmManager is required (not `WorkManager`) because the cue must fire at an exact wall-clock time; WorkManager batches and may delay by minutes.
- **App killed mid-rest:** the AlarmManager schedule survives process death and the notification fires regardless. On re-launch (Resume flow above), `RestTimerController` reads the persisted deadline from `WorkoutEntity` (added field: `restDeadlineUtc`, nullable) and either restores the live countdown or marks the timer as already elapsed.
- **Permission:** Android 13+ requires `POST_NOTIFICATIONS`; requested at first workout start, not at app launch.

A foreground Service was rejected as overkill — the audible cue at deadline is the only thing the user actually needs while the phone is in their pocket.
