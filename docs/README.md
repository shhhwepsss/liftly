# Liftly — Documentation

Reference docs for Liftly. Start with the product side to understand *what* Liftly is, then dive into either codebase's architecture.

## Product

- [PRD](./product/prd.md) — MVP scope, goals, and non-goals.
- [Ubiquitous Language](./product/ubiquitous-language.md) — shared vocabulary for product, code, and UI copy. Every bolded term elsewhere is defined here.
- [Abbreviations](./product/abbreviations.md) — expansions for acronyms used across docs and code (1RM, FCM, JWT, …).

## Android client

[`android/`](./android/README.md) — Kotlin + Jetpack Compose. MVI with StateFlow, Room + DataStore, WorkManager-backed offline flush.

## Backend

[`backend/`](./backend/README.md) — NestJS + TypeScript + PostgreSQL. Modular monolith with `controller → use case → service → repository` layering. Full database schema in [`backend/schema.dbml`](./backend/schema.dbml).
