# Architectural Style & Rationale

Liftly uses a **modular monolith with domain-aligned NestJS modules**, each structured in a four-layer pattern: **controller → use case → service → repository**. This is not hexagonal architecture — the overhead of ports/adapters is unjustified for a one-person MVP — but modules are internally cohesive enough that extracting a service later (if Streak math or Progression logic grows complex) requires moving files, not redesigning boundaries.

The use-case layer is mandatory and load-bearing:

- **Controllers** are thin HTTP adapters. They parse the request, call exactly one use case, and shape the response. They do not inject services.
- **Use cases** are medium-sized orchestrators. One use case == one user-facing operation. They are the **only** layer permitted to depend on more than one service, and they are where cross-module coordination lives (e.g., flushing a workout writes Sets, re-evaluates Progression rules, and creates Notifications — all behind one use case).
- **Services** own a single module's domain logic and depend only on their own module's repositories and pure domain helpers. **Services never import other services**, in their own module or any other. Any cross-service work — including same-module cross-service work — goes through a use case.
- **Repositories** wrap Drizzle calls for one aggregate.

Even trivial CRUD endpoints route through a use case. The cost is one delegating file per endpoint; the benefit is uniformity — there's one place to add transactions, audit logging, or cross-cutting orchestration later without rewriting controllers.

The driving constraint is **fat backend**: Rep Range evaluation, Progression Event creation, PR calculation, and Streak math must live server-side so future iOS/web clients get them for free. DDD-lite is applied at the module boundary level — the `progression` module owns its domain rules and does not leak evaluation logic into `workout` or `notification`. Modules communicate **through use cases that compose services across modules** (never through one service importing another), which prevents the most common monolith trap of implicit coupling.
