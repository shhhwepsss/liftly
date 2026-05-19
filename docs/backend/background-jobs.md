# Background Jobs

**Recommendation: `pg-boss` (PostgreSQL-backed job queue). No Redis.**

`pg-boss` runs entirely within the existing PostgreSQL instance: it creates its own schema and uses `SKIP LOCKED` for reliable at-least-once delivery. For Liftly's scale (a handful of users, a few reminder jobs per day) this is sufficient.

**Jobs:**
- **`reminder.check`** — runs daily via `pg-boss` scheduled job. The job is a thin scheduler hook that calls `background/use-cases/evaluate-reminders.use-case.ts`, which queries recent workouts (`workout.service`) and user reminder preferences (`account.service`), then enqueues an FCM dispatch job via `notification.service`.
- **`fcm.dispatch`** — worker that sends a single FCM push via Firebase Admin SDK. Calls `notification.service` within its own module (no use case needed — single-service work). Decoupled from the request thread so a slow FCM call never delays the flush response.
- **PR / Streak evaluation** — triggered inline by `workout/use-cases/complete-workout.use-case.ts` (not via job queue) because they are fast synchronous DB reads. The use case calls `pr.service` and `streak.service` directly.

**Why not BullMQ + Redis:** Redis is real operational overhead — another service to deploy, monitor, and back up. At this scale, `pg-boss` eliminates the dependency entirely. If job throughput ever demands Redis (it won't for a friends-only app), the migration is a queue-client swap.

**Why not `@Cron` (NestJS/node-cron):** `@Cron` runs in-process and is not durable — a deploy during the cron window loses the job silently. `pg-boss` survives restarts.
