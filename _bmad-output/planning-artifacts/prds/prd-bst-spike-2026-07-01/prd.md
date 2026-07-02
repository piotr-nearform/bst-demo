---
title: "PRD — Double-Booking Guard (BST Scheduling Spike)"
status: final
created: 2026-07-01
updated: 2026-07-01
---

# PRD — Double-Booking Guard (BST Scheduling Spike)

## 1. Context

This is a **throwaway spike** to rehearse the ways of working Nearform and KPMG will agree on for the real Budget Submission Tool, ahead of Discovery. The app is disposable; the pipeline is the deliverable. The load-bearing thing the spike proves — and the direct answer to KPMG's hottest anticipated pushback, *limited trust in AI building the app* — is that **AI-authored code is safe because a named human owns it**, evidenced by a small, test-first, reviewable PR with a spec that lives in version control. A named human owning the merge is **more** accountability than today, **not less**: AI is the tooling that makes that ownership reviewable, not a replacement for the decision.

The one feature carried through the pipeline is the **double-booking guard**: a planner assigns a person to a job over a date range, and the system refuses assignments that clash. The feature was chosen because its one hard edge — two planners booking the same person at the same instant — forces a real product↔engineering spec conversation and showcases *how we work* rather than trivial CRUD.

Hard constraints (Azure-only / UK residency, ATDD test-first, spec-as-contract, agents-draft-humans-approve) are defined in the project constitution `_bmad-output/project-context.md` and are **not restated here**; they govern this PRD.

## 2. Goal & Success Criteria

**Goal:** ship one **single human-approved pull request** that demonstrates AI-authored-but-human-owned delivery end to end.

The spike succeeds when that PR contains all of:

- A compact PR description restating the acceptance criteria **verbatim**, with a **table** mapping **each AC → the test that covers it**.
- A green **P0 Playwright acceptance test** that fires two concurrent, clashing assignments and asserts **exactly one wins**.
- The **story + this PRD's spec decision in version control**, with author and timestamp visible in the diff.
- A **multi-model automated review** trail plus a **named human approval** on the merge.

## 3. Scope

**In scope:**

- A minimal planner-facing **UI** to create an assignment (person → job over a date range) and see the outcome — success or a clash rejection.
- Rejecting an assignment that clashes with an existing one for the same person, where **clash = inclusive date-range overlap** (see FR3).
- The concurrency case: two planners submitting clashing assignments for the same person mid-flight, resolved atomically at the data layer so exactly one persists.

**Explicitly out of scope:** skills-matching, availability calendars, authentication/roles, and **UI polish** (styling and layout beyond what is needed to show the guard working). Person and job records are assumed pre-seeded; managing them is not part of the spike.

**Target:** whole spike wrapped in 1–2 days.

## 4. User Journey

**UJ-1 — The race.** Priya and Marcus are both planners staffing an audit. Auditor Sam is already free for the first week of August. At nearly the same moment, Priya books Sam onto the *Acme* job for 3–7 Aug and Marcus books Sam onto the *Beta* job for the same days. Both submit. One booking is accepted and persisted; the other planner sees a clear on-screen message that Sam already has a clashing assignment for those dates — not a crash, a generic error, or a silent success that later vanishes. Sam is never double-booked.

## 5. Functional Requirements

**Feature: Double-Booking Guard**

- **FR1 — Create assignment (via UI).** A planner can create an assignment through a simple UI, binding a *person* to a *job* over an inclusive date range (start date, end date, day granularity). Minimal fields only: person, job, start, end.
- **FR2 — Non-clashing assignment succeeds.** An assignment for a person whose date range does not overlap any existing assignment for that same person is accepted and persisted.
- **FR3 — Clash is rejected.** An assignment is refused if its date range overlaps an existing assignment for the **same person**; no double-booking is ever persisted. **Clash = inclusive overlap:** two assignments clash if they share *any* day, endpoints included — a person cannot be shared even on their last day (so 3–7 Aug clashes with 7–10 Aug). This single rule covers touching-boundary and partial-overlap cases alike. *(PO decision, 2026-07-01 — see §9.)*
- **FR4 — Concurrency is resolved atomically (P0).** When two clashing assignments for the same person are submitted concurrently, exactly one is persisted and the other is rejected. Enforcement is atomic at the data layer — never a read-then-check that can interleave. The specific mechanism is an architecture decision (see `addendum.md`); the PRD fixes only the observable contract: *exactly one wins*.
- **FR5 — The loser gets a clear, deterministic rejection, at two layers.** The rejected attempt must be deterministic and distinguishable from a system error — never a 500, a timeout, or a silent no-op. **API layer:** returns **HTTP 409 Conflict** identifying the clash (person, dates). **UI layer:** shows the planner a plain-language message, e.g. *"Sorry, Sam's already booked for those dates"* (exact copy finalized in the build). *(PO decision, 2026-07-01 — see §9.)*

## 6. Acceptance Criteria

The AC below are the spec-as-contract source the ATDD Playwright tests are authored from, and are restated in the PR description mapped to their covering tests. Tests key on a **stable UI hook** (e.g. a test id / ARIA role), not the final message copy — the wording is not yet fixed, but the presence of a deterministic clash-rejection is.

- **AC1 (P0, concurrency — via UI):** Given Sam has no assignment for 3–7 Aug, when two planners in two separate browser sessions submit clashing bookings for Sam over 3–7 Aug at the same moment, then exactly one booking is stored, the winning planner sees success, and the losing planner sees the clash-rejection message. *(covers FR3, FR4, FR5; Playwright drives two browser contexts)*
- **AC2 (happy path — via UI):** Given Sam has no clashing assignment, when a planner books Sam for a free date range through the UI, then the booking is stored and success is shown. *(covers FR1, FR2)*
- **AC3 (sequential clash, incl. boundary — via UI):** Given Sam is already booked 3–7 Aug, when a planner submits a booking that overlaps — including one that only shares the final day (7–10 Aug) — then it is rejected with the clash-rejection message and nothing is stored. *(covers FR3, FR5)*
- **AC4 (API contract — 409):** Given Sam is already booked 3–7 Aug, when a clashing assignment for Sam is submitted **directly to the API**, then the API responds **HTTP 409 Conflict** and nothing is persisted. *(covers FR5 API layer, FR3; a service/API-level test complementing the UI tests, and the deterministic pin for the race noted in `addendum.md`)*

## 7. Cross-cutting Requirements

- **Concurrency correctness is the primary quality attribute** and the P0 risk; it is enforced at the data layer and proven by AC1. This is the one place depth is warranted — everything else is deliberately thin.
- **Spec-as-contract.** If behaviour changes, this PRD and the linked story change in the same PR; drift is treated as a failure. The Azure Boards story links back via `AB#<id>`, with PRD-in-version-control as the canonical source.
- **Slices stay small.** The whole feature lands in one PR a single reviewer can read and own; the test evidence is written to read well. Small slices pay three times — **accountability** (a human can genuinely own the PR), **velocity** (review stays cheap, so it doesn't eat the AI speed gains), and **reviewability**. *Caveat: AI tools that generate large changes push against this — keep slices small deliberately.* This is a hard constraint on *how*, not a nice-to-have — reviewability is the sales pitch.
- **Review focuses where risk is.** The reviewer reads the tests closely, skims the implementation, and watches sensitive files (dependencies, `package.json`, infra). The review gate is detailed in `../roles-and-handoffs.md`.
- Remaining non-functionals (hosting, residency, secrets, review, approval) are governed by the constitution and not duplicated here.

## 8. Success Metrics

Because the deliverable is a way of working, the metrics are process metrics of the one PR:

- **AC1 P0 test is green** and asserts exactly-one-wins (not weakened to pass). *Counter-metric: a test relaxed or an AC dropped to force green.*
- **PR is small enough to review in one sitting.** *Counter-metric: the slice bloats beyond a single readable PR.*
- **Spec traceability holds:** every AC maps to a test in the PR description, and the PRD/story decision is in the diff with author + timestamp. *Counter-metric: code merged that has drifted from the PRD.*
- **A named human approves the merge** after the multi-model review pass. *Counter-metric: any artifact marked "done" by an agent.*
- **The P0 test was red first** — written from the AC and failing before any implementation existed (ATDD). *Counter-metric: a test written or relaxed after the code to fit it.*
- **The story is linked both ways** via `AB#<id>` (GitHub ↔ Azure Boards), so the human's board view and the canonical PRD-in-VC stay in sync. *Counter-metric: a merged change with no traceable story link.*

## 9. Resolved Decisions (the PO spec negotiation)

The concurrent-clash edge forced three product↔engineering decisions. Resolved with the PO on 2026-07-01:

1. **Clash definition (FR3).** Clash = **inclusive date-range overlap** — a person cannot be shared on any day, including the first and last. Touching-boundary and partial-overlap are therefore both clashes, collapsing to one simple rule: *any shared day is a clash.* (Supersedes the brainstorm's deferral of the boundary/partial edges — the decision simplifies the rule rather than adding edge-handling.)
2. **Losing-planner contract (FR5).** Two layers: the API returns **HTTP 409**; the UI shows a plain-language message ("Sorry, Sam's already booked for those dates"), with copy finalized in the build.
3. **Demo surface (FR1/FR4).** The tool **has a UI**, and the acceptance tests **drive the UI** (Playwright, two browser sessions for the concurrency case). The atomic guarantee still lives at the data layer; the UI test proves it end to end.

## 10. Glossary

- **Assignment** — a booking of one *person* onto one *job* over an inclusive date range (start–end, day granularity).
- **Clash** — two assignments for the **same person** whose inclusive date ranges share *any* day, endpoints included (see FR3).
- **Planner** — the operator role who creates assignments (the single human role in scope).
- **P0** — the highest business-risk test tier; it must pass and must never be weakened to go green.
