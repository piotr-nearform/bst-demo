---
title: "PRD — Double-Booking Guard (BST Scheduling Spike)"
status: draft
created: 2026-07-01
updated: 2026-07-01
---

# PRD — Double-Booking Guard (BST Scheduling Spike)

## 1. Context

This is a **throwaway spike** to rehearse the ways of working Nearform and KPMG will agree on for the real Budget Submission Tool, ahead of Discovery. The app is disposable; the pipeline is the deliverable. The load-bearing thing the spike proves: **AI-authored code is safe because a named human owns it**, evidenced by a small, test-first, reviewable PR with a spec that lives in version control.

The one feature carried through the pipeline is the **double-booking guard**: a planner assigns a person to a job over a date range, and the system refuses assignments that clash. The feature was chosen because its one hard edge — two planners booking the same person at the same instant — forces a real product↔engineering spec conversation and showcases *how we work* rather than trivial CRUD.

Hard constraints (Azure-only / UK residency, ATDD test-first, spec-as-contract, agents-draft-humans-approve) are defined in the project constitution `_bmad-output/project-context.md` and are **not restated here**; they govern this PRD.

## 2. Goal & Success Criteria

**Goal:** ship one **single human-approved pull request** that demonstrates AI-authored-but-human-owned delivery end to end.

The spike succeeds when that PR contains all of:

- A compact PR description restating the acceptance criteria and mapping **each AC → the test that covers it**.
- A green **P0 Playwright acceptance test** that fires two concurrent, clashing assignments and asserts **exactly one wins**.
- The **story + this PRD's spec decision in version control**, with author and timestamp visible in the diff.
- A **multi-model automated review** trail plus a **named human approval** on the merge.

## 3. Scope

**In scope — the concurrent-clash edge only:**

- Creating an assignment (person → job over a date range).
- Rejecting an assignment that clashes with an existing one for the same person.
- The concurrency case: two planners submitting clashing assignments for the same person mid-flight, resolved atomically at the data layer so exactly one persists.

**Explicitly out of scope:** partial-overlap edges, touching-boundary edges (e.g. does an end date meeting a start date clash), skills-matching, availability calendars, authentication/roles, and UI polish. Person and job records are assumed pre-seeded; managing them is not part of the spike.

**Target:** whole spike wrapped in 1–2 days.

## 4. User Journey

**UJ-1 — The race.** Priya and Marcus are both planners staffing an audit. Auditor Sam is already free for the first week of August. At nearly the same moment, Priya books Sam onto the *Acme* job for 3–7 Aug and Marcus books Sam onto the *Beta* job for the same days. Both submit. One booking is accepted and persisted; the other planner is told, clearly, that Sam already has a clashing assignment for those dates — not shown a crash, a generic error, or a silent success that later vanishes. Sam is never double-booked.

## 5. Functional Requirements

**Feature: Double-Booking Guard**

- **FR1 — Create assignment.** A planner can create an assignment binding a *person* to a *job* over a date range (start date, end date, day granularity). `[ASSUMPTION]` minimal fields only: person, job, start, end.
- **FR2 — Non-clashing assignment succeeds.** An assignment for a person whose date range does not overlap any existing assignment for that same person is accepted and persisted.
- **FR3 — Clash is rejected.** An assignment whose date range overlaps an existing assignment for the **same person** is refused; no double-booking is ever persisted. `[ASSUMPTION]` "clash" = interior overlap of the two ranges; touching-boundary and partial-overlap semantics are out of scope, so the guard treats clearly-overlapping ranges as clashes and defers the fiddly boundary cases (see Open Questions — this is the core PO spec decision).
- **FR4 — Concurrency is resolved atomically (P0).** When two clashing assignments for the same person are submitted concurrently, exactly one is persisted and the other is rejected. Enforcement is atomic at the data layer — never a read-then-check that can interleave. The specific mechanism is an architecture decision (see `addendum.md`); the PRD fixes only the observable contract: *exactly one wins*.
- **FR5 — The loser gets a clear, deterministic rejection.** The rejected planner receives an explicit rejection that identifies the clash (which person, which dates). It must be deterministic and distinguishable from a system error — not a 500, a timeout, or a silent no-op. `[ASSUMPTION]` surfaced as a definite "already assigned" response (e.g. HTTP 409 with a message); exact response shape and copy are the PO decision in Open Questions.

## 6. Acceptance Criteria

The AC below are the spec-as-contract source the ATDD Playwright tests are authored from, and are restated in the PR description mapped to their covering tests.

- **AC1 (P0, concurrency):** Given Sam has no assignment for 3–7 Aug, when two clashing assignments for Sam over 3–7 Aug are submitted concurrently, then exactly one is persisted and the other is rejected — verified by asserting one success + one rejection and a single stored assignment. *(covers FR3, FR4, FR5)*
- **AC2 (happy path):** Given Sam has no clashing assignment, when a planner assigns Sam to a job for a free date range, then the assignment is persisted. *(covers FR1, FR2)*
- **AC3 (sequential clash):** Given Sam is already assigned for 3–7 Aug, when a planner submits a clashing assignment for Sam, then it is rejected with the clear "already assigned" response and nothing is persisted. *(covers FR3, FR5)*

## 7. Cross-cutting Requirements

- **Concurrency correctness is the primary quality attribute** and the P0 risk; it is enforced at the data layer and proven by AC1. This is the one place depth is warranted — everything else is deliberately thin.
- **Spec-as-contract.** If behaviour changes, this PRD and the linked story change in the same PR; drift is treated as a failure. The Azure Boards story links back via `AB#<id>`, with PRD-in-version-control as the canonical source.
- **Slices stay small.** The whole feature lands in one PR a single reviewer can read and own; the test evidence is written to read well. This is a hard constraint on *how*, not a nice-to-have — reviewability is the sales pitch.
- Remaining non-functionals (hosting, residency, secrets, review, approval) are governed by the constitution and not duplicated here.

## 8. Success Metrics

Because the deliverable is a way of working, the metrics are process metrics of the one PR:

- **AC1 P0 test is green** and asserts exactly-one-wins (not weakened to pass). *Counter-metric: a test relaxed or an AC dropped to force green.*
- **PR is small enough to review in one sitting.** *Counter-metric: the slice bloats beyond a single readable PR.*
- **Spec traceability holds:** every AC maps to a test in the PR description, and the PRD/story decision is in the diff with author + timestamp. *Counter-metric: code merged that has drifted from the PRD.*
- **A named human approves the merge** after the multi-model review pass. *Counter-metric: any artifact marked "done" by an agent.*

## 9. Open Questions (the PO spec negotiation)

These are the decisions the concurrent-clash edge forces the PO and engineer to make together — resolving them is part of the demo, not a blocker to drafting:

1. **Clash definition (FR3).** Confirm in-scope = interior overlap only, with touching-boundary and partial-overlap deferred. This is the core spec decision the feature was chosen to surface.
2. **Losing-planner contract (FR5).** Confirm the rejection shape and copy — e.g. HTTP 409 + a message naming the clashing dates. What exactly does the loser see?
3. **Surface for the demo (FR1/FR4).** Does the P0 test drive a minimal UI, or exercise the assignment endpoint/service directly? `[ASSUMPTION]` the cleanest concurrency proof is at the API/service layer and no UI is required for the money-shot; confirm before scaffolding, since it shapes the first story.
