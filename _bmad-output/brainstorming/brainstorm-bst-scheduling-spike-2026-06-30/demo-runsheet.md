# Demo Runsheet — Double-Booking Guard Spike

> Scope is deliberately tiny: **concurrent-clash only** — no skills-matching, availability, partial-overlap, touching-boundary, or UI polish. The process is the point, not the app.

---

## Part A — The 4-minute KPMG demo flow (the money-shot)

Demoing **one human-approved PR** that proves AI-authored-but-human-owned code. Feature: assign a person to a job over a date range; refuse clashing assignments. Edge in scope: two planners booking the same person at the same instant — exactly one wins.

**1. Open the PR (~40s)**
- Say: "Here's the whole feature in one PR — small enough that one person can read and own it."
- Show: compact PR description restating the AC verbatim, with a table mapping each AC to the test that covers it.

**2. The green P0 concurrency test (~60s)**
- Say: "This is the rule that matters, and here's the test proving it holds."
- Show: the passing P0 Playwright test firing two concurrent clashing assignments — assert exactly one succeeds, one is rejected.

**3. The spec decision in version control (~60s)**
- Say: "The product decision lives in version control, not someone's head — when and where it was made."
- Show: linked story + PRD entry choosing DB exclusion constraint / optimistic locking and defining what the losing planner sees; point to author + timestamp in the diff.

**4. The review trail (~50s)**
- Say: "Breadth comes from automated multi-model review; the call comes from a named person."
- Show: multi-model automated review pass on the PR, then the final approval with a named human reviewer attached.

**5. Closing line (~30s)**
- Say: "AI wrote most of this. A person owns it. Here's exactly where, and here's the test proving the rule holds."
- Tie back: a named human owning the merge is **more** accountability than today, not less — AI is the tooling that makes that ownership reviewable.

---

## Part B — Build order (1–2 days), PRD-first

The BMAD pipeline pulls the React/Node/Azure scaffold into existence as the first story demands it — **not scaffold-first**.

1. **Write the first PRD line + decision.** State the AC and the concurrency mechanism (exclusion constraint / optimistic locking) + losing-planner UX. *Human owns this call.*
2. **Create the AB# story** in Azure Boards; link GitHub via `AB#<id>` so commits/PRs update the ticket. Meet the client where they are; PRD-in-VC stays canonical.
3. **Write the red P0 acceptance test first** (Playwright ATDD) from the AC — two concurrent clashing assignments, expect exactly one winner. Test fails (no impl yet).
4. **Let the scaffold emerge** — first story forces minimal React/Node/Azure setup. Azure-only, UK hosting. *Human approves any sensitive/infra/dep change.*
5. **AI implements to green** — smallest change that passes the P0 test; keep the slice vertical and PR-readable.
6. **Multi-model automated review** (breadth pass) on the PR; reviewer focuses on test evidence, skims impl, watches sensitive files.
7. **Named human approval + merge.** *Human owns the merge and the accountability.* Spec-as-contract: if code drifts from the PRD/story, that's a failure — fix the spec or the code.
