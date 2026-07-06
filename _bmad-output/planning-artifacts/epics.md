---
stepsCompleted: ['step-01-validate-prerequisites', 'step-02-design-epics', 'step-03-create-stories', 'step-04-final-validation']
inputDocuments:
  - '_bmad-output/planning-artifacts/prds/prd-bst-spike-2026-07-01/prd.md'
  - '_bmad-output/planning-artifacts/prds/prd-bst-spike-2026-07-01/addendum.md'
  - '_bmad-output/planning-artifacts/architecture/architecture-bst-spike-2026-07-02/ARCHITECTURE-SPINE.md'
  - '_bmad-output/project-context.md'
---

# BST spike (Double-Booking Guard) - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the BST spike (Double-Booking Guard), decomposing the requirements from the PRD, PRD addendum, and Architecture spine into implementable stories. This is a throwaway spike to rehearse the BST ways of working — the pipeline is the deliverable, not the app.

## Requirements Inventory

### Functional Requirements

FR1: A planner can create an assignment through a simple UI, binding a person to a job over an inclusive date range (start date, end date, day granularity). Minimal fields only: person, job, start, end.
FR2: An assignment for a person whose date range does not overlap any existing assignment for that same person is accepted and persisted.
FR3: An assignment is refused if its date range overlaps an existing assignment for the same person; no double-booking is ever persisted. Clash = inclusive overlap — two assignments clash if they share any day, endpoints included (so 3–7 Aug clashes with 7–10 Aug).
FR4: (P0) When two clashing assignments for the same person are submitted concurrently, exactly one is persisted and the other is rejected. Enforcement is atomic at the data layer — never a read-then-check that can interleave.
FR5: The rejected attempt is deterministic and distinguishable from a system error — never a 500, timeout, or silent no-op. API layer returns HTTP 409 Conflict identifying the clash (person, dates); UI layer shows the planner a plain-language message.

### NonFunctional Requirements

NFR1: Concurrency correctness is the primary quality attribute and the P0 risk; it is enforced at the data layer and proven by AC1. This is the one place depth is warranted — everything else is deliberately thin.
NFR2: Azure only, UK data residency. Anything hosted or stored targets Azure in a UK region; never another cloud or a non-UK region. (Constitution)
NFR3: No secrets in code or git — use environment variables / Azure Key Vault. Validate and sanitise all external input. (Constitution)
NFR4: ATDD is non-negotiable and test-first — Playwright acceptance tests are authored from the story AC, red before implementation, and never weakened, skipped, or rewritten to go green. (Constitution)
NFR5: Spec-as-contract — if behaviour changes, the PRD and the linked story change in the same PR; drift is treated as a failure.
NFR6: Slices stay small — the whole feature lands in one PR a single reviewer can read and own; the PR carries the AC→test table, the spec in-diff with author + timestamp, and the two-way board link.
NFR7: Review focuses where risk is — the reviewer reads the tests closely, skims the implementation, and watches sensitive files (dependencies, package.json, infra).

### Additional Requirements

Starter template: none specified. There is no starter/greenfield template; the scaffold (UI + Node service + database) is pulled into existence by the first story (PRD-first, not scaffold-first). The JS stack versions stay deferrable until a story demands them.

- AD-1: Non-overlap is a database constraint, not application logic. Postgres exclusion constraint on `assignment`: `EXCLUDE USING gist (person_id WITH =, daterange(start_date, end_date, '[]') WITH &&)` (requires `btree_gist`). The losing concurrent write fails at commit with SQLSTATE `23P01`. No code path (service, UI, or job) may implement its own overlap check.
- AD-2: The service translates the violation; it does not detect clashes. Its only role is to catch `23P01` and map it to the loser contract — no SELECT-then-decide. On `23P01` the transaction is rolled back and the response body is built from the request payload + caught error only, never a follow-up query inside the failed transaction (which raises `25P02`).
- AD-3: The loser gets a deterministic rejection at two layers. API returns HTTP 409 Conflict with the AD-6 body; UI shows a plain-language message bound to a stable `data-testid` hook (copy finalized in build). Every expected rejection (clash, range validity, unknown ids) is a `4xx`, never a `500`.
- AD-4: The date range is the single source of the clash rule. Dates stored as an inclusive range at day granularity: bare `YYYY-MM-DD` in `date` columns, expressed as `daterange(start_date, end_date, '[]')`. No `timestamp`/`timestamptz` type touches an assignment date anywhere. Range validity (`end ≥ start`; `start == end` is a valid single-day booking) is enforced by the API as a `4xx` precondition.
- AD-5: The extension is a version-controlled step, not environment tribal knowledge. Enabling `btree_gist` is part of the tracked schema: `azure.extensions` must list it (Azure server parameter) AND a migration runs `CREATE EXTENSION IF NOT EXISTS btree_gist`.
- AD-6: The 409 body is a fixed contract shared by API and UI: `{ "code": "DOUBLE_BOOKING", "personId": <uuid>, "personName": <string>, "start": "YYYY-MM-DD", "end": "YYYY-MM-DD" }`. The UI renders `personName`; tests assert on `code`, not prose.
- AD-7: The P0 race is pinned deterministically, not left to UI timing. Each `create` is exactly one transaction. AC1's exactly-one-wins is proven by two genuinely concurrent committed transactions at the service/DB level; the Playwright two-context UI test complements it end-to-end but does not substitute for the deterministic pin.
- Structural seed (layered single-app spike): `ui/` (React planner form), `api/` (Node service: routes + 23P01→409 mapping, no clash logic), `db/migrations/` (schema + CREATE EXTENSION + EXCLUDE constraint), `tests/e2e/` (Playwright AC1/AC2/AC3), `tests/api/` (AC4 direct-to-API 409).
- Entities: `Person`, `Job`, `Assignment`; Person/Job pre-seeded (not managed here); `person_id`/`job_id` are NOT NULL FKs; `uuid` primary keys.
- Stack pinned: PostgreSQL 17 (Azure Database for PostgreSQL Flexible Server, UK South) + `btree_gist` + Playwright are load-bearing/confirmed. Node (LTS) and React are indicative, fixed at scaffold.
- Test isolation: acceptance/P0 runs use a fresh, isolated database — never shared mutable state — so "exactly one wins" is repeatable.

### UX Design Requirements

None. No UX design contract exists for this spike. UI message copy and surfacing (inline vs toast/banner; whether the form clears or preserves attempted dates) are deferred to story level; AD-3 fixes only the stable-hook + plain-language contract.

### FR Coverage Map

FR1: Epic 1 — Create assignment via UI (person → job over inclusive date range)
FR2: Epic 1 — Non-clashing assignment succeeds and persists
FR3: Epic 1 — Clash rejected (inclusive overlap); enforced by DB EXCLUDE constraint (AD-1, AD-4)
FR4: Epic 1 — Concurrency atomic, exactly-one-wins (P0); AD-1 + AD-7
FR5: Epic 1 — Deterministic loser contract, HTTP 409 + UI message; AD-2, AD-3, AD-6

All 5 FRs mapped to Epic 1; no gaps. NFRs are addressed across the epic (NFR2–NFR7 are cross-cutting/constitutional; NFR1 concurrency correctness is the P0 spine of the feature stories; NFR6/NFR7 are delivered by the two enabler stories).

## Epic List

### Epic 1: Double-Booking Guard — human-owned, test-first delivery

A planner can create person → job assignments over an inclusive date range through a minimal UI, and the system atomically refuses double-bookings for the same person — proven by a red-first P0 concurrency test — delivered through the AI-drafts / human-approves pipeline the spike exists to demonstrate. This is a single-component, single-PR slice: the architecture spine is final and every requirement touches the same `ui/` + `api/` + `db/` core, so the work is one epic with ordered stories rather than split epics.

**FRs covered:** FR1, FR2, FR3, FR4, FR5 (all)

**NFRs addressed:** NFR1 (concurrency correctness — P0 spine), NFR2–NFR5 (Azure/UK, secrets, ATDD, spec-as-contract — cross-cutting), NFR6 (small single-reviewer PR + two-way board link — enabler story), NFR7 (risk-based review gate — enabler story)

**Story ordering (detailed in Step 3):** the two phase-zero enablers come first as ordered stories — story→board sync (one-way repo → Linear: each story create/update in version control is mirrored to its Linear ticket; NFR6) and the PR review gate (multi-model GitHub Agentic Workflows + named-human approval, NFR7) — because the pipeline is the deliverable and these must precede feature stories. Then the data-layer story (schema + EXCLUDE constraint, pulled into existence PRD-first), then the feature stories. Stories are ordered so each depends only on earlier ones (no forward dependencies) and each fits a single dev-agent context.

## Epic 1: Double-Booking Guard — human-owned, test-first delivery

A planner can create person → job assignments over an inclusive date range through a minimal UI, and the system atomically refuses double-bookings for the same person — proven by a red-first P0 concurrency test — delivered through the AI-drafts / human-approves pipeline the spike exists to demonstrate.

### Story 1.1: Story→board sync — repo story spec mirrored to Linear (one-way) (enabler)

As a delivery team,
I want every create or update of a BMAD story in the repo to be reflected in its linked Linear ticket, one-directionally (repo → Linear),
So that the board view always mirrors the canonical version-controlled story without a second source of truth or a parallel GitHub Issues copy. (NFR6)

**Acceptance Criteria:**

**Given** a new BMAD story spec is created in the repo (e.g. the stories produced by this session)
**When** the sync runs
**Then** a linked Linear ticket is created reflecting the story
**And** the linking identifier (Linear key) ties the ticket to the story spec.

**Given** an existing BMAD story spec is updated in the repo
**When** the sync runs
**Then** the linked Linear ticket is updated to match the repo story
**And** the repo story spec remains the source of truth.

**Given** the sync direction
**When** documented in the ways-of-working
**Then** it is stated as one-way (repo → Linear): the BMAD story spec in version control is canonical, Linear is a downstream mirror
**And** changes made only in Linear are not authoritative and do not flow back into the repo.

### Story 1.2: PR review gate — multi-model review + named-human approval (enabler)

As a reviewer/approver,
I want a multi-model automated review to run on every PR and a named human to own the merge approval,
So that AI-authored code is demonstrably human-owned. (NFR7; constitution rule 5)

**Acceptance Criteria:**

**Given** a PR is opened
**When** the review workflow triggers
**Then** a multi-model automated review runs (GitHub Agentic Workflows)
**And** its trail is posted on the PR.

**Given** the automated review has run
**When** merge is attempted
**Then** merge is blocked until a named human approves
**And** no agent can mark the work "done" itself.

**Given** the review-gate configuration
**When** committed
**Then** it lives in version control (workflow file), not manual/tribal setup.
**And** the spike carve-out is honoured: GitHub out-of-the-box review models are used; Azure-only is waived for review tooling only.

**Given** the feature PR
**When** opened
**Then** its description restates the story acceptance criteria verbatim, with a table mapping each AC to its covering test
**And** the story/spec appears in the diff with author + timestamp and the Linear key — so behaviour and its spec change together (spec-as-contract). (NFR6, NFR5)

### Story 1.3: Data layer — double-booking impossible by construction

As a planner,
I want my assignments stored in a schema that makes double-booking impossible at the data layer,
So that no overlapping booking for a person can ever persist, regardless of application code. (FR3 substrate; AD-1, AD-4, AD-5 — pulls the scaffold into existence, PRD-first)

**Acceptance Criteria:**

**Given** a fresh database
**When** the version-controlled migration runs
**Then** `btree_gist` is enabled via `CREATE EXTENSION IF NOT EXISTS` (with `azure.extensions` listing it)
**And** the `assignment` table exists with `EXCLUDE USING gist (person_id WITH =, daterange(start_date, end_date, '[]') WITH &&)`. (AD-1, AD-5)

**Given** an existing assignment for a person over 3–7 Aug
**When** an overlapping range for the same person — including the boundary case 7–10 Aug — is inserted directly
**Then** the insert fails with SQLSTATE `23P01`
**And** nothing is persisted. (AD-1, AD-4 boundary)

**Given** assignment dates
**When** stored
**Then** they use bare `YYYY-MM-DD` values in `date` columns
**And** no `timestamp`/`timestamptz` type touches an assignment date anywhere. (AD-4)

**Given** a non-overlapping range for the same person
**When** inserted directly
**Then** it succeeds and persists. (FR2 at the data layer)

**Given** a person booked 3–7 Aug
**When** an adjacent non-overlapping range (8–10 Aug) for the same person is inserted directly
**Then** it succeeds — pinning the rule as inclusive but not off-by-one wide (the mirror of the 7–10 clash). (FR2, AD-4)

**Given** a person booked 3–7 Aug
**When** a different person is booked for the same 3–7 Aug
**Then** it succeeds — the constraint is scoped per `person_id`. (AD-1)

**Given** a single-day range (`start == end`)
**When** inserted directly
**Then** it is accepted as a valid booking
**And** a second single-day booking for the same person on that day clashes with SQLSTATE `23P01`. (AD-4 single-day validity + boundary)

### Story 1.4: Create assignment — happy path (API)

As a planner,
I want to submit a person → job assignment over a date range to the API and have a non-clashing one persisted,
So that valid bookings are recorded. (FR1 API, FR2)

**Acceptance Criteria:**

**Given** the person has no assignment for the requested range
**When** a planner POSTs a valid assignment (person, job, start, end)
**Then** it is persisted
**And** the API returns success with the created assignment. (FR1, FR2)

**Given** an invalid range (`end < start`), a missing field, or an unknown person/job id
**When** the request is submitted
**Then** the API returns a `4xx` precondition error — `400` for bad input, `404` for unknown ids
**And** nothing is persisted, and no `500` is returned. (AD-3, AD-4 range validity, sanitise-input)

**Given** a create request
**When** it is handled
**Then** it executes as exactly one transaction. (AD-7 groundwork)

### Story 1.5: Clash rejection — deterministic 409 contract (API)

As a planner whose booking clashes,
I want a clear, deterministic 409 rejection instead of a system error,
So that I know the person is already booked. (FR3 observable, FR5 API; AD-2, AD-3, AD-6 — covers AC4)

**Acceptance Criteria:**

**Given** the person is already booked 3–7 Aug
**When** a clashing assignment for that person is submitted directly to the API
**Then** the API responds `HTTP 409 Conflict` with body `{ "code": "DOUBLE_BOOKING", "personId": <uuid>, "personName": <string>, "start": "YYYY-MM-DD", "end": "YYYY-MM-DD" }`
**And** nothing is persisted. (AC4, FR5 API, AD-6)

**Given** the person is booked 3–7 Aug
**When** a booking that shares only the final day (7–10 Aug) is submitted to the API
**Then** it is rejected with `409`
**And** nothing is stored. (AC3 boundary at API level, FR3)

**Given** a clash occurs
**When** the service builds the rejection response
**Then** the body is derived from the request payload plus the caught error only — no `SELECT` inside the aborted transaction (no `25P02`), no `500`
**And** the service performs no overlap check of its own; the constraint is the only source of the rule. (AD-2, AD-3)

### Story 1.6: Concurrency P0 — exactly-one-wins, pinned deterministically

As the delivery team,
I want the P0 race proven by two genuinely concurrent committed transactions at the service/DB level,
So that "exactly one wins" is guaranteed, not accidentally serialized. (FR4, NFR1, AD-7 — P0; covers AC1 at service level)

**Acceptance Criteria:**

**Given** the person has no assignment for 3–7 Aug
**When** two clashing create transactions for that person over those dates are committed genuinely concurrently
**Then** exactly one persists
**And** the other is rejected via the 409 contract. (AC1 P0, FR4, AD-1)

**Given** the concurrency test
**When** it is written
**Then** it uses two separate connections/transactions (never one shared connection) against a fresh, isolated database
**And** the race is genuinely exercised, not accidentally serialized. (AD-7, test isolation)

**Given** the P0 test
**When** it is authored
**Then** it is red before any implementation exists
**And** it is never weakened, relaxed, or rewritten to go green. (NFR4 ATDD)

**Given** two concurrent create transactions for the same person over adjacent non-overlapping ranges
**When** both commit
**Then** both persist — the guard rejects only genuine clashes, not all concurrency for a person. (FR4, FR2)

### Story 1.7: Planner UI — create assignment and see the outcome

As a planner,
I want a minimal UI to create an assignment and see success or a plain-language clash rejection,
So that I can staff people without double-booking them. (FR1 UI, FR5 UI — covers AC2, AC3 via UI, AC1 two-context end-to-end)

**Acceptance Criteria:**

**Given** the person has no clashing assignment
**When** a planner books them for a free date range through the UI
**Then** the booking is stored
**And** success is shown. (AC2, FR1, FR2)

**Given** the person is already booked 3–7 Aug
**When** a planner submits an overlapping booking via the UI — including one sharing only the final day (7–10 Aug)
**Then** the UI shows the clash-rejection message, bound to a stable `data-testid` and rendering `personName` from the 409 body
**And** nothing is stored. (AC3 via UI, FR3, FR5, AD-3, AD-6)

**Given** the person has no assignment for 3–7 Aug
**When** two planners in two separate browser contexts submit clashing bookings for those dates at the same moment (`Promise.all` on the two submits)
**Then** exactly one booking is stored, the winning planner sees success, and the losing planner sees the clash-rejection message. (AC1 end-to-end, FR4, FR5)

**Given** a booking is rejected for a clash
**When** the clash message is shown
**Then** the planner's attempted dates are preserved in the form (not cleared)
**And** the booking can be corrected and resubmitted without re-entry. (resolves the addendum's open story-level UX detail; FR5)

**Given** the UI acceptance tests
**When** they are authored
**Then** they key on the stable `data-testid` / ARIA role, never the message copy. (AD-3, test convention)
