# Input Reconciliation — brainstorm-intent.md → PRD + Addendum

**Source:** `brainstorming/.../brainstorm-intent.md`
**Deliverable checked:** `prd.md` + `addendum.md`
**Date:** 2026-07-01

## Verdict

The PRD+addendum capture the source's *functional* content faithfully — the double-booking guard, the concurrency edge, the demo end-state, the scope cuts, and all four Keystone Insights have a home. What thins out is the source's **qualitative center of gravity**: the spike exists to *sell a way of working to a skeptical client*, and several framing/intent threads that carry that persuasion load are either demoted to sub-clauses or dropped. A functional-requirements structure has done what it always does — preserved the *what*, diluted the *why it matters to whom*.

Gaps are ranked by importance below.

---

## Gap 1 (High) — The "AI-safety trust demo aimed at KPMG pushback" is the source's spine; the PRD keeps the mechanism but loses the audience and the argument

**Source:** §Goal is emphatic and repeated — the spike proves "**AI-authored code is safe**" and "This directly answers the hottest anticipated KPMG pushback: **limited trust in AI building the app**" (lines 7, 13). It frames the human owner as "net *more* accountability than today, not less. AI is tooling that streamlines, not replaces, the decision" (line 9).

**Deliverable:** PRD §1 keeps "AI-authored code is safe because a named human owns it" but drops the audience and the adversarial framing. The phrase "hottest anticipated KPMG pushback / limited trust in AI" appears **nowhere** in prd.md or addendum.md. The "net *more* accountability, not less" argument — the actual rhetorical payload — is gone.

**Why it matters:** This is the reason the spike exists. Without it, a downstream reader treats this as a small scheduling feature with good test hygiene, not as a **client-trust artifact with a persuasion goal**. The "more accountability, not less" line is the answer to the objection; losing it strips the spike of its point.

## Gap 2 (High) — "Engineering hygiene = the sales pitch" (Keystone 1) is present but softened; the "test evidence *is* the product" claim and the reviewer's specific behaviour are diluted

**Source:** Keystone 1 (line 37): "A readable, test-first PR is simultaneously the dev practice and the KPMG trust demo — one artifact serving both. The **test evidence *is* the product**; it must read well. The reviewer focuses on tests, skims implementation, **watches sensitive files (package.json, infra, deps)**."

**Deliverable:** PRD §7 captures "reviewability is the sales pitch" and "test evidence is written to read well" — good. But the concrete reviewer *behaviour* — focus on tests, skim implementation, **watch sensitive files (package.json, infra, deps)** — is entirely absent from both files. This is a specific, actionable review posture that the source deliberately articulated.

**Why it matters:** "Watch sensitive files" is a security/trust signal that directly serves Gap 1's audience (a KPMG reviewer worried about what AI might slip into deps/infra). It is exactly the kind of non-functional nuance an FR/AC structure sheds. It belongs somewhere downstream (review checklist / demo runsheet) or it will not happen.

## Gap 3 (Medium) — Keystone 4's "pays three times" framing is flattened to a single constraint, and its caveat is dropped

**Source:** Keystone 4 (line 40): small vertical slices is "the keystone lever, paying three times: **accountability, velocity** (review stays cheap, so it doesn't eat the AI speed gains), and **reviewability**." Explicit caveat: "AI tools that generate large changes may pressure this — keep slices small deliberately."

**Deliverable:** PRD §7 ("Slices stay small") and §8 (counter-metric on slice bloat) capture reviewability and accountability well. But the **velocity** leg — "review stays cheap so it doesn't eat the AI speed gains" — is missing, and the **caveat** (AI tools pressure slice size; resist deliberately) is absent from both files.

**Why it matters:** The velocity argument is half of why small slices matter commercially (AI is fast; if review is expensive you lose the gain). The caveat is a live risk in an AI-authored pipeline and a natural counter-metric — dropping it removes a guardrail the source consciously flagged.

## Gap 4 (Medium) — Keystone 3's "tension *dissolves*" framing is reduced to a mechanical sync rule

**Source:** Keystone 3 (line 39): the code-as-truth vs humans'-tools tension "**dissolves**" — "Meet the client where they are without giving up the source of truth." It also names **Jira** as an alternative to Azure Boards ("Azure Boards (or Jira)").

**Deliverable:** PRD §7 states the `AB#` link mechanism and PRD-as-canonical correctly, but frames it as a sync procedure, not as the *resolution of a philosophical tension* — the "meet the client where they are" diplomacy is lost. "Jira" as an alternative is dropped (only Azure Boards named).

**Why it matters:** The "meet the client where they are" framing is another client-relationship signal (flexibility on their tooling, firmness on the source of truth). It reassures a client who may not adopt Azure Boards. The Jira mention preserves optionality the source deliberately kept open.

## Gap 5 (Medium) — "PRD-first, not scaffold-first" is preserved but its *ordering discipline* as a demonstrable practice is understated

**Source:** §Step One (lines 51–53): "Write the **first PRD line (PRD-first)**, and let the BMAD pipeline **pull the scaffold into existence as the first story demands it — not scaffold-first**." This is stated as *the* first move and a discipline to be shown.

**Deliverable:** The addendum stack note preserves "PRD-first, not scaffold-first" well (line 26). But it lives only in the stack note as a deferral mechanism; the PRD body never elevates "PRD-first" as a *demonstrated way of working* / success signal in its own right.

**Why it matters:** Minor, because the phrase survives. But the source treats PRD-first as a visible practice the demo proves, not just a scheduling convenience for stack choice. Slight loss of emphasis.

## Gap 6 (Low) — Keystone 2 is honoured; noted for completeness

**Source:** Keystone 2 (line 38): choosing the concurrent-clash edge and choosing to showcase multi-role ways of working "are the *same decision* — the edge case is what makes the spec conversation real."

**Deliverable:** Well preserved. PRD §1, §9 ("the PO spec negotiation") and the three resolved decisions directly realise this — the edge case is explicitly what forced the PO↔engineer conversation. **No gap.** The only micro-nuance: the source frames it as "the *same decision*," which the PRD renders as cause-and-effect rather than identity — negligible.

---

## Intent/emphasis losses not tied to a single Keystone

- **"The app is disposable; the process is the point."** Present in PRD §1 ("app is disposable; the pipeline is the deliverable") — preserved.
- **"Net more accountability than today, not less."** (line 9) — **dropped** (see Gap 1). The single most quotable trust line in the source.
- **Concurrency mechanism as trust-relevant, not just correct.** The addendum treats Option A/B as an architecture decision (correctly). But the source's framing — that choosing the *right layer* is "precisely what lets the spike showcase *how we work* rather than trivial CRUD" (line 19) — is a *demonstration* argument, not just an engineering one. The addendum keeps the "not a naive read-then-check" non-negotiable but loses the "this is what makes it a showcase" reason. Minor.

## Summary of where dropped content should land (not a rewrite proposal, just placement)

- Gaps 1, 3, 4: PRD §1 / §2 framing and §8 counter-metrics — these are *intent* losses, best restored in prose, not FRs.
- Gap 2 (watch sensitive files): review checklist / demo runsheet, downstream.
- Gap 5: minor emphasis; optional.
