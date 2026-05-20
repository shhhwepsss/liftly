# Liftly — Documentation

Reference docs for Liftly. Start with the product side to understand *what* Liftly is, then dive into either codebase's architecture.

## Product

- [PRD](./product/prd.md) — MVP scope, goals, and non-goals.
- [Ubiquitous Language](./product/ubiquitous-language.md) — shared vocabulary for product, code, and UI copy. Every bolded term elsewhere is defined here.
- [Abbreviations](./product/abbreviations.md) — expansions for acronyms used across docs and code (1RM, FCM, JWT, …).

## Android client

[`android/`](./android/README.md) — Kotlin + Jetpack Compose. MVI with StateFlow, Room + DataStore, WorkManager-backed offline flush.

## Backend

[`backend/`](./backend/README.md) — NestJS + TypeScript + PostgreSQL. Modular monolith with `controller → use case → service → repository` layering. Full database schema in [`backend/architecture/schema.dbml`](./backend/architecture/schema.dbml).

## Architecture decisions

- [`decisions/`](./decisions/README.md) — Cross-stack ADRs. Each stack also has its own decision log: [`backend/decisions/`](./backend/decisions/README.md), [`android/decisions/`](./android/decisions/README.md).
- See [`AGENTS.md`](../AGENTS.md) at the repo root for how to navigate the docs, when to write an ADR, and what to update after writing one.
