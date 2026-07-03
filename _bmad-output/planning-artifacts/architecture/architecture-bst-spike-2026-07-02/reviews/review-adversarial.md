---
name: 'Adversarial review — Double-Booking Guard spine'
type: architecture-review
reviews: 'ARCHITECTURE-SPINE.md'
stance: adversarial
created: '2026-07-02'
---

# Adversarial review — Double-Booking Guard spine

**Verdict:** The spine is directionally strong — pinning the clash rule to a single Postgres `EXCLUDE` constraint (AD-1/AD-2/AD-4) genuinely removes the classic read-then-check race and the two-owners problem for the *rule*. But it under-specifies enough **shared data shapes and boundary semantics** that two units can each obey AD-1..AD-5 to the letter and still build incompatibly, or ship a P0 test that passes without proving the guarantee. Below are concrete incompatible pairs, each with the tightening I'd add. Findings H1–H4 are the ones that break the feature or the P0 test; the rest are lower.

Method: for each finding I construct two units, one level down, each fully compliant with the spine, that diverge.

---

## H1 — [HIGH] The 409 body shape is named but not pinned; API and UI units diverge on it

**The two units.** The API story reads AD-3 ("returns HTTP 409 Conflict **with a body identifying the clash (person + dates)**") and the error-shape convention ("a body naming the clashing person + dates"). The UI story reads the same words. Both comply.

- API unit ships `{ "error": "conflict", "personId": "…", "start": "2026-08-03", "end": "2026-08-07" }`.
- UI unit, needing to render *"Sorry, Sam's already booked for those dates"*, expects `{ "conflict": { "person": { "id": "…", "name": "Sam" }, "range": { "from": "…", "to": "…" } } }` — because the copy names the *person* ("Sam"), not a UUID, and AD-3 says the message is "plain-language."

**The divergence.** Both satisfy AD-3 word-for-word. The UI cannot render the human name from the API body (only `personId` is present), and the field names/nesting don't match, so the UI either shows a UUID, shows nothing, or the units simply don't wire together. AC4 (API-level) can still pass green while AC1/AC3 (UI) fail — a split that hides the break until integration.

**Sharper still:** does the body carry the *incoming* dates the loser tried, the *existing* winning assignment's dates, or both? AD-3 says "the clash (person + dates)" — that phrase reads two ways. For UJ-1 the useful message is about the *existing* booking; a unit could ship either.

**Tightening — new AD-6 (409 body is a pinned schema).** Fix the exact JSON: field names, whether `personId` **and** a display `personName` are both present (UI needs a human label without a second fetch — and `Person` is pre-seeded, so the API can join it), and explicitly *which* dates the body carries (recommend: the attempted range, since the existing winning row may be one of several). One literal example object in the spine. This is the single most likely silent break.

---

## H2 — [HIGH] "Nothing is persisted" + "409" is a two-owner state-mutation path the spine leaves open

**The two units.** AD-2 says the service catches `23P01` and maps it; AC3/AC4 say "nothing is stored." The API story author, wanting a nice REST surface, wraps assignment creation so the endpoint does: (1) `INSERT`; on `23P01`, (2) `SELECT` the existing clashing row to build the rich 409 body H1 wants. That `SELECT`-after-fail is *not* a "SELECT-then-decide" clash check — the decision was already made by the constraint — so it does **not** violate AD-2 as written ("performs no `SELECT`-then-decide" / "no code path may implement its own overlap check"). Fully compliant.

**The divergence.** Now two units can legitimately touch assignment state in one request within a transaction: the failed `INSERT` and the follow-up `SELECT`. If the `INSERT` and the `SELECT` are in the **same** transaction, the `23P01` has already aborted it — the follow-up `SELECT` errors with `25P02 (in_failed_sql_transaction)`, and the planner gets a 500, violating AD-3 ("a clash is never surfaced as a system error"). A compliant-looking API unit produces the exact failure AD-3 forbids. Whether this happens depends entirely on transaction/`ROLLBACK` framing the spine never states.

**Tightening — extend AD-2.** State the transaction contract: the create is a single statement/transaction; on `23P01` the service **rolls back**, then may issue a *separate* read (outside the aborted tx) to enrich the body, or (simpler, recommended for a spike) build the 409 body **only** from the request payload + the caught error, with no follow-up query. Pin one. This also closes H1's "which dates" ambiguity toward the attempted range.

---

## H3 — [HIGH] The P0 concurrency test can pass green without exercising the race

**The two units.** The DB/migration unit ships the `EXCLUDE` constraint (AD-1). The test unit writes AC1 per the addendum: two Playwright contexts, `Promise.all` on the two submits, assert exactly one wins. Both comply with AD-1 and the test conventions.

**The divergence — non-determinism the ADs permit.** "As close to simultaneous as possible" is not simultaneous. If the app serializes the two requests (single Node process, one DB connection, or an implicit per-request transaction that commits before the next starts), the *second* `INSERT` sees the first already **committed** and fails with `23P01` — the constraint works, "exactly one wins" holds, test is green. But this green proves only the **sequential** guard (already covered by AC3); the genuine *concurrent commit race* (both transactions in flight, one loses at commit) was never exercised. The P0 — the whole point of the spike — is asserted by a test that can't distinguish the race being handled from the race never occurring. Nothing in AD-1..AD-5 forces the test to prove concurrency *actually interleaved at the DB*.

Worse divergence: if the API unit ever runs with a read-then-check fallback disabled but pools connections such that both writes land in one transaction (batching), the constraint check is deferred to commit and still catches it — but a *different* API unit using `autocommit` per statement gets different interleaving. The test's green/red is a function of API-unit transaction choices the spine doesn't pin, so the P0 result is non-deterministic across compliant implementations.

**Tightening — new AD-7 (concurrency is proven, not hoped).** The P0 must include a **deterministic** race pin at the data/service layer (the addendum's "thin service/DB-level check" — make it mandatory, not "may"): e.g. two real concurrent transactions that both `INSERT` then both `COMMIT`, asserting exactly one raises `23P01` and one row exists. And pin that each assignment create commits in its **own** transaction (no request batching), so the UI-level AC1 exercises a real concurrent commit rather than accidental serialization. Without this, "the P0 is green" is not evidence the guarantee holds.

---

## H4 — [MEDIUM/HIGH] Day-granularity + timezone: `start_date`/`end_date` shape is interpretable two ways

**The two units.** AD-4 says dates are ISO-8601 **calendar dates**, day granularity, stored as `daterange(start_date, end_date, '[]')`. The migration unit types the columns as `date`. Compliant. The API unit, receiving JSON, parses the incoming `start`/`end` as `timestamptz` / JS `Date` (midnight UTC) before passing to the DB. Also arguably compliant — it's still "a calendar date."

**The divergence.** A planner in UK local time (BST, UTC+1 in August — note UJ-1 is *August*) submits "3 Aug". The API unit that does `new Date("2026-08-03")` gets `2026-08-03T00:00:00Z` = `2026-08-03 01:00 BST`; a unit that treats the string as a bare local date gets `2026-08-03`. If either boundary shifts by a day under TZ conversion, the *inclusive-overlap* result at the boundary (the "7 Aug touches 7–10 Aug" case, AC3's headline) flips. Two compliant units disagree on whether 7 Aug clashes — the exact case FR3/AC3 were written to nail. The spine pins the *storage* type region (`daterange`) but not that no time/zone component ever enters the pipeline.

**Tightening — extend AD-4.** State: the wire format is a bare `YYYY-MM-DD` string; the API must **not** coerce it through any timezone-bearing type; DB columns are `date` (not `timestamptz`); the whole feature is timezone-free (day granularity means *civil dates*, no instant). One sentence closes a boundary-flipping divergence in the one rule the spike exists to demonstrate.

---

## H5 — [MEDIUM] Degenerate / invalid ranges are unowned (`end < start`, and validity of a zero-length range)

**The two units.** FR1/AD-4 define an inclusive range but never say `end_date >= start_date`. The UI unit lets a planner pick end-before-start (or doesn't). The API unit passes it through (AD-2 forbids clash logic, and this feels like clash-adjacent logic, so a cautious author omits the check). `daterange('2026-08-07','2026-08-03','[]')` **throws** `range lower bound must be less than or equal to upper bound` — SQLSTATE `22000`, *not* `23P01`. AD-3 only maps `23P01`; this surfaces as a 500, violating "never a system error."

**The divergence.** UI unit assumes the API validates; API unit assumes the constraint/DB handles it or that it's out of scope; result is an unhandled 500 on a trivial input. Neither unit violates a stated AD. Also: is a single-day booking (`start == end`) valid? `'[]'` makes it a legal 1-day range that correctly clashes — fine — but nobody's told the units that's intended vs. an error.

**Tightening — small addition (could fold into AD-4 or the error-shape convention).** Declare range validity a **precondition** owned by exactly one layer (recommend API input validation → `400`, distinct from the `409` clash path), state `end >= start` is required, and confirm `start == end` (one-day booking) is valid. Keeps the `409` path meaning exactly "clash" (the error-shape convention says "no other path returns 409" — good — but says nothing about the malformed-input path).

---

## H6 — [LOW/MEDIUM] `Person`/`Job` are "pre-seeded" but the FK-failure and unknown-id paths are unowned

**The two units.** Conventions say `Person`/`Job` are pre-seeded, "not managed here." AC4 submits "directly to the API." An API-level test (or a fuzzing reviewer) posts an assignment with a `person_id` that isn't seeded. If `assignment.person_id` has an FK, this is `23503 (foreign_key_violation)` → unmapped → 500. If it has **no** FK (a unit could omit it — the spine's ERD shows the relationship but AD-nothing mandates the constraint), the row inserts against a non-existent person and the guard silently operates on orphan data.

**The divergence.** One unit adds FKs, another doesn't; the "pre-seeded, not managed" note is read by one as "so I don't need FKs" and by the other as "so FKs are safe to assume." Not fatal to the demo, but it's a two-interpretations gap on an entity relationship.

**Tightening.** One line: `assignment.person_id`/`job_id` are `NOT NULL` FKs to the seeded tables; unknown id is a `400`/`404`, not a `409` and not a `500`. Cheap, removes an orphan-data and a 500 path.

---

## H7 — [LOW] "Stable test hook" is named but not pinned; UI and test units can pick different hooks

**The two units.** AD-3 and the test convention both say tests key on a "stable UI hook (test id / ARIA role)." The UI unit ships `role="alert"` for the clash message. The test unit keys on `data-testid="clash-message"`. Both "comply." They don't match; AC1/AC3 fail on a selector, not on behaviour — noise that looks like a real failure and burns the small review budget.

**Tightening.** Pin the literal hook name(s) in the spine (e.g. `data-testid="clash-rejection"` for the loser, `data-testid="booking-success"` for the winner) so UI and test units share one contract. Trivial, but it's exactly the UI↔test seam AD-3 says matters.

---

## Summary of tightenings to add

| # | Sev | Hole | Add / tighten |
|---|-----|------|----------------|
| H1 | HIGH | 409 body shape unpinned → UI/API diverge | **AD-6**: literal 409 JSON schema (incl. `personName`, which dates) |
| H2 | HIGH | failed-INSERT + enrich-SELECT = 500 / two mutation touches | extend **AD-2**: transaction/rollback contract; build body from payload+error |
| H3 | HIGH | P0 can go green via accidental serialization | **AD-7**: mandatory deterministic race pin + one-tx-per-create |
| H4 | MED/HIGH | TZ coercion flips boundary overlap (August/BST) | extend **AD-4**: bare `YYYY-MM-DD`, `date` cols, no timezone type anywhere |
| H5 | MED | `end<start` / zero-length range unowned → 500 | precondition owned by API → `400`; confirm `start==end` valid |
| H6 | LOW/MED | pre-seeded FK / unknown-id path unowned | FKs `NOT NULL`; unknown id → `400`/`404`, not `409`/`500` |
| H7 | LOW | test hook named but not pinned → selector mismatch | pin literal `data-testid` values in spine |

The spine's core bet (rule lives only in the constraint) is sound and I could not break the *"two overlapping rows persist"* invariant itself — AD-1 holds. Every hole above is instead a **seam** the spine leaves to interpretation: the shared 409 shape (H1), the transaction/error framing around the catch (H2, H5, H6), the meaning of "concurrent" and "day" (H3, H4), and the UI↔test hook (H7). H1–H4 are the ones that will either break the feature at integration or let the P0 pass without proving the guarantee.
