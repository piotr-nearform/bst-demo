# Reconciliation Review — Architecture Spine vs. Source Inputs

**Spine reviewed:** `ARCHITECTURE-SPINE.md` (Double-Booking Guard), created 2026-07-02, status draft.
**Sources:** PRD (`prd.md`), Addendum (`addendum.md`), Constitution (`project-context.md`).
**Reviewer intent:** find what did NOT land — especially quiet requirements (tone, constraint, intent) the AD structure may have dropped.

---

## Verdict

The spine is **technically excellent and faithful on the mechanism** — AD-1 through AD-5 capture the concurrency guarantee, the exactly-one-wins contract, the two-layer loser contract, the inclusive-overlap rule, and the Azure/UK/secrets/extension constraints precisely, with clean traceability. **But it is silent on the spike's actual thesis.** The PRD is emphatic that *the app is not the deliverable — the reviewable, human-owned pipeline is* ("reviewability is the sales pitch"). The spine architects the app and drops almost everything about *how the code arrives and is owned*. Because an architecture spine is meant to govern everything built from it, this omission means the load-bearing selling point of the whole engagement has no invariant protecting it.

Severity legend: **HIGH** = a source intent the spine fails to govern and which will bite downstream; **MED** = partial/weak coverage; **LOW** = minor or over-reach worth noting.

---

## 1. PRD (`prd.md`)

### HIGH — The core thesis (reviewable, human-owned AI code) is unrepresented as an invariant
The PRD's §1 and §7 make reviewability the *point*: "AI-authored code is safe because a named human owns it," "reviewability is the sales pitch," "a hard constraint on *how*, not a nice-to-have." The spine never encodes this. There is no AD governing PR shape, slice size, or the read-the-tests/skim-the-impl review posture. An architecture that omits the one thing the spike exists to prove has dropped the requirement most costly to lose.
**Fix:** add an AD (e.g. AD-6 "The change is shaped to be owned") binding §7 — one small PR, tests-first evidence, AC→test table, spec-in-diff — as a first-class invariant, or explicitly delegate it to a named process artifact the spine points to. Do not leave it unowned.

### HIGH — "Slices stay small / one reviewable PR" is a structural constraint the spine ignores
§7 elevates small slices to "a hard constraint on *how*," with an explicit caveat that AI tools push toward large diffs. The spine's Structural Seed lays out `ui/ api/ db/ tests/` but says nothing about keeping the change reviewable in one sitting (a §8 success metric and counter-metric). This is a shaping constraint on the architecture output itself.
**Fix:** state, in Consistency Conventions or a new AD, that the feature lands as one small PR a single reviewer can own; note the AI-large-diff counter-pressure.

### HIGH — ATDD "red-first, never weakened" is under-governed
The PRD (§6, §8) and constitution both make ATDD non-negotiable: the P0 test must be **red first**, authored from AC *before* implementation, and **never weakened to go green** (explicit counter-metric). The spine's Tests convention says only that tests key on stable hooks and pin AC4 deterministically — it captures the *shape* of tests but not the *ordering/immutability* rule (test precedes code; test is never relaxed). That rule governs how every story built from this spine executes.
**Fix:** add to the Tests convention (or an AD) that acceptance tests are authored from AC and red before implementation, and are never weakened — a red AC means the code is wrong, not the test.

### MED — PR evidence artifacts (AC→test table, spec-in-diff, author+timestamp, both-way board link) not reflected
§2 and §8 enumerate concrete deliverable artifacts: PR description restating AC verbatim with an AC→test table, story+PRD spec in version control with author/timestamp visible in the diff, multi-model automated review trail, named human approval, and two-way `AB#<id>` GitHub↔Azure Boards linkage. The spine mentions none of these. Some are process, but the *traceability* ones (AC→test mapping, spec-in-diff) are architectural evidence the structure should anticipate.
**Fix:** at minimum note in Consistency Conventions that traceability (AC→test, spec-in-VC, `AB#<id>` both ways) is part of the delivered artifact set.

### MED — "Agents draft; a human owns and approves" absent
PRD §8 counter-metric: "any artifact marked 'done' by an agent" is a failure; a named human approves the merge. Constitution rule 5 says the same. The spine, itself an agent-drafted artifact, does not carry this stance forward to what it governs.
**Fix:** reflect the agents-draft/humans-approve gate in the conventions or the thesis AD.

### LOW — User journey tone ("not a crash, not a silent success that later vanishes") is captured but only implicitly
UJ-1 and FR5 stress the *felt* experience: a clear on-screen message, "Sam is never double-booked," never a silent success that later vanishes. AD-3 captures "never a 500/timeout/silent no-op" well. The human-reassurance tone is adequately governed. Noted as satisfied.

---

## 2. Addendum (`addendum.md`)

### Satisfied — Mechanism choice
The addendum defers the concurrency mechanism (Option A DB exclusion constraint vs Option B optimistic locking) and flags Option A as the lower-risk default. The spine adopts Option A (AD-1) — a legitimate architecture decision this artifact is entitled to make. **Good**, though see below on recording it.

### MED — The "record the mechanism choice in the story/PRD diff" instruction is not surfaced
The addendum says the *how* "is an architecture call, to be recorded in the story/PRD diff at implementation time per the demo runsheet." The spine picks Option A but does not note that this choice (and the rejection of Option B / naive read-then-check) must land in the diff as a recorded decision — a spec-as-contract obligation.
**Fix:** note that the AD-1 mechanism decision is itself a spec artifact that belongs in the PR diff.

### MED — "A thin service/DB-level check may complement the UI test to pin the race deterministically" — partially captured
The addendum and PRD AC4 both call for a service/DB-level test to pin the race deterministically because two near-simultaneous UI clicks are not truly simultaneous. The spine's `tests/api/` + AC4 convention captures the API 409 pin. It does **not** explicitly capture the *reason* — that the UI concurrency test alone cannot deterministically prove exactly-one-wins, so the deterministic pin is load-bearing, not optional. Downstream this risks the API test being treated as a nice-to-have.
**Fix:** in the Tests/Test-isolation convention, state why the deterministic API/DB-level pin is required alongside the two-context UI test.

### LOW — Story-level UX detail correctly deferred
Inline-vs-toast and clear-vs-preserve-dates are pushed to story level by both addendum and spine (Deferred §). Correctly handled.

### LOW — Python / Azure DevOps openness under-noted
The addendum warns KPMG "may push for Python and/or the full Azure DevOps suite over GitHub." The spine keeps Node/React indicative (good) but presents GitHub Actions / Azure Boards as settled in the deployment envelope and stack framing without flagging the DevOps-suite openness. Minor, since the constitution lists GitHub, but the indicative/confirmed line should not silently harden.
**Fix:** one line noting the toolchain (GitHub vs Azure DevOps) is indicative, per addendum.

---

## 3. Constitution (`project-context.md`)

### Satisfied — Azure-only / UK residency
Rule 1 is well reflected: Stack pins Azure Database for PostgreSQL Flexible Server, UK South; deployment envelope is an Azure-UK subgraph. **Good.**

### Satisfied — No secrets in code/git
Rule 2's secrets clause is captured in the Secrets convention and the Key Vault node. **Good.**

### MED — Input validation/sanitisation clause dropped
Rule 2 has two halves: "No secrets in code or git" **and** "Validate and sanitise all external input." The spine captures secrets but not input validation. FR1 takes planner-supplied dates/ids through the API — a natural place for the validation obligation.
**Fix:** add input validation/sanitisation to the conventions (e.g. reject malformed date ranges, `end < start`) — it is a constitution rule, not optional.

### MED — Spec-as-contract captured for schema, weak on behaviour-change discipline
Rule 4 / PRD §7: if behaviour changes, PRD + story change in the *same* PR; drift is a failure. The spine states "the constraint **is** the spec for FR3" (excellent for the schema) but does not carry the broader same-PR-or-it's-drift discipline. This is the ongoing rule that governs every future change to what the spine describes.
**Fix:** note in conventions that any behaviour change updates PRD+story in the same change; drift = failure (per constitution rule 4).

### MED — "Surface constraints, don't silently work around them" (rule 6) not reflected
Rule 6 is a meta-rule for every workflow. The spine's Deferred section makes lean calls (e.g. Container Apps) which is fine, but the surface-don't-circumvent stance isn't stated. Low-to-medium; mostly a process rule.

### Satisfied — ATDD/Playwright as source of truth
Constitution rule 3 (Playwright, authored from AC) is reflected in the Stack ("ATDD source of truth") and Tests convention — though the *test-first, never-weaken* ordering is the gap already raised under PRD HIGH.

---

## Over-reach beyond PRD scope

### LOW — PostgreSQL version breadth
Stack notes "16/17/18 all supported on Azure" and pins 17. Fine and useful, but slightly beyond the minimum; ensure the "pinned 17" is the governing statement and the version breadth is just rationale, not an invitation to drift.

### LOW — App-host lean (Container Apps)
The spine names a lean default (Container Apps) in Deferred while keeping it deferred. This is within an architect's remit and explicitly marked deferred/revisitable — not true over-reach, noted for completeness.

### None material
The spine is disciplined about scope: it explicitly lists skills-matching, availability, auth/roles, person/job management, and UI polish as out (matching PRD §3). No speculative abstraction — it honours the "smallest thing that works" constitution preamble. The over-reach risk is low; the real risk is *under*-reach on the process thesis.

---

## Summary of gaps by theme

| Theme | Source | Status in spine | Severity |
| --- | --- | --- | --- |
| Reviewable/human-owned AI code (the thesis) | PRD §1, §7 | Absent | HIGH |
| Small slice / one reviewable PR | PRD §7, §8 | Absent | HIGH |
| ATDD red-first, never weakened | PRD §6/§8, Const. 3 | Test *shape* only; ordering/immutability missing | HIGH |
| PR evidence: AC→test table, spec-in-diff, `AB#` both ways | PRD §2, §8 | Absent | MED |
| Agents draft / humans approve | PRD §8, Const. 5 | Absent | MED |
| Record mechanism choice in diff | Addendum | Choice made, obligation not noted | MED |
| Why the deterministic API/DB pin is required | Addendum, PRD AC4 | API test present; rationale missing | MED |
| Input validation/sanitisation | Const. 2 | Absent (secrets half only) | MED |
| Spec-as-contract behaviour-change discipline | Const. 4, PRD §7 | Schema only | MED |
| Exactly-one-wins P0 contract | PRD FR4/AC1 | **Captured (AD-1)** | OK |
| Two-layer loser contract (409 + UI) | PRD FR5/AC4 | **Captured (AD-3)** | OK |
| Inclusive overlap, single source | PRD FR3 | **Captured (AD-4)** | OK |
| Azure/UK/secrets/extension | Const. 1/2, addendum | **Captured (AD-5, Stack, Secrets)** | OK |

**Bottom line:** the spine governs the *artifact* impeccably and the *pipeline that makes the artifact trustworthy* barely at all. For a spike whose stated deliverable is the pipeline, that inversion is the finding that matters most. Recommend adding a thesis/ownership AD and folding the ATDD-ordering, input-validation, and spec-as-contract-behaviour rules into the conventions before this spine governs any story.
