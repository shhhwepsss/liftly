# Module / Bounded-Context Layout

Seven domain NestJS modules plus a `background` module map to the UL domains. Each module owns its controllers, use cases, services, repositories, and Drizzle schemas. Use cases live in a per-module `use-cases/` folder. Schemas live in a per-module `schema/` folder (see [persistence layer](./persistence.md)). Files marked **(orchestrator)** below import services from more than one module; all other use cases delegate 1:1 to their own module's service.

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
│   ├── account.repository.ts
│   └── schema/
│       └── user.schema.ts
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
│   ├── set.repository.ts
│   └── schema/
│       ├── workout.schema.ts
│       └── set.schema.ts
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
│   ├── rule-evaluator.ts                           # Pure, stateless evaluator — see domain-logic.md
│   ├── routine.repository.ts
│   ├── user-exercise-config.repository.ts
│   ├── progression-event.repository.ts
│   └── schema/
│       ├── routine.schema.ts
│       ├── routine-exercise.schema.ts
│       ├── user-exercise-config.schema.ts
│       └── progression-event.schema.ts
│
├── bodyweight/
│   ├── bodyweight.module.ts
│   ├── bodyweight.controller.ts                    # POST /weigh-ins, GET /weigh-ins
│   ├── use-cases/
│   │   ├── log-weigh-in.use-case.ts
│   │   └── list-weigh-ins.use-case.ts
│   ├── bodyweight.service.ts                       # Weigh-in persistence, latest-Weigh-in lookup, staleness check
│   ├── bodyweight.repository.ts
│   └── schema/
│       └── weigh-in.schema.ts
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
│   ├── notification.repository.ts
│   └── schema/
│       ├── notification.schema.ts
│       └── fcm-token.schema.ts
│
├── exercise/
│   ├── exercise.module.ts
│   ├── exercise.controller.ts                      # GET /exercises, POST/DELETE custom
│   ├── use-cases/
│   │   ├── list-exercises.use-case.ts
│   │   ├── create-custom-exercise.use-case.ts
│   │   └── delete-custom-exercise.use-case.ts
│   ├── exercise.service.ts
│   ├── exercise.repository.ts
│   └── schema/
│       └── custom-exercise.schema.ts               # preset exercises live in db/shared
│
├── background/
│   ├── background.module.ts
│   ├── jobs/
│   │   ├── reminder-check.job.ts                   # Calls evaluate-reminders.use-case
│   │   └── fcm-dispatch.job.ts                     # Calls notification.service.dispatch (own module)
│   └── use-cases/
│       └── evaluate-reminders.use-case.ts          # (orchestrator) workout + account + notification
│
└── db/
    ├── shared/
    │   └── exercise.schema.ts                      # preset Exercise library, owned by no module
    ├── migrations/                                 # generated SQL migrations
    └── index.ts                                    # Drizzle client setup
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
