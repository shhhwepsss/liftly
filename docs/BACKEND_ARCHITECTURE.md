# Liftly — Backend Architecture Plan

**Status:** Draft for implementation  
**Stack:** Node.js + NestJS + TypeScript + PostgreSQL  
**Last updated:** 2026-05-19

---

## 1. Architectural Style & Rationale

Liftly uses a **modular monolith with domain-aligned NestJS modules**, each structured in a four-layer pattern: **controller → use case → service → repository**. This is not hexagonal architecture — the overhead of ports/adapters is unjustified for a one-person MVP — but modules are internally cohesive enough that extracting a service later (if Streak math or Progression logic grows complex) requires moving files, not redesigning boundaries.

The use-case layer is mandatory and load-bearing:

- **Controllers** are thin HTTP adapters. They parse the request, call exactly one use case, and shape the response. They do not inject services.
- **Use cases** are medium-sized orchestrators. One use case == one user-facing operation. They are the **only** layer permitted to depend on more than one service, and they are where cross-module coordination lives (e.g., flushing a workout writes Sets, re-evaluates Progression rules, and creates Notifications — all behind one use case).
- **Services** own a single module's domain logic and depend only on their own module's repositories and pure domain helpers. **Services never import other services**, in their own module or any other. Any cross-service work — including same-module cross-service work — goes through a use case.
- **Repositories** wrap Drizzle calls for one aggregate.

Even trivial CRUD endpoints route through a use case. The cost is one delegating file per endpoint; the benefit is uniformity — there's one place to add transactions, audit logging, or cross-cutting orchestration later without rewriting controllers.

The driving constraint is **fat backend**: Rep Range evaluation, Progression Event creation, PR calculation, and Streak math must live server-side so future iOS/web clients get them for free. DDD-lite is applied at the module boundary level — the `progression` module owns its domain rules and does not leak evaluation logic into `workout` or `notification`. Modules communicate **through use cases that compose services across modules** (never through one service importing another), which prevents the most common monolith trap of implicit coupling.


---

## 2. Module / Bounded-Context Layout

Seven domain NestJS modules plus a `background` module map to the UL domains. Each module owns its controllers, use cases, services, and repositories. Use cases live in a per-module `use-cases/` folder. Files marked **(orchestrator)** below import services from more than one module; all other use cases delegate 1:1 to their own module's service.

```
src/
├── main.ts
├── app.module.ts
│
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts                          # POST /auth/google, POST /auth/refresh
│   ├── use-cases/
│   │   ├── authenticate-with-google.use-case.ts    # (orchestrator) auth + account
│   │   └── refresh-access-token.use-case.ts
│   ├── auth.service.ts                             # JWT issuance, refresh-token storage
│   ├── google-token.verifier.ts                    # Wraps google-auth-library
│   ├── jwt.strategy.ts
│   └── refresh-token.repository.ts
│
├── account/
│   ├── account.module.ts
│   ├── account.controller.ts                       # GET/PATCH /me, POST /me/fcm-token
│   ├── use-cases/
│   │   ├── get-my-profile.use-case.ts
│   │   ├── update-my-profile.use-case.ts
│   │   └── register-fcm-token.use-case.ts          # (orchestrator) account + notification
│   ├── account.service.ts                          # User upsert, settings, defaults
│   └── account.repository.ts
│
├── workout/
│   ├── workout.module.ts
│   ├── workout.controller.ts                       # Workout CRUD, flush, complete
│   ├── use-cases/
│   │   ├── start-workout.use-case.ts
│   │   ├── list-workouts.use-case.ts
│   │   ├── get-workout-detail.use-case.ts
│   │   ├── flush-workout.use-case.ts               # (orchestrator) workout + bodyweight + exercise + progression + notification
│   │   └── complete-workout.use-case.ts            # (orchestrator) workout + progression (PR + Streak)
│   ├── workout.service.ts                          # Workout lifecycle, Set persistence (no cross-module deps)
│   ├── workout.repository.ts
│   └── set.repository.ts
│
├── progression/
│   ├── progression.module.ts
│   ├── progression.controller.ts                   # Routine CRUD, sync payload, Working Weight reads
│   ├── use-cases/
│   │   ├── list-routines.use-case.ts               # (orchestrator) progression + exercise
│   │   ├── create-routine.use-case.ts              # (orchestrator) progression + exercise (validates exerciseIds)
│   │   ├── update-routine.use-case.ts
│   │   ├── delete-routine.use-case.ts
│   │   └── get-sync-payload.use-case.ts            # (orchestrator) progression + exercise + bodyweight
│   ├── progression.service.ts                      # Trigger/Deload Rule eval, Progression Event persistence, Working Weight updates
│   ├── pr.service.ts                               # PR calculation (own-module only)
│   ├── streak.service.ts                           # Streak math (own-module only)
│   ├── rule-evaluator.ts                           # Pure, stateless evaluator — shared contract (see §4)
│   ├── routine.repository.ts
│   ├── user-exercise-config.repository.ts
│   └── progression-event.repository.ts
│
├── bodyweight/
│   ├── bodyweight.module.ts
│   ├── bodyweight.controller.ts                    # POST /weigh-ins, GET /weigh-ins
│   ├── use-cases/
│   │   ├── log-weigh-in.use-case.ts
│   │   └── list-weigh-ins.use-case.ts
│   ├── bodyweight.service.ts                       # Weigh-in persistence, latest-Weigh-in lookup, staleness check
│   └── bodyweight.repository.ts
│
├── notification/
│   ├── notification.module.ts
│   ├── notification.controller.ts                  # GET /notifications, PATCH /:id/read, POST /read-all
│   ├── use-cases/
│   │   ├── list-notifications.use-case.ts
│   │   ├── mark-notification-read.use-case.ts
│   │   └── mark-all-notifications-read.use-case.ts
│   ├── notification.service.ts                     # Notification persistence + FCM dispatch enqueue
│   ├── fcm.client.ts                               # Thin wrapper around Firebase Admin SDK
│   ├── fcm-token.repository.ts
│   └── notification.repository.ts
│
├── exercise/
│   ├── exercise.module.ts
│   ├── exercise.controller.ts                      # GET /exercises, POST/DELETE custom
│   ├── use-cases/
│   │   ├── list-exercises.use-case.ts
│   │   ├── create-custom-exercise.use-case.ts
│   │   └── delete-custom-exercise.use-case.ts
│   ├── exercise.service.ts
│   └── exercise.repository.ts
│
└── background/
    ├── background.module.ts
    ├── jobs/
    │   ├── reminder-check.job.ts                   # Calls evaluate-reminders.use-case
    │   └── fcm-dispatch.job.ts                     # Calls notification.service.dispatch (own module)
    └── use-cases/
        └── evaluate-reminders.use-case.ts          # (orchestrator) workout + account + notification
```

**Dependency graph (use-case edges only — services have no cross-module edges):**

```
auth/use-cases               → auth.service, account.service
account/use-cases            → account.service, notification.service          (FCM token registration)
workout/use-cases            → workout.service, bodyweight.service, exercise.service,
                               progression.service, notification.service       (flush + complete)
progression/use-cases        → progression.service, exercise.service,
                               bodyweight.service                              (sync payload, routine reads)
bodyweight/use-cases         → bodyweight.service
notification/use-cases       → notification.service
exercise/use-cases           → exercise.service
background/use-cases         → workout.service, account.service,
                               notification.service                            (reminder evaluation)
```

No service appears on the right-hand side of another service's dependencies — every cross-module arrow is owned by a use case. The dependency graph is acyclic: use cases sit above services, and services sit above repositories within their own module.

**Why this layering matters concretely.**

- `flush-workout.use-case.ts` is the single source of truth for what a flush does. The transaction wrapper, ordering (persist Sets → upsert Weigh-ins → re-evaluate rules → write Progression Events → enqueue Progression Notifications), and reconciliation building all live here. Splitting this across `workout.service` and `progression.service` (the previous draft) leaked workout-shaped concerns into the progression module.
- `complete-workout.use-case.ts` calls `pr.service` and `streak.service` after marking the workout complete. Neither PR nor Streak math need to know that workouts exist — they receive the inputs they need and return their results.
- `evaluate-reminders.use-case.ts` (called from the `reminder-check` job) reads recent workouts (`workout.service`), checks user reminder preferences (`account.service`), and enqueues a Progression-or-reminder Notification (`notification.service`). The job itself is a thin scheduler hook; the orchestration is the use case so it's testable without `pg-boss` in the loop.

**Open conventions** (flag before scaffolding the first use case):

- **File layout (confirmed):** `src/<module>/use-cases/<verb-noun>.use-case.ts`, class name `<VerbNoun>UseCase`, single `execute(input)` method.
- **Transactions:** the use case is the natural place for a transaction boundary — orchestrators that touch multiple repositories run inside `db.transaction(tx => ...)` opened in the use case, with services accepting an optional transactional client. *Open:* whether services should ever start their own transactions for single-aggregate writes — deferred until the first multi-service orchestrator is scaffolded.
- **Return shapes:** use cases return plain DTOs (not ORM entities); controllers serialize them with no business logic.

---

## 3. Persistence Layer

**ORM: Drizzle.**

Drizzle gives a TypeScript-first schema (plain `pgTable` definitions in code), a SQL-like query builder with full type inference, and a migration tool (`drizzle-kit`) that diffs the schema against the live database and emits plain `.sql` files we can read and commit. It runs as a thin layer over `postgres` (porsager) or `pg` — no separate query-engine binary, no codegen step, no shadow database.

**Why not Prisma:**
- **Runtime overhead.** Prisma ships a Rust query engine that the Node process talks to over an IPC channel. For a small backend that overhead is real and unjustified.
- **Shadow database.** Prisma's migrate workflow provisions a throwaway "shadow" database to compute migration diffs. It's a leaky abstraction that complicates local dev and CI, and the value (drift detection) doesn't pay for the friction at this scale.
- **Codegen + CLI overhead.** `prisma generate` after every schema change, a separate generated client, and a CLI whose commands (`db push`, `migrate dev`, `migrate deploy`, `migrate reset`) overlap confusingly. Drizzle's `drizzle-kit generate` → commit SQL → `drizzle-kit migrate` is shorter and more transparent.
- **Studio is not a benefit.** A GUI table viewer doesn't carry weight in the decision; `psql`, DBeaver, or any standard tool covers it.

**Why not Kysely / raw SQL / TypeORM:**
- **Kysely** is a pure query builder with no schema modeling. Liftly's schema is small but not trivial, and a single source of truth in TS (Drizzle's `pgTable`) is worth the small amount of magic over hand-managing types and SQL separately.
- **Raw SQL** (`postgres` directly) is viable but moves more friction onto every query — type-safe schema definitions and relational helpers (`with`, `relations`) pay for themselves quickly.
- **TypeORM** is ruled out for the same reason as before: decorator-based modeling conflates domain and persistence in ways that hurt the repository pattern here.

### Migration workflow

```
1. Edit src/db/schema.ts (the canonical schema)
2. npx drizzle-kit generate         → writes a numbered SQL migration to drizzle/
3. Review the generated SQL, commit alongside the schema change
4. npx drizzle-kit migrate          → applies pending migrations to the target DB
```

Generated migrations are plain SQL files — reviewed, version-controlled, and applied identically in CI, staging, and prod. There is no shadow database.

### Table list

The full schema — tables, columns, enums, indexes, and foreign keys — lives in [`schema.dbml`](./schema.dbml) (DBML format, renderable at [dbdiagram.io](https://dbdiagram.io)). Edit that file when adding or modifying tables; this section only summarizes the table set and calls out non-obvious indexes (see below).

**Tables** (see `schema.dbml` for full definitions):

- `users` — identity, settings, per-user defaults for Rep Range / increment / Trigger Rule.
- `exercises` — preset Exercise library (`is_preset = true`).
- `custom_exercises` — per-user custom Exercises. See [§11 Q1](#11-open-questions) on possible merge into `exercises`.
- `routines`, `routine_exercises` — user Routines and their ordered Exercise list.
- `user_exercise_configs` — per-user, per-Exercise Working Weight, Rep Range, increment, Trigger Rule.
- `workouts`, `sets` — Workout lifecycle and individual Set records (client-generated UUIDs + idempotency keys).
- `weigh_ins` — Bodyweight log (client-generated UUIDs + idempotency keys).
- `progression_events` — audit log of every Working Weight change (direction, before/after, originating Workout).
- `notifications`, `fcm_tokens` — in-app Notification feed and registered FCM tokens (one user → many tokens).

**Soft-delete policy:** none at MVP. The app is invite-only; account deletion is out of scope per PRD §8.

**Non-obvious indexes:**
- PR calculation walks `sets` filtered by `(user_id, exercise_id)` ordered by `logged_at_utc DESC`, looking for max `weight_kg` where `reps >= rep_range_upper`. The composite index on sets covers this scan.
- Streak math requires the most recent ~52 workout `completed_at_utc` values per user; the `(user_id, completed_at_utc DESC)` index on `workouts` covers it.
- The notification feed is append-only and only ever queried newest-first; the DESC index avoids a sort.

---

## 4. Domain Logic Placement

All domain logic lives in the backend. Specifically:

- **Trigger Rule / Deload Rule evaluation** → `progression/rule-evaluator.ts`, a pure stateless function `evaluateRules(sets: SetResult[], config: ExerciseConfig): RuleEvaluation`. No I/O, no NestJS dependencies. Invoked directly by use cases that need a rule decision (notably `flush-workout.use-case`), without going through a service.
- **Progression Event creation** → `progression/progression.service.ts` (`createProgressionEvent`, `updateWorkingWeight`). The service knows how to persist an event and adjust the Working Weight; it does **not** know about workouts or flushes. The call is initiated by `workout/use-cases/flush-workout.use-case.ts` after Sets are committed.
- **PR calculation** → `progression/pr.service.ts`, called by `workout/use-cases/complete-workout.use-case.ts` after a Workout is marked complete.
- **Streak math** → `progression/streak.service.ts`, called by `workout/use-cases/complete-workout.use-case.ts` after a Workout is marked complete.
- **Progression Notification creation** → `notification/notification.service.ts`, called by `workout/use-cases/flush-workout.use-case.ts` for each Progression Event that fires.

**Client-side rule duplication strategy: rules-as-config returned by the sync API.**

The client does not receive compiled Kotlin or a shared library. Instead, the sync payload (§8) includes the full `ExerciseConfig` for each Exercise in the Routine — Rep Range, Trigger Rule variant, set position evaluated (first/last), threshold. The Android client hard-codes the same evaluation logic against these config values.

**Why:** The Trigger Rule is intentionally simple (one comparison: `set[position].reps >= threshold`). Shipping it as config data keeps the server authoritative over thresholds and variants without requiring a shared runtime. The client re-implements one comparison expression. If the rule ever becomes complex enough that duplication becomes risky, the migration path is a `/evaluate` endpoint the client calls mid-workout — but that requires connectivity and defeats offline-tolerance. Config-driven duplication is the right call at this complexity level.

The server re-evaluates using the same `rule-evaluator.ts` pure function during flush. If the client sent a Progression Event that the server's re-evaluation does not agree with, the server **drops the client's decision and recomputes**, returning a `reconciliation` block in the flush response (§5) that tells the client what was overridden.

---

## 5. API Design

All endpoints require `Authorization: Bearer <jwt>` except `POST /auth/google`. All timestamps are UTC ISO 8601.

### Auth

| Method | Path | Notes |
|---|---|---|
| `POST` | `/auth/google` | Body: `{ idToken: string }`. Returns `{ accessToken, refreshToken, user }`. |
| `POST` | `/auth/refresh` | Body: `{ refreshToken }`. Returns `{ accessToken }`. |

### Account & Settings

| Method | Path | Notes |
|---|---|---|
| `GET` | `/me` | Returns User profile + settings. |
| `PATCH` | `/me` | Update display unit, locale, theme, defaults. |
| `POST` | `/me/fcm-token` | Register FCM token. Body: `{ token }`. |

### Exercises

| Method | Path | Notes |
|---|---|---|
| `GET` | `/exercises` | Returns preset library + user's Custom Exercises. |
| `POST` | `/exercises` | Create Custom Exercise. |
| `DELETE` | `/exercises/:id` | Delete Custom Exercise (own only). |

### Routines & Working Weights

| Method | Path | Notes |
|---|---|---|
| `GET` | `/routines` | Returns user's Routines with ordered Exercise list and current Working Weights. |
| `POST` | `/routines` | Create Routine. |
| `PATCH` | `/routines/:id` | Update name / exercise order. |
| `DELETE` | `/routines/:id` | Delete Routine. |
| `GET` | `/routines/:id/sync-payload` | **App-open sync.** Returns full Routine config, per-Exercise Working Weights, Rep Ranges, Trigger/Deload Rule configs, and latest Weigh-in. This is what populates the 24h TTL cache. |

### Workout Lifecycle

| Method | Path | Notes |
|---|---|---|
| `POST` | `/workouts` | Start Workout. Body: `{ id: clientUuid, routineId, startedAtUtc, idempotencyKey }`. Idempotent. |
| `GET` | `/workouts` | Workout history (paginated, newest-first). |
| `GET` | `/workouts/:id` | Single Workout detail with all Sets. |
| `POST` | `/workouts/:id/complete` | Mark Workout complete. Body: `{ completedAtUtc }`. Triggers PR + Streak eval. |

### Offline Flush / Reconcile

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

**What the server does** — orchestrated entirely by `flush-workout.use-case.ts` inside a single transaction (mechanism TBD with the ORM choice, see §3):
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

### Bodyweight

| Method | Path | Notes |
|---|---|---|
| `POST` | `/weigh-ins` | Log a Weigh-in. Accepts client-generated UUID + idempotency key. |
| `GET` | `/weigh-ins` | Paginated Weigh-in history. |

### Notifications

| Method | Path | Notes |
|---|---|---|
| `GET` | `/notifications` | Paginated feed, newest-first. Includes unread count in headers. |
| `PATCH` | `/notifications/:id/read` | Mark single Notification read. |
| `POST` | `/notifications/read-all` | Mark all read. |

---

## 6. Auth

**Strategy: stateless JWT with short-lived access tokens + long-lived refresh tokens, both stored server-side as revocable records.**

Flow:
1. Android obtains a Google ID token via Google Sign-In SDK.
2. Client sends `POST /auth/google { idToken }`.
3. Server verifies the token using the Firebase Admin SDK (or `google-auth-library`) — this validates signature against Google's public keys and confirms `aud` matches the app's client ID.
4. Server upserts a `User` record keyed on `googleSub` from the token claims.
5. Server issues a JWT access token (15-minute expiry) and a refresh token (UUID stored in a `refresh_tokens` table, 30-day expiry).
6. Client uses access token for all API calls. On 401, client exchanges refresh token for a new access token via `POST /auth/refresh`.

**Why not fully stateless (refresh token in JWT):** For a tiny private app the ops cost of a `refresh_tokens` table is near zero, and it buys immediate revocation (useful if a test device is lost). Fully stateless refresh tokens cannot be revoked without a denylist — same complexity, less control.

**Why not sessions/cookies:** The client is Android, not a browser. Bearer tokens are the natural fit and avoid cookie/CSRF complexity.

---

## 7. Background Jobs

**Recommendation: `pg-boss` (PostgreSQL-backed job queue). No Redis.**

`pg-boss` runs entirely within the existing PostgreSQL instance: it creates its own schema and uses `SKIP LOCKED` for reliable at-least-once delivery. For Liftly's scale (a handful of users, a few reminder jobs per day) this is sufficient.

**Jobs:**
- **`reminder.check`** — runs daily via `pg-boss` scheduled job. The job is a thin scheduler hook that calls `background/use-cases/evaluate-reminders.use-case.ts`, which queries recent workouts (`workout.service`) and user reminder preferences (`account.service`), then enqueues an FCM dispatch job via `notification.service`.
- **`fcm.dispatch`** — worker that sends a single FCM push via Firebase Admin SDK. Calls `notification.service` within its own module (no use case needed — single-service work). Decoupled from the request thread so a slow FCM call never delays the flush response.
- **PR / Streak evaluation** — triggered inline by `workout/use-cases/complete-workout.use-case.ts` (not via job queue) because they are fast synchronous DB reads. The use case calls `pr.service` and `streak.service` directly.

**Why not BullMQ + Redis:** Redis is real operational overhead — another service to deploy, monitor, and back up. At this scale, `pg-boss` eliminates the dependency entirely. If job throughput ever demands Redis (it won't for a friends-only app), the migration is a queue-client swap.

**Why not `@Cron` (NestJS/node-cron):** `@Cron` runs in-process and is not durable — a deploy during the cron window loses the job silently. `pg-boss` survives restarts.

---

## 8. Cache & Sync Semantics

### What the client downloads on app open

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

### TTL invalidation vs. mid-workout changes

A Progression Event that fires during workout A updates the server's Working Weight for workout B. If the client is mid-workout A (with a warm cache), the old Working Weight in cache is correct — the Progression Event does not apply until the next session. The flush response's `updatedWorkingWeights` map is the mechanism by which the client learns the new Working Weight, so it can update the cache after workout A completes without waiting for the 24h TTL to expire.

If the user edits a Routine from a second device mid-workout (unlikely but possible), the client will operate on stale cache data for that workout. The server reconciles on flush and the next app open re-syncs. This is acceptable given the single-user, single-device MVP target.

### Idempotency and conflict resolution

- All client-originated writes (Sets, Weigh-ins, Workout creation) carry **client-generated UUIDs** as primary keys and a separate `idempotencyKey` field. The server upserts on `idempotency_key`, making repeated flushes safe.
- **Progression decisions use server-recompute, not last-write-wins.** The client's decision is an input, not a command. The server re-evaluates and may override (see §5 flush endpoint).
- There is no multi-device real-time merge problem at MVP (Android only, no simultaneous sessions).

---

## 9. Deployment Surface

### Dockerfile (multi-stage)

```
Stage 1 (builder):  node:22-alpine → install all deps, compile TypeScript
Stage 2 (runner):   node:22-alpine → copy dist/ + node_modules (prod only)
Expose: 3000
CMD: node dist/main.js
```

Migrations run as a separate one-shot command (`npx drizzle-kit migrate`) in the deploy pipeline before the new container starts — not at runtime inside the app process. The `drizzle/` migration folder is baked into the image so the migrator and the app run identical migration state.

### Environment variables (12-factor)

```
DATABASE_URL          postgres://...
JWT_SECRET            <random 256-bit>
JWT_REFRESH_SECRET    <random 256-bit>
GOOGLE_CLIENT_ID      <OAuth client ID for token verification>
FIREBASE_CREDENTIALS  <base64-encoded service account JSON>
PORT                  3000
NODE_ENV              production
```

No provider-specific SDKs. All configuration through env vars. The same image runs on Fly.io (`fly.toml`), Railway (env panel), Render (`render.yaml`), or a bare VPS (`docker run --env-file .env`).

### Healthcheck

`GET /health` — returns `200 { status: "ok", db: "ok" }` after a `SELECT 1` against Postgres. Used by all PaaS platforms for readiness probes.

### Fresh deploy checklist (any platform)

1. Provision PostgreSQL (managed or self-hosted).
2. Set env vars.
3. Run `npx drizzle-kit migrate` (one-shot container or pre-deploy hook).
4. Deploy the app container.
5. Set up `pg-boss` scheduler: it auto-initializes its schema on first connect.

The only manual step that varies by platform is where env vars are entered.

---

## 10. Observability

**Recommendation: structured JSON logging (Pino) + Sentry for error tracking. Nothing else at MVP.**

- **Pino** is NestJS's fastest logger, outputs structured JSON that any log aggregator (Fly.io's built-in, Papertrail, Logtail) can ingest without configuration.
- **Sentry** free tier: captures unhandled exceptions with stack traces and request context. One `npm install @sentry/node` and one DSN env var.
- No metrics/tracing (Prometheus, Datadog, OpenTelemetry) at MVP. The user base is small enough that error reports from friends are the primary signal.
- Log these events explicitly: flush received (with Set count), reconciliation overrides (warn level), FCM dispatch failures (error level), auth failures (warn level).

When the app grows past "friends-only," the next step is a hosted log aggregator (Logtail or Axiom, both cheap) and a Grafana Cloud free-tier dashboard on Pino JSON output — no architecture change required.

---

## 11. Open Questions

These require product input before the relevant module is implemented:

1. **Routine exercise polymorphism.** `routine_exercises.exercise_id` needs to reference either `exercises` (presets) or `custom_exercises` (per-user). Options: a nullable dual FK, a single `exercises` table with a `user_id` nullable column, or a polymorphic type column. Decision affects the schema and the sync payload query. **Lean: merge into a single `exercises` table with `user_id = NULL` for presets and a `is_preset` flag.**

2. **Working Weight canonical location.** The PRD sketch has `UserExerciseConfig.workingWeightKg`. Two valid approaches: (a) store it directly and update it on each Progression Event; (b) derive it by replaying Progression Events (pure audit log). Approach (a) is simpler and fast to read. Approach (b) gives a full audit trail but requires a reduce on read. **Lean: store directly in `user_exercise_configs`, keep `progression_events` as the audit log.**

3. **Starter Routine seeding.** The PRD says a "Full Body Starter Routine" is pre-loaded on account creation. Is this seeded from a `starter_routines` table (cloned per user at sign-up) or templated in code? Does it vary by user locale/unit preference? If the exercise list or default weights ever change, should existing users get updates? This needs a decision before `account.service.ts` is written.

4. **Progression Event — user declined.** The PRD says "No → no change, no record beyond the user's choice." If the server re-evaluates and also concludes the rule was met, but the client sent `decision: rejected`, the server currently respects that decision (§5). Confirm: a user can indefinitely decline a triggered Increase Modal with no server-side consequence. If so, is there a future "overdue progression" notification type?

5. **PR definition edge case.** The UL defines PR as "heaviest Working Weight completed at the upper bound of the Rep Range." Does this mean the Set must have `reps >= repRangeUpper`, or `reps == repRangeUpper`? And does weight refer to `weightKg` (total) or `addedWeightKg` for Bodyweight Exercises? The PR query in `pr.service.ts` depends on this.

6. **FCM token rotation.** Android clients rotate FCM tokens. The current model collects all tokens per user in `fcm_tokens`. Should stale tokens be pruned (e.g., on FCM delivery failure returning `registration-token-not-registered`)? This is a reliability concern worth deciding before the FCM dispatcher is written.

7. **Reminder cadence configurability.** PRD §9 leans toward 3 days. Is this hardcoded or a per-user setting? If it becomes a setting, it needs a column in `users` and a field in `GET /me`.

