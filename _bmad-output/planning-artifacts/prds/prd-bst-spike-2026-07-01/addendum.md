# Addendum — Downstream depth (architecture / stories)

Depth the brief volunteered that belongs downstream, not in the PRD body. Preserved here so it isn't lost.

## Concurrency mechanism (architecture decision)

The PRD fixes only the observable contract (FR4: exactly one of two concurrent clashing writes wins). The *how* is an architecture call, to be recorded in the story/PRD diff at implementation time per the demo runsheet:

- **Option A — DB exclusion constraint.** Enforce non-overlap declaratively at the database (e.g. Postgres `EXCLUDE USING gist` over person + daterange). The clash becomes impossible to persist; the losing write fails on constraint violation, which the service maps to the FR5 rejection. Strongest correctness guarantee, no application race window.
- **Option B — Optimistic locking.** Version/serialize on the person's assignment set; the second writer's commit fails the version check and retries/rejects. More application logic; correctness depends on getting the check right.

The brainstorm's non-negotiable: **not** a naive read-then-check, which has an interleave window and is exactly the bug the P0 test is designed to catch. Mechanism choice is deferred to `bmad-architecture` / the first story, but Option A is the lower-risk default given the constraint is expressible at the data layer.

## Losing-planner UX (story detail)

FR5 says the loser gets a clear, deterministic rejection identifying the clash. Concrete shape (HTTP 409 vs a domain error, message copy, whether a UI renders it) is a story-level decision tied to Open Question #2 and the surface decision (Open Question #3).

## Stack note

React / Node are *indicative* per the constitution, not confirmed; KPMG may push for Python and/or the full Azure DevOps suite over GitHub. The scaffold is pulled into existence by the first story (PRD-first, not scaffold-first), so the stack choice stays deferrable until a story demands it.
