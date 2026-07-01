# PRD Quality Review — Double-Booking Guard (BST Scheduling Spike)

## Overall verdict

This is an unusually tight, self-aware PRD for what it is: a one-feature rehearsal spike whose real deliverable is a way of working, not a product. It knows its own thesis (AI-authored code is safe because a named human owns it, proven by a small test-first PR), scopes ruthlessly, and states its three product↔engineering decisions as decisions with a date and owner. It is fit to feed ATDD. The one substantive gap that matters for spec-as-contract: **FR5 asserts an API-layer HTTP 409 contract that no acceptance criterion actually verifies** — every AC drives the UI. Fix that traceability hole and this is a clean pass.

## Decision-readiness — strong

Decisions are stated as decisions, not smuggled in as "considerations." §9 names three resolved trade-offs with an owner and date (PO, 2026-07-01), and each links back to the FR it governs (FR3, FR5, FR1/FR4). The clash-definition decision (§9.1) is honest about what it supersedes ("the brainstorm's deferral of the boundary/partial edges") and *why* — it simplifies the rule rather than adding edge-handling. That is exactly the kind of trade-off surfacing the rubric wants. The concurrency mechanism is correctly left open (FR4 fixes the observable contract, defers the *how* to the addendum/architecture) rather than being falsely resolved.

No red flag of everything-balances-everything. The PRD is explicit that concurrency correctness is *the* quality attribute and everything else is "deliberately thin" (§7) — a real prioritisation, not a neutral smoothing.

### Findings
- **low** No open questions remain (§9 closes all three) — appropriate for a green-light-to-build spike, noted only to confirm it is intentional, not an omission.

## Substance over theater — strong

Almost no furniture. The single UJ (§4) is load-bearing: it names protagonists (Priya, Marcus, Sam), fixes concrete dates (3–7 Aug), and enumerates the failure modes the design must avoid ("not a crash, a generic error, or a silent success that later vanishes") — it drives AC1 directly. Success Metrics (§8) are genuinely process metrics matched to the thesis, and every one carries a counter-metric ("a test relaxed or an AC dropped to force green"), which is precisely the anti-theater discipline the rubric rewards. No persona bloat, no invented differentiation section, no boilerplate NFRs (the constitution's NFRs are referenced, not copied — correct by design).

### Findings
- None. This dimension is clean.

## Strategic coherence — strong

The PRD has a clear thesis and every part serves it. The feature was chosen *because* its one hard edge (concurrent clash) forces a real spec conversation (§1), the metrics validate the way-of-working thesis rather than measuring app activity, and scope is cut to match (§3 out-of-scope: skills-matching, auth, calendars, UI polish). This is the opposite of a backlog with headings.

### Findings
- None.

## Done-ness clarity — adequate (one real gap)

Most FRs carry a testable consequence. FR2/FR3 are crisp ("accepted and persisted" / "no double-booking is ever persisted"), FR3 pins the ambiguous case with a worked example (3–7 Aug clashes with 7–10 Aug), and FR4 states an observable contract ("exactly one wins"). The AC set is the strength here: AC1–AC3 are Given/When/Then and name the exact assertions.

The gap is a coverage hole, not vagueness: **FR5 specifies two layers — API returns HTTP 409, UI shows a message — but all three ACs are UI-driven and assert only the "already booked" message.** The 409 half of FR5 has no covering AC, so the spec-as-contract chain into ATDD is incomplete for that clause. The addendum even flags a "thin service/DB-level check may complement the UI test," acknowledging the UI path alone cannot pin everything — but that complementary check is not lifted into an AC, so it is not part of the contract.

Secondary: FR5's UI copy is explicitly "finalized in the build," which is fine, but AC1/AC3 assert the planner "sees the 'already booked' message" — the test needs a stable assertion target (a role/testid or a substring), not final marketing copy. Worth a one-line note so the ATDD author does not assert on volatile text.

### Findings
- **high** FR5's API-layer 409 contract has no covering AC (§6 vs FR5) — every AC drives the UI; the HTTP 409 clause is asserted nowhere, leaving half of a spec-as-contract FR untested. *Fix:* add AC4 (API/integration level): "when a clashing assignment is POSTed, the API returns HTTP 409 identifying the clash (person, dates) and nothing is persisted," and mark FR5 as covered by AC1+AC3 (UI) **and** AC4 (API). This also realises the addendum's "thin service/DB-level check" as contract, which strengthens the deterministic-race guarantee.
- **medium** ACs assert on UI copy that is deliberately not final (§6 AC1/AC3, FR5) — asserting on "already booked" text couples the P0 test to unfinalised copy. *Fix:* state the assertion target as a stable locator (e.g. a `data-testid="clash-error"` region or a documented invariant substring) so copy can change without breaking the test.
- **low** FR1 says "day granularity" but no FR states timezone/date-boundary handling. *Fix:* one line noting dates are civil calendar days (no time-of-day), so "shares any day" is unambiguous for the DB constraint author. Likely obvious for a spike, hence low.

## Scope honesty — strong

Omissions are explicit, not inferred. §3 lists out-of-scope items by name (skills-matching, availability calendars, auth/roles, UI polish) and states the seeding assumption ("Person and job records are assumed pre-seeded"). The 1–2 day target is stated. No silent de-scoping. Open-items density is essentially zero, which is correct for a build-ready spike of this size — the rubric explicitly says high counts are only a problem on green-light PRDs, and here there are none.

### Findings
- **low** "UI polish… beyond what is needed to show the guard working" is a soft boundary (§3) — an implementer could over- or under-build. Acceptable for a 1–2 day spike, but a one-line floor ("a form + a success/rejection message, no styling system") would remove judgement calls.

## Downstream usability — adequate

This PRD is chain-top (feeds ATDD, story, architecture), so traceability matters. FR/AC IDs are contiguous and unique (FR1–FR5, AC1–AC3), and every AC cites the FRs it covers inline. Cross-references resolve (§9 ↔ FRs, addendum ↔ FR4/FR5). The single UJ has named protagonists. The reverse map is complete on the FR side — FR1✓(AC2) FR2✓(AC2) FR3✓(AC1,AC3) FR4✓(AC1) FR5✓(AC1,AC3, minus the 409 clause noted above).

Weaknesses are minor: there is **no Glossary**, so domain nouns (assignment, clash, person, job, planner) rely on being used consistently in prose. They *are* used consistently here, so this is low-cost for a 2-page doc, but a 5-line glossary would let the architecture/story steps source-extract "clash = inclusive overlap" without re-reading FR3's prose. The AC→test mapping is promised in the PR description (§6, §8) rather than in the PRD, which is the right place for it.

### Findings
- **medium** No Glossary; the load-bearing term "clash" is defined only inline in FR3 (§5) — downstream (architecture DB constraint, ATDD) must re-derive it from prose. *Fix:* add a 4–5 line glossary (assignment, clash = inclusive date-range overlap, person, job, planner) so the invariant is extractable as a unit. Low effort, real downstream payoff given the whole feature hinges on this one definition.

## Shape fit — strong

Correctly shaped. This is a single-operator-style internal capability spike, and it uses the capability-spec shape (FR-centric, one illustrative UJ, operational/process SMs) rather than over-formalising with a persona roster or multiple UJs. It is not under-formalised either — the one race UJ is exactly where a named-protagonist journey earns its place because concurrency is the whole point. The constitution constraints are referenced not restated, which the brief confirms is by design. No forced shape.

### Findings
- None.

## Mechanical notes

- **ID continuity:** FR1–FR5 and AC1–AC3 contiguous and unique. No gaps or duplicates. Cross-refs (§9→FR, addendum→FR) all resolve.
- **FR↔AC roundtrip:** every FR maps to ≥1 AC; every AC cites its FRs. Only defect is the FR5 *sub-clause* (HTTP 409) with no AC — see High finding.
- **Glossary:** absent. Terms used consistently in prose; see Medium finding.
- **Assumptions Index:** no inline `[ASSUMPTION]` tags. Given all decisions are resolved in §9 with an owner, this is acceptable; the "pre-seeded records" and "indicative stack" assumptions live in §3 and the addendum respectively rather than a formal index — fine at this size.
- **UJ protagonist naming:** UJ-1 names Priya, Marcus, Sam inline. Good.
- **Addendum discipline:** downstream depth (concurrency mechanism options A/B, UX surfacing, stack) is correctly parked in addendum.md, not leaked into the PRD body. Well done.
