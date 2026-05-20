# Liftly — Android

**Status:** Draft for implementation
**Stack:** Kotlin + Jetpack Compose + Android SDK
**Last updated:** 2026-05-20

Single-module Compose app with MVI + StateFlow. Room + DataStore for local storage; WorkManager-backed offline flush.

## Contents

1. [Architectural style & rationale](./architecture/style.md)
2. [Module layout](./architecture/modules.md)
3. [Layering](./architecture/layering.md)
4. [Local persistence](./architecture/persistence.md)
5. [Sync engine (incl. rest timer)](./architecture/sync.md)
6. [Trigger rule evaluator placement](./architecture/progression.md)
7. [Auth & session](./architecture/auth.md)
8. [Notification handling](./architecture/notifications.md)
9. [Networking layer](./architecture/networking.md)
10. [Dependency injection](./architecture/di.md)
11. [Theming, formatting, i18n & navigation](./architecture/ux.md)
12. [Testing strategy](./architecture/testing.md)
13. [Project tooling](./architecture/tooling.md)
14. [Open questions & cross-doc follow-ups](./open-questions.md)
15. [Architecture decisions](./decisions/README.md) (android-local ADRs)

For cross-stack decisions, see [`docs/decisions/`](../decisions/README.md). For the docs system itself (when to read what, how to write an ADR, what to update after), see [`AGENTS.md`](../../AGENTS.md) at the repo root.
