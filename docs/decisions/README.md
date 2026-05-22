# Cross-stack Architecture Decisions

ADRs that affect more than one stack (backend + Android client, or any future iOS/web client) live here. Stack-local decisions go in `docs/backend/decisions/` or `docs/android/decisions/`.

For the docs system itself (how to write an ADR, what to update after), see [`AGENTS.md`](../../AGENTS.md) at the repo root.

## Conventions

- **Format:** MADR-lite — Title, Status, Date, Context, Considered Options, Decision, Consequences, Revisit Trigger, Pros and Cons.
- **Numbering:** per-folder sequential (`0001`, `0002`, …); restart at `0001` in each of the three decisions folders.
- **Citation across stacks:** prefix with the stack folder when referencing from another stack, e.g., `backend/ADR-0001`, `android/ADR-0003`, or just `ADR-0001` when referencing a cross-stack ADR from within `docs/decisions/`.
- **Statuses:** `Proposed` | `Accepted` | `Superseded by ADR-XXXX` | `Deprecated`.

## Index

| # | Title | Status | Read when |
|---|-------|--------|-----------|
| 0001 | [Refresh token in the response body, not an HttpOnly cookie](./0001-bearer-tokens-not-httponly-cookies.md) | Accepted | Touching the auth token flow; answering why the refresh token isn't an HttpOnly cookie; adding any non-Android client. |

## Stack-local decision logs

- [`backend/decisions/`](../backend/decisions/README.md) — Backend ADRs (Drizzle, NestJS layering, etc.).
- [`android/decisions/`](../android/decisions/README.md) — Android ADRs (MVI, Room, etc.).
