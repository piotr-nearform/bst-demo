# Brainstorm Intent: BST Scheduling Spike

## Goal / Why

The spike's real job is to **establish ways of working that KPMG (client) and Nearform agree on**, reusable to build the actual BST project after Discovery. We are currently pre-Discovery. The app is disposable; the process is the point.

The load-bearing thing to prove: **AI-authored code is safe**, via a visible control chain:

- A human owns the final PR approval and is accountable for it — net *more* accountability than today, not less. AI is tooling that streamlines, not replaces, the decision.
- The PRD lives in version control, so product decisions become traceable (what was decided, when, and where it was captured).
- Automated multi-model review supports the human reviewer (who now has more to review, not less).

This directly answers the hottest anticipated KPMG pushback: limited trust in AI building the app.

## Chosen Feature (the one feature)

**The double-booking guard.** A planner assigns a person to a job over a date range; the system refuses any assignment that clashes with an existing one.

**In scope — only the concurrent-clash edge:** two planners booking the same person at the same time mid-flight. This must be handled at the right layer (e.g. a DB exclusion constraint or optimistic locking) — **not** a naive read-then-check. This non-trivial concurrency case is precisely what lets the spike showcase *how we work* rather than trivial CRUD.

**Out of scope (explicitly):** partial-overlap edges, touching-boundary edges, skills-matching, availability calendar, UI polish.

**Target:** wrap the whole spike in 1–2 days.

## Constraints (project constitution)

- **Azure-only**, with **UK data residency**.
- **React** front end, **Node** back end — both *indicative*, not confirmed. (KPMG may push for Python and/or full Azure suite / Azure DevOps over GitHub.)
- **Playwright acceptance tests written first** from the AC.
- **GitHub + Azure Boards**, linked via `AB#<id>`.
- **ATDD test-first** is non-negotiable.
- **Spec-as-contract**: code and PRD/story stay in sync; drift is a failure.
- **Agents draft, humans approve.**

## The 4 Keystone Insights

1. **Engineering hygiene = the sales pitch.** A readable, test-first PR is simultaneously the dev practice and the KPMG trust demo — one artifact serving both. The test evidence *is* the product; it must read well. The reviewer focuses on tests, skims implementation, watches sensitive files (package.json, infra, deps).
2. **The concurrent-clash edge forces the PO↔engineer spec negotiation.** Choosing this feature and choosing to showcase multi-role ways of working are the *same decision* — the edge case is what makes the spec conversation real.
3. **Code-as-truth vs humans'-tools tension dissolves.** PRD-in-version-control is canonical; Azure Boards (or Jira) is the human's view; the `AB#` link keeps the two in sync for free. Meet the client where they are without giving up the source of truth.
4. **Small vertical slices is the keystone lever, paying three times:** accountability (a human can genuinely read and own the PR), velocity (review stays cheap, so it doesn't eat the AI speed gains), and reviewability. (Caveat: AI tools that generate large changes may pressure this — keep slices small deliberately.)

## Demo End-State (success criterion)

A **single human-approved pull request** that proves AI-authored-but-human-owned:

- A compact PR description restating the AC and mapping which tests cover each AC.
- A green **P0 Playwright test** firing two concurrent, clashing assignments where exactly one wins.
- The linked **story + PRD decision in version control**, with author and timestamp.
- The **multi-model review** trail plus a **named human approval**.

## Step One

Write the **first PRD line (PRD-first)**, and let the BMAD pipeline pull the React/Node/Azure scaffold into existence as the first story demands it — **not scaffold-first**.
