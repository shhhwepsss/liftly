# Cache & Sync Semantics

## What the client downloads on app open

`GET /routines/:id/sync-payload` (called for each Routine the user has) returns:

```
{
  "routine": { "id", "name", "exercises": [ { "position", "exerciseId", "name", "equipment", ... } ] },
  "configs": {
    "<exerciseId>": {
      "workingWeightKg": 60.0,
      "repRangeLower": 12,
      "repRangeUpper": 15,
      "incrementKg": 5.0,
      "triggerRule": { "variant": "first_set_upper_bound", "setPosition": 0, "threshold": 15 },
      "deloadRule":  { "variant": "last_set_lower_bound",  "setPosition": -1, "threshold": 12 },
      "restSeconds": 90
    }
  },
  "latestWeighInKg": 82.5,
  "weighInStaleDays": 0,
  "syncedAtUtc": "2026-05-19T08:00:00Z"
}
```

The client caches this with a 24-hour TTL. The `syncedAtUtc` timestamp is stored locally; on app open the client compares it against now and re-fetches if expired.

## TTL invalidation vs. mid-workout changes

A Progression Event that fires during workout A updates the server's Working Weight for workout B. If the client is mid-workout A (with a warm cache), the old Working Weight in cache is correct — the Progression Event does not apply until the next session. The flush response's `updatedWorkingWeights` map is the mechanism by which the client learns the new Working Weight, so it can update the cache after workout A completes without waiting for the 24h TTL to expire.

If the user edits a Routine from a second device mid-workout (unlikely but possible), the client will operate on stale cache data for that workout. The server reconciles on flush and the next app open re-syncs. This is acceptable given the single-user, single-device MVP target.

## Idempotency and conflict resolution

- All client-originated writes (Sets, Weigh-ins, Workout creation) carry **client-generated UUIDs** as primary keys and a separate `idempotencyKey` field. The server upserts on `idempotency_key`, making repeated flushes safe.
- **Progression decisions use server-recompute, not last-write-wins.** The client's decision is an input, not a command. The server re-evaluates and may override (see the [flush endpoint](./api.md#offline-flush--reconcile)).
- There is no multi-device real-time merge problem at MVP (Android only, no simultaneous sessions).
