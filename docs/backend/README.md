# Liftly — Backend

**Status:** Draft for implementation
**Stack:** Node.js + NestJS + TypeScript + PostgreSQL
**Last updated:** 2026-05-20

Modular monolith with `controller → use case → service → repository` layering. Drizzle on Postgres; per-bounded-context schema folders.

## Contents

1. [Architectural style & rationale](./architecture/style.md)
2. [Module / bounded-context layout](./architecture/bounded-contexts.md)
3. [Persistence layer](./architecture/persistence.md) (full DB schema: [`schema.dbml`](./architecture/schema.dbml))
4. [Domain logic placement](./architecture/domain-logic.md)
5. [API design](./architecture/api.md)
6. [Auth](./architecture/auth.md)
7. [Background jobs](./architecture/background-jobs.md)
8. [Cache & sync semantics](./architecture/sync.md)
9. [Deployment surface](./architecture/deployment.md)
10. [Observability](./architecture/observability.md)
11. [Open questions](./open-questions.md)
12. [Architecture decisions](./decisions/README.md) (backend-local ADRs)

For cross-stack decisions, see [`docs/decisions/`](../decisions/README.md). For the docs system itself (when to read what, how to write an ADR, what to update after), see [`AGENTS.md`](../../AGENTS.md) at the repo root.
