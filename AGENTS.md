# AGENTS.md — How to work with Liftly's docs

This file is the entry point for any agent (human or LLM) navigating the documentation. Read it before opening anything else under `docs/`.

The docs system is built around one principle: **each file is loadable in isolation, and the index tells you whether to load it.** Don't pre-load everything; let the indices guide you.

---

## Map of the documentation

```
docs/
├── README.md                          # top-level index — product + stack pointers
│
├── product/                           # product context (read first to understand WHAT Liftly is)
│   ├── prd.md                         # MVP scope, goals, non-goals
│   ├── ubiquitous-language.md         # canonical vocabulary; every bolded term elsewhere is defined here
│   └── abbreviations.md               # 1RM, FCM, JWT, …
│
├── decisions/                         # cross-stack ADRs (affect backend AND android)
│   └── README.md                      # index + conventions
│
├── backend/
│   ├── README.md                      # slim link map — load this first when working on backend
│   ├── architecture/                  # current-state architecture docs (the "how it works now")
│   │   ├── style.md
│   │   ├── bounded-contexts.md
│   │   ├── persistence.md
│   │   ├── domain-logic.md
│   │   ├── api.md
│   │   ├── auth.md
│   │   ├── background-jobs.md
│   │   ├── sync.md
│   │   ├── deployment.md
│   │   ├── observability.md
│   │   └── schema.dbml                # full DB schema (DBML format)
│   ├── decisions/                     # backend-local ADRs (the "why we chose this")
│   │   ├── README.md                  # index — has a "Read when" column
│   │   └── 0001-no-bidirectional-relations.md
│   └── open-questions.md              # backlog of unresolved questions
│
└── android/                           # same shape as backend/
    ├── README.md
    ├── architecture/
    │   ├── style.md
    │   ├── modules.md
    │   ├── layering.md
    │   ├── persistence.md
    │   ├── sync.md
    │   ├── progression.md
    │   ├── auth.md
    │   ├── notifications.md
    │   ├── networking.md
    │   ├── di.md
    │   ├── ux.md
    │   ├── testing.md
    │   └── tooling.md
    ├── decisions/
    │   └── README.md
    └── open-questions.md
```

**`architecture/` vs `decisions/` — the distinction matters.**

- **`architecture/`** describes the system *as it is right now*. Read these to understand current behaviour, structure, or constraints. They are rewritten in place when the system changes.
- **`decisions/`** is a log of *why* the system is the way it is, plus the trade-offs that were considered and rejected. ADRs are append-only: when a decision is superseded, the old ADR stays and gets a `Superseded by ADR-XXXX` status — it is not deleted or rewritten.

---

## How to navigate (don't load more than you need)

1. **Always start at a README.** Top-level `docs/README.md`, or the stack README (`docs/backend/README.md`, `docs/android/README.md`). They are deliberately small — load them cheaply, then jump to specific files.
2. **For "how does X work now?" questions** → open the relevant file under `architecture/`. Follow sibling links inside it.
3. **For "why is X this way?" or "should I change X?" questions** → open `decisions/README.md` for that stack and check the "Read when" column for relevant ADRs before opening any single ADR.
4. **For terminology** → `docs/product/ubiquitous-language.md` is the single source of truth. Every bolded term elsewhere is defined there.
5. **For schema details** → `docs/backend/architecture/schema.dbml` is the authoritative table list (until MVP; see persistence.md for the source-of-truth flip).
6. **Don't bulk-load entire directories.** The system is designed for one-file-at-a-time loads driven by the indices.

---

## When to change which doc

| What changed | What to update |
|---|---|
| The system's current behaviour, structure, or layout changed | The relevant file in `architecture/`. Rewrite in place. |
| A meaningful choice was made between alternatives, especially one that may need revisiting later | Write a new ADR in the appropriate `decisions/` folder. |
| A product-level scope change (MVP includes/excludes something new) | `docs/product/prd.md`. |
| New domain term, or a term's meaning shifted | `docs/product/ubiquitous-language.md`. |
| Schema table/column/index changed | Until MVP: edit `docs/backend/architecture/schema.dbml` first (it is the spec), then chase the Drizzle TS schemas. After MVP: edit the Drizzle TS schema; CI regenerates DBML. |
| A question that doesn't yet have an answer | `docs/{backend,android}/open-questions.md`. |

**Heuristic — does this need an ADR?** If you can answer *yes* to any of these, write an ADR:
- Did you pick option A over option B (or C)?
- Could a reasonable person have made a different choice?
- Will the trade-off matter again in three months?
- Are you deferring something on purpose (e.g., "no bidirectional relations yet")?

If none of those, an architecture-doc update is enough.

---

## How to write an ADR

**Choose the right folder:**

- **Cross-stack** (decision affects backend AND android, or any future client): `docs/decisions/`.
- **Backend-local**: `docs/backend/decisions/`.
- **Android-local**: `docs/android/decisions/`.

When in doubt: if the decision is about how *one* codebase implements something, it's stack-local. If it changes a contract between codebases (API shape, sync protocol, auth flow), it's cross-stack.

**Pick the next number** by looking at the index in that folder's `README.md`. Numbering is per-folder sequential, zero-padded to four digits (`0001`, `0002`, …). The three folders have independent counters.

**Filename:** `NNNN-kebab-case-title.md`, e.g., `0002-fcm-token-rotation-policy.md`.

**Citation format** when referencing from another doc:
- Same folder: `ADR-0001`.
- Different stack folder: prefix with the folder, e.g., `backend/ADR-0001`, `android/ADR-0003`.
- Cross-stack ADR referenced from anywhere: `ADR-0001` is acceptable, or path-prefix `decisions/ADR-0001` for clarity.

**Format: MADR-lite.** Use this template:

```markdown
# ADR-NNNN: <short imperative title>

- **Status:** Proposed | Accepted | Superseded by ADR-XXXX | Deprecated
- **Date:** YYYY-MM-DD

## Context and Problem Statement

What forced this decision? State the problem in 1-3 paragraphs. Include constraints (deadlines, scale, team size) that drive the answer.

## Considered Options

1. **<Option A>** — one-paragraph description.
2. **<Option B>** — one-paragraph description.
3. **<Option C>** — one-paragraph description.

## Decision

**Option N: <name>.**

One paragraph explaining why this option wins given the context.

## Consequences

**Positive**
- …

**Negative**
- …

## Revisit Trigger

Concrete condition(s) under which this ADR should be reopened. Avoid vague triggers like "when it becomes a problem"; write specific signals (e.g., "when ≥2 use cases need X and the workaround is materially worse").

## Pros and Cons of the Options

### Option A: <name>
- **Good:** …
- **Bad:** …

### Option B: <name>
- **Good:** …
- **Bad:** …
```

If an ADR genuinely needs Decision Drivers or Confirmation sections (full MADR), add them — but only when they earn their place.

---

## What to update *after* writing an ADR

An ADR isn't done when the file is written. To keep the docs system coherent:

1. **Update the index.** Add a row to the `README.md` table in the same `decisions/` folder, with the `Read when` column filled in. Without this row, the ADR is unreachable.
2. **Update the architecture docs that now reflect this decision.** If the ADR changes how schemas are organized, update `backend/architecture/persistence.md`. If it changes module rules, update `bounded-contexts.md` / `modules.md`. The ADR explains *why*; the architecture doc explains *what it looks like now*.
3. **Link from the architecture doc to the ADR.** Inline reference at the point the decision is felt: *"See [`backend/ADR-0001`](../decisions/0001-…) for the rationale."*
4. **If this ADR supersedes a previous one:** edit the old ADR's status to `Superseded by ADR-XXXX` and add a note at the top. Update the old ADR's row in the index. Do not delete the old ADR.
5. **Clear or move anything in `open-questions.md` that this ADR resolves.**
6. **If the ADR affects more than one stack:** also link from the *other* stack's README or relevant architecture doc, so a reader landing in either stack discovers it.
7. **Bump the `Last updated:` date** on any architecture file you edited.

---

## What to update when an architecture doc changes (without a new ADR)

1. The architecture doc itself.
2. Any other architecture doc that references the changed behaviour.
3. The `Last updated:` date in that doc's header.
4. If the change touches a term, also update `docs/product/ubiquitous-language.md`.
5. If the change resolves an open question, remove it from `open-questions.md`.

---

## Anti-patterns (don't do these)

- **Don't duplicate content across `architecture/` and `decisions/`.** Architecture says *what is*; decisions say *why*. They link to each other; they don't repeat each other.
- **Don't expand the README.** It's intentionally slim so an agent loads it cheaply. New content goes into a file under `architecture/` or `decisions/` and gets a link from the README.
- **Don't rewrite an ADR.** ADRs are immutable once Accepted. To change a decision, write a new ADR that supersedes the old one.
- **Don't add an "ADR-like" section to an architecture doc.** If something is a decision worth recording, it belongs in `decisions/`, not buried inside `architecture/`.
- **Don't skip the Revisit Trigger.** Vague triggers ("when it gets bad") never fire. Concrete ones do.
