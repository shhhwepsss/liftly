# Persistence Layer

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

## Migration workflow

```
1. Edit src/db/schema.ts (the canonical schema)
2. npx drizzle-kit generate         → writes a numbered SQL migration to drizzle/
3. Review the generated SQL, commit alongside the schema change
4. npx drizzle-kit migrate          → applies pending migrations to the target DB
```

Generated migrations are plain SQL files — reviewed, version-controlled, and applied identically in CI, staging, and prod. There is no shadow database.

## Table list

The full schema — tables, columns, enums, indexes, and foreign keys — lives in [`schema.dbml`](./schema.dbml) (DBML format, renderable at [dbdiagram.io](https://dbdiagram.io)). Edit that file when adding or modifying tables; this section only summarizes the table set and calls out non-obvious indexes (see below).

**Tables** (see `schema.dbml` for full definitions):

- `users` — identity, settings, per-user defaults for Rep Range / increment / Trigger Rule.
- `exercises` — preset Exercise library (`is_preset = true`).
- `custom_exercises` — per-user custom Exercises. See [Q1 in open questions](./open-questions.md) on possible merge into `exercises`.
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
