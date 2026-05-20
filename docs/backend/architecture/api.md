# API Design

All endpoints require `Authorization: Bearer <jwt>` except `POST /auth/google`. All timestamps are UTC ISO 8601.

## Auth

| Method | Path | Notes |
|---|---|---|
| `POST` | `/auth/google` | Body: `{ idToken: string }`. Returns `{ accessToken, refreshToken, user }`. |
| `POST` | `/auth/refresh` | Body: `{ refreshToken }`. Returns `{ accessToken }`. |

## Account & Settings

| Method | Path | Notes |
|---|---|---|
| `GET` | `/me` | Returns User profile + settings. |
| `PATCH` | `/me` | Update display unit, locale, theme, defaults. |
| `POST` | `/me/fcm-token` | Register FCM token. Body: `{ token }`. |

## Exercises

| Method | Path | Notes |
|---|---|---|
| `GET` | `/exercises` | Returns preset library + user's Custom Exercises. |
| `POST` | `/exercises` | Create Custom Exercise. |
| `DELETE` | `/exercises/:id` | Delete Custom Exercise (own only). |

## Routines & Working Weights

| Method | Path | Notes |
|---|---|---|
| `GET` | `/routines` | Returns user's Routines with ordered Exercise list and current Working Weights. |
| `POST` | `/routines` | Create Routine. |
| `PATCH` | `/routines/:id` | Update name / exercise order. |
| `DELETE` | `/routines/:id` | Delete Routine. |
| `GET` | `/routines/:id/sync-payload` | **App-open sync.** Returns full Routine config, per-Exercise Working Weights, Rep Ranges, Trigger/Deload Rule configs, and latest Weigh-in. This is what populates the 24h TTL cache. |

## Workout Lifecycle

| Method | Path | Notes |
|---|---|---|
| `POST` | `/workouts` | Start Workout. Body: `{ id: clientUuid, routineId, startedAtUtc, idempotencyKey }`. Idempotent. |
| `GET` | `/workouts` | Workout history (paginated, newest-first). |
| `GET` | `/workouts/:id` | Single Workout detail with all Sets. |
| `POST` | `/workouts/:id/complete` | Mark Workout complete. Body: `{ completedAtUtc }`. Triggers PR + Streak eval. |

## Offline Flush / Reconcile

**`POST /workouts/:id/flush`** — This is the most important endpoint.

**What the client sends:**
```
{
  "idempotencyKey": "<uuid>",
  "sets": [
    {
      "id": "<client-uuid>",
      "exerciseId": "...",
      "reps": 15,
      "weightKg": 60.0,
      "addedWeightKg": null,
      "bodyweightSnapshotKg": null,
      "loggedAtUtc": "2026-05-19T09:14:00Z",
      "idempotencyKey": "<uuid>"
    }
  ],
  "progressionDecisions": [
    {
      "exerciseId": "...",
      "decision": "accepted" | "rejected",
      "direction": "up" | "down",
      "proposedAfterKg": 65.0,
      "clientEvalTriggeredBySetId": "<set-uuid>"
    }
  ],
  "weighIns": [
    {
      "id": "<client-uuid>",
      "bodyweightKg": 82.5,
      "measuredAtUtc": "2026-05-19T09:00:00Z",
      "idempotencyKey": "<uuid>"
    }
  ]
}
```

**What the server does** — orchestrated entirely by `flush-workout.use-case.ts` inside a single transaction (see [persistence layer](./persistence.md)):
1. `workout.service.upsertSets(...)` — upsert all Sets by `idempotency_key` (already-stored sets are no-ops).
2. `bodyweight.service.upsertWeighIns(...)` — upsert Weigh-ins.
3. `rule-evaluator.evaluateRules(...)` — re-evaluate Trigger/Deload Rules server-side using the pure function.
4. For each `progressionDecisions` entry, reconcile and dispatch:
   - If client said `accepted` and server agrees → `progression.service.createProgressionEvent(...)` + `progression.service.updateWorkingWeight(...)` + `notification.service.createProgressionNotification(...)`.
   - If client said `accepted` but server disagrees → server **overrides to rejected**, logs reconciliation. No service calls beyond a structured log line.
   - If client said `rejected` but server would have triggered → server **does not force acceptance** (user explicitly declined).
5. The use case assembles the response below from its in-memory results; controller serializes.

No service in this list imports another. The use case is the only place that knows the full sequence.

**What the server returns:**
```
{
  "flushedSetIds": ["<uuid>", ...],
  "flushedWeighInIds": ["<uuid>", ...],
  "progressionEvents": [ { "exerciseId": "...", "direction": "up", "afterKg": 65.0 } ],
  "reconciliation": [
    {
      "exerciseId": "...",
      "clientDecision": "accepted",
      "serverDecision": "rejected",
      "reason": "trigger_rule_not_met"
    }
  ],
  "updatedWorkingWeights": { "<exerciseId>": 65.0 }
}
```

**Conflict semantics:** idempotency keys make repeated flushes safe. The `reconciliation` array is the explicit surface where client/server disagreements are expressed. The client must update its local cache with `updatedWorkingWeights` after a successful flush.

## Bodyweight

| Method | Path | Notes |
|---|---|---|
| `POST` | `/weigh-ins` | Log a Weigh-in. Accepts client-generated UUID + idempotency key. |
| `GET` | `/weigh-ins` | Paginated Weigh-in history. |

## Notifications

| Method | Path | Notes |
|---|---|---|
| `GET` | `/notifications` | Paginated feed, newest-first. Includes unread count in headers. |
| `PATCH` | `/notifications/:id/read` | Mark single Notification read. |
| `POST` | `/notifications/read-all` | Mark all read. |
