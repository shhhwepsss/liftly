# Backend Architecture Decisions

Backend-local ADRs. Decisions that affect the Android client (or any future iOS/web client) belong in [`docs/decisions/`](../../decisions/README.md) instead.

For the docs system itself (how to write an ADR, what to update after), see [`AGENTS.md`](../../../AGENTS.md) at the repo root.

## Conventions

- **Format:** MADR-lite — Title, Status, Date, Context, Considered Options, Decision, Consequences, Revisit Trigger, Pros and Cons.
- **Numbering:** per-folder sequential (`0001`, `0002`, …).
- **Citation:** `backend/ADR-0001` when referenced from outside this folder; `ADR-0001` when referenced from inside.
- **Statuses:** `Proposed` | `Accepted` | `Superseded by ADR-XXXX` | `Deprecated`.

## Index

| #    | Title                                                                            | Status   | Read when                                                                  |
|------|----------------------------------------------------------------------------------|----------|----------------------------------------------------------------------------|
| 0001 | [No bidirectional Drizzle relations](./0001-no-bidirectional-relations.md)       | Accepted | Writing Drizzle schemas; considering `.with()` traversal in a query.       |
