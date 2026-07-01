# Input Reconciliation — demo-runsheet.md vs PRD

**Source:** `_bmad-output/brainstorming/brainstorm-bst-scheduling-spike-2026-06-30/demo-runsheet.md`
**Deliverable:** `prd.md` (+ `addendum.md`)
**Date:** 2026-07-01
**Purpose:** Surface anything from the runsheet DROPPED or DILUTED in the PRD. No rewrites proposed.

---

## Method

The runsheet has two parts:
- **Part A** — the 4-minute demo money-shot: four artefacts that must be visible in the one PR.
- **Part B** — the PRD-first build order (7 steps).

I check whether PRD §2 (Goal & Success) and §8 (Success Metrics) faithfully capture every element of the money-shot, then whether anything build-order-relevant that belongs in a PRD is missing.

---

## Part A — Money-shot element-by-element

The runsheet's money-shot is four artefacts in one human-approved PR:

| # | Runsheet element (Part A) | In PRD §2? | In PRD §8? | Verdict |
|---|---|---|---|---|
| 1 | Compact PR description restating AC **verbatim**, with a table mapping **each AC → its test** | §2 bullet 1: "restating the acceptance criteria and mapping each AC → the test that covers it" | §8 bullet 3: "every AC maps to a test in the PR description" | **Captured** (see dilution D1 on "verbatim" + "table") |
| 2 | Green **P0 concurrency** Playwright test — two concurrent clashing assignments, **exactly one wins** | §2 bullet 2, verbatim intent | §8 bullet 1: "AC1 P0 test is green and asserts exactly-one-wins (not weakened to pass)" | **Captured, strongly** |
| 3 | Spec decision **in version control** — linked story + PRD entry, **author + timestamp in the diff** | §2 bullet 3: "story + this PRD's spec decision in version control, with author and timestamp visible in the diff" | §8 bullet 3: "PRD/story decision is in the diff with author + timestamp" | **Captured** |
| 4 | **Multi-model** automated review trail + **named human** approval on merge | §2 bullet 4, verbatim intent | §8 bullet 4: "named human approves the merge after the multi-model review pass" | **Captured** |

**Headline: all four money-shot artefacts survive into both §2 and §8.** The reconciliation is fundamentally clean. The gaps below are dilutions of narrative/framing detail, not dropped requirements.

---

## Ranked gaps

### G1 — "Restating the AC **verbatim**" and the **AC↔test table** are softened (DILUTED) — Medium

Runsheet step 1 is specific: the PR description "restating the AC **verbatim**, with a **table** mapping each AC to the test that covers it." Two concrete demo-visible properties:
- **verbatim** restatement (not paraphrase), and
- a **table** form for the AC↔test map.

PRD §2/§8 both say "mapping each AC → the test" but drop **verbatim** and drop **table**. For a demo whose whole pitch is reviewability, the *verbatim* restatement is load-bearing (it's what lets a reviewer trust the map without cross-checking wording), and the table is the visual money-shot. Not fatal — §6 preamble says AC "are restated in the PR description mapped to their covering tests" — but the two crisp adjectives from the runsheet are gone. **Location:** runsheet L14 vs prd §2 bullet 1, §6 L57, §8 bullet 3.

### G2 — The demo's **accountability thesis** (named human = *more* accountability than today) is absent from Goal/Success (DILUTED) — Medium

The runsheet's closing line (step 5, L28–29) is the sales thesis: *"AI wrote most of this. A person owns it… a named human owning the merge is **more** accountability than today, not less."* The PRD §1 Context captures the safety framing ("AI-authored code is safe because a named human owns it"), and §2/§8 capture the *mechanics* (named approval, review trail). But the **"more accountability than today, not less"** comparative — the actual persuasive punch of the demo — is nowhere. This is arguably fine (it's rhetoric, not a requirement), so ranked Medium not High: a finalize reviewer should consciously decide whether the success criterion should encode *why* this matters, or leave it as pure artefact-checklist. **Location:** runsheet L27–29; no counterpart in prd §2/§8.

### G3 — Build order: **Azure Boards `AB#<id>` story link** is present but not a success criterion (PARTIAL) — Low/Medium

Build order step 2 (L38): "Create the AB# story in Azure Boards; link GitHub via `AB#<id>` so commits/PRs update the ticket." The PRD carries this in §7 (Cross-cutting: "The Azure Boards story links back via `AB#<id>`") — so it is **not dropped**. But it does not appear in §2 Goal or §8 Metrics. Given the runsheet frames "meet the client where they are; PRD-in-VC stays canonical" as a deliberate WoW proof point, consider whether the *demonstrable* Azure Boards ↔ GitHub linkage belongs in the success checklist, not just cross-cutting prose. Low-Medium because it is genuinely captured, just not elevated. **Location:** runsheet L38 vs prd §7 L66.

### G4 — Build order: **red-first / test-fails-before-impl** is implied but not asserted as success (DILUTED) — Low

Build order steps 3 & 5 (L39, L41) are explicit about the **ATDD red→green sequence**: write the red P0 test first (it *fails*, no impl), then AI implements the smallest change to green. The PRD says the test is "green" (§2, §8) and that AC are "the spec-as-contract source the ATDD Playwright tests are authored from" (§6) — so test-first is present in spirit. But the **red-phase-first** ordering (test demonstrably fails before implementation exists) — a core ATDD proof and part of the "not weakened to pass" story — is not stated as a success property. The §8 counter-metric "a test relaxed or an AC dropped to force green" guards the green side but not the red-first side. Low because ATDD-first is a constitution rule (§1 defers to `project-context.md`), so it is governed, just not surfaced. **Location:** runsheet L39/L41 vs prd §6 L57, §8 bullet 1.

### G5 — "AI wrote **most** of this" (authorship proportion) is not evidenced (DILUTED) — Low

The runsheet's closing (L28) and Part A framing (L9, "AI-authored-but-human-owned") lean on AI having authored *most* of the code. The PRD §1 says "AI-authored code is safe because a named human owns it" and §2 says "AI-authored-but-human-owned delivery." The *ownership* is well-evidenced (named approval); the *authorship-by-AI* claim has no artefact backing it in the success criteria — nothing in the PR proves AI wrote it. Likely immaterial for a spike (it's a narration point, not a checkable artefact), hence Low, but worth a conscious call at finalize. **Location:** runsheet L9, L28.

---

## Elements confirmed NOT dropped (checked, present)

- **Exactly-one-wins atomic guarantee** — §2 bullet 2, FR4, AC1, §7 (primary quality attribute). Strongly carried; arguably *strengthened* vs runsheet.
- **Two browser sessions / Playwright drives the race** — AC1 (§6), addendum test-design note. Present (an addition beyond the runsheet's level of detail).
- **Concurrency mechanism (exclusion constraint / optimistic locking)** — runsheet steps 1 & 3 name it; PRD correctly defers the *how* to `addendum.md` and keeps §2/FR4 to the observable contract. Correct placement, not a gap.
- **Single readable PR / small slice** — §2 (single PR), §7 (slices stay small), §8 bullet 2 + counter-metric. Strongly carried.
- **Scaffold emerges PRD-first, not scaffold-first** — build order steps 1 & 4; PRD §1 + addendum stack note. Carried in prose (correctly downstream/architectural, not a §2/§8 metric).
- **Spec-as-contract / drift = failure** — §7, §8 counter-metric. Carried.
- **Losing-planner UX contract** — runsheet steps 1 & 3 ("what the losing planner sees"); PRD FR5 + §9 + addendum. Present and *expanded* (409 + UI copy), so enriched not diluted.

---

## Bottom line

No money-shot artefact was **dropped** — all four are in both §2 and §8. Every gap is a **dilution of narrative specificity** (G1 verbatim/table, G2 accountability thesis, G5 authorship proportion) or an **un-elevated build-order detail** that is captured elsewhere in the PRD but absent from the success checklist (G3 Azure Boards link, G4 red-first ordering). A finalize reviewer's only real decisions: (1) whether §2's PR-description criterion should say *verbatim* + *table* (G1, the most concrete miss), and (2) whether the demo's accountability thesis (G2) deserves a home in the success criteria or stays as Context framing.
