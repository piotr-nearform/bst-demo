# Addendum — Downstream depth (architecture / stories)

Depth the brief volunteered that belongs downstream, not in the PRD body. Preserved here so it isn't lost.

## Concurrency mechanism (architecture decision)

The PRD fixes only the observable contract (FR4: exactly one of two concurrent clashing writes wins). The *how* is an architecture call, to be recorded in the story/PRD diff at implementation time per the demo runsheet:

- **Option A — DB exclusion constraint.** Enforce non-overlap declaratively at the database (e.g. Postgres `EXCLUDE USING gist` over person + daterange). The clash becomes impossible to persist; the losing write fails on constraint violation, which the service maps to the FR5 rejection. Strongest correctness guarantee, no application race window.
- **Option B — Optimistic locking.** Version/serialize on the person's assignment set; the second writer's commit fails the version check and retries/rejects. More application logic; correctness depends on getting the check right.

The brainstorm's non-negotiable: **not** a naive read-then-check, which has an interleave window and is exactly the bug the P0 test is designed to catch. Mechanism choice is deferred to `bmad-architecture` / the first story, but Option A is the lower-risk default given the constraint is expressible at the data layer.

## Losing-planner UX (resolved with PO, 2026-07-01)

FR5 is now decided at two layers: the API returns **HTTP 409** identifying the clash; the UI renders a plain-language message ("Sorry, Sam's already booked for those dates"). Exact copy is finalized in the build. Story-level detail: how the message surfaces (inline on the form vs a toast/banner) and whether the form clears or preserves the attempted dates.

## Testing concurrency through the UI (test-design note)

The PO confirmed acceptance tests drive the **UI**, including the P0 concurrency case. Playwright approach: open **two browser contexts**, fill both booking forms, and fire the two submits as close to simultaneously as possible (e.g. `Promise.all` on the two actions) to exercise the race. Assert exactly one booking persists, the winner sees success, the loser sees the "already booked" message.

Note the guarantee still lives at the **data layer** (FR4) — the UI test proves the behaviour end to end but two near-simultaneous clicks are not perfectly simultaneous; the atomic constraint/lock is what actually makes "exactly one wins" true. A thin service/DB-level check may complement the UI test to pin the race deterministically.

## Stack note

React / Node are *indicative* per the constitution, not confirmed; KPMG may push for Python and/or the full Azure DevOps suite over GitHub. With a UI now in scope, the first story's scaffold will include a front end as well as a Node service + database — but the scaffold is still pulled into existence by the story (PRD-first, not scaffold-first), so the stack choice stays deferrable until a story demands it.
