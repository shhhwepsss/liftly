# Open Questions & Cross-Doc Follow-ups

## Cross-document follow-ups

The Android decisions imply changes to the [backend architecture](../backend/README.md) that should be made in lockstep:

1. **`/me` becomes the bootstrap endpoint.** The Account module must return the full Routine list with per-Exercise configs, the latest Weigh-in + `weighInStaleDays`, the current Streak count, and a per-Exercise PR map — not just the user profile. The existing `GET /routines/:id/sync-payload` is superseded (or kept as a per-Routine refresh path used only after Routine edits).
2. **`POST /workouts/:id/flush` may run before `POST /workouts/:id/complete`** but always after `POST /workouts`. Backend semantics should be explicit that a Workout exists from the moment of start, not completion.
3. **Force-update support** (post-MVP) — backend will need a `minSupportedAndroidVersion` field in `/me` (or a dedicated `/version-check` endpoint) so the client can hard-block stale installs before they reach the workout flow. Tracked in [[project-liftly-force-update]].

## Remaining open questions

These still require product or technical input:

1. **Starter Routine identity for the home screen.** The PRD says the Starter Routine is pre-loaded on account creation. After the user edits or deletes it, what's pinned at the top of the home screen? (Currently assumed: nothing — Routines render in creation/edit order.)
2. **Streak display when the user has no active streak.** Show "0 weeks" / "Start a streak" / hide the chip entirely? Affects HomeScreen.
3. **PR badge granularity.** Per-Exercise (one badge on the most recent PR Set) or per-Set (badge any Set that beat the running max)? Affects ExerciseHistoryScreen and the per-Exercise PR map shape in `/me`.
