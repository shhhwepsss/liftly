# ADR-0001: No bidirectional Drizzle relations

- **Status:** Accepted
- **Date:** 2026-05-20

## Context and Problem Statement

Schemas are split per bounded context (`src/<module>/schema/*.schema.ts`), with shared/lookup tables under `src/db/shared/`. This introduces cross-module foreign keys — `sets.user_id → users.id`, `workouts.user_id → users.id`, `progression_events.workout_id → workouts.id`, etc.

Drizzle's `relations()` helper supports declaring both the child→parent side (`set.user`) and the parent→child side (`user.workouts`) for type-safe nested queries like `db.query.users.findMany({ with: { workouts: true } })`. Declaring both sides means `user.schema.ts` has to import `workouts` (and every other child table that points at users). That creates a TypeScript import cycle between bounded contexts at the schema layer — the same trap we work hard to avoid at the service layer.

## Considered Options

1. **Declare only the child-side relation.** `set.user`, `workout.user`, etc. Parent → child traversal via `.with()` is not available; joins are written explicitly when needed.
2. **Park all `relations()` calls in `src/db/relations.ts`.** A single shared file imports from both sides and declares the full graph. Tables themselves stay acyclic.
3. **Forgo `relations()` entirely.** Use the query builder (`.leftJoin(...)`) for every relational read.

## Decision

**Option 1: child-side relations only.**

We don't currently have a use case that needs parent→child relational traversal. We will not pre-emptively introduce the `src/db/relations.ts` barrel or accept the cross-module import cost.

## Consequences

**Positive**
- No cross-module schema imports beyond FK references.
- Smallest current code surface — no extra file, no extra mental model.
- Aligns with the bounded-context principle that a parent module shouldn't know which children point at it.

**Negative**
- `db.query.users.findMany({ with: { workouts: true } })` and similar parent→child traversals are not available. Code that needs the parent-then-children shape writes an explicit `.leftJoin(...)` or two queries.
- If multiple use cases later need parent→child relational traversal, we revisit (see Revisit Trigger).

## Revisit Trigger

Revisit when **both** of these hold:

1. At least two use cases need `db.query.X.findMany({ with: { Y: true } })`-style parent→child traversal.
2. The equivalent manual join is materially worse than the relational helper would be (verbose, error-prone, or significantly slower).

At that point, migrate to Option 2: introduce `src/db/relations.ts`, declare both sides of the relevant relations there, and keep the table files themselves free of reverse imports.

## Pros and Cons of the Options

### Option 1: Child-side only
- **Good:** no cycles, no new files, smallest current surface, easiest to reason about.
- **Bad:** no parent→child `.with()` traversal; must write joins explicitly.

### Option 2: Shared `src/db/relations.ts` barrel
- **Good:** full bidirectional traversal; tables stay acyclic; one place owns the graph.
- **Bad:** every schema change touches two files; the barrel has to import from every module, which is a coupling the bounded-context rules otherwise forbid.

### Option 3: No `relations()`, query builder only
- **Good:** simplest mental model; zero relational magic; no cycle risk by construction.
- **Bad:** every relational read is verbose at the call site; no nested result shaping; fights Drizzle's grain.
