# Roles & Handoffs — BST Spike Ways of Working

_Living document. This is the real deliverable of the spike: the agreed choreography of **who comes in when, what they do, and where the human approval gates are**. The app is just what we run it on. Corrected by contact with reality as we run each stage — see "Log as we go" at the bottom._

Status: **draft hypothesis** (pre-Discovery, 2026-07-01). Not yet agreed with KPMG.

---

## The core principle: one driver, two models

**One driver.** The **Technical Lead (TL)** drives every BMAD session — runs the agents, produces the artifacts. The other humans are **reviewers/approvers at gates**, not drivers. (BMAD agent personas are tools the TL operates, not human seats — "we don't have a test architect" just means that seat is reviewed by the TL, not that the capability is missing.)

**Two models** — the only thing that changes per stage is *how the reviewer engages*:

- **Collaborative** — the reviewer (usually the KPMG PO) is involved **during** creation, not just at the end. Two concrete mechanisms: **(a) PRD-as-a-reviewed-PR** — the draft is committed and the PO reviews/comments on the diff as it grows (async, self-documenting, *same review gate as code*); or **(b) live pairing** on the session (synchronous, higher bandwidth, needs the PO's live time). Either way they shape it as it forms, so acceptance is co-authorship, not a rubber stamp.
- **Draft-then-review** — the TL (solo or with the team) produces the artifact, then the reviewer reviews and accepts after.

---

## The choreography

**Reviewer vs accepter:** the **accepter** owns the gate (the go/no-go); **reviewers** give input but don't hold the gate.

| Stage | Model | Accepts (gate) | Also reviews |
|---|---|---|---|
| Analysis / brainstorm | solo (internal) | Nearform internal | — |
| **PRD** | **Collaborative** | **KPMG PO** | Simon (KPMG Tech Lead) |
| **UX / design** _(if a UI is in play)_ | Draft-then-review _(ingest the designer's work — see Notes)_ | **Antoine (UI/UX designer)** for design fidelity | KPMG PO (experience fit) · Simon |
| Architecture | Draft-then-review | **Nearform Technical Director** | Simon (KPMG Tech Lead) |
| **Epics & story list** | **Collaborative** | **KPMG PO** | Simon (KPMG Tech Lead) |
| **Dev cycle — per story** <br>_(create-story → ATDD → dev-story → review)_ | solo | Nearform peer review _(details TBD)_ | — |

**Order correction (instance #2, 2026-07-02):** Architecture now sits **before** Epics & story list — matching BMAD (`bmad-create-epics-and-stories` is `preceded-by: bmad-architecture`) and dependency reality: stories must be written against the architecture decisions (e.g. the concurrency mechanism the PRD deferred). The earlier draft had epics before architecture; that ordering was wrong.

Two altitudes:
- **Planning (client-facing):** PRD + UX + epic/story list → the client-owned artifacts, PO (and designer) involved during creation.
- **Execution (internal):** the **dev cycle** is one repeatable module, run once per story — BMAD's create-story → ATDD → dev-story → review. Solo (TL drives) with **peer review**, which is where code-level accountability lives (a named human approves and owns the merge). We'll detail the internal gates of this module **later**.

Notes:
- **Architecture is owned by Nearform** (TD accepts) but **reviewed by KPMG's Tech Lead (Simon)** — client-visible, not a client gate. The PO shouldn't gatekeep a technical design decision, but the client TL gets to see and comment on it.
- **UX / design ingestion.** If Antoine (or any external designer) supplies designs — Figma, mockups, a design system — the design step is **`bmad-ux`**, run by the TL to *ingest and structure* his work rather than invent UX from scratch. It distills his artifacts into the BMAD **UX spine**: `DESIGN.md` (visual identity + design tokens) and `EXPERIENCE.md` (information architecture, states, interactions, accessibility, journeys), under `planning_artifacts/ux-designs/`. That spine is the **canonical, version-controlled contract**; Antoine's Figma stays the human source it links back to (the PRD-in-VC vs Azure-Boards pattern again). Downstream, `bmad-architecture` consumes it ("plus UX if present") and `bmad-create-epics-and-stories` extracts **UX-DR** requirements (tokens, each named component, a11y) — each becoming a story with testable AC. Implementation fidelity back to Antoine's design is checkable during dev with design-diff review agents. Antoine's involvement in the *spike* is **unconfirmed**; this row is a hypothesis for the real engagement.
- The standalone ATDD row folded **into** the dev cycle — it's a step of each cycle, not a separate stage. (Split it back out if you'd rather track it independently.)

---

## The review gate (breadth pass + named human)

Where code-level trust is proven — the review step of the dev cycle. Two components, in order:

1. **Automated multi-model breadth pass.** A **GitHub Agentic Workflow** runs **dual-model** review on the PR — breadth a single human can't match in the available time, captured on the PR as a review trail. *(Which two models, and exactly what they inspect, is CI config — firmed up when the pipeline is scaffolded. Not a PRD concern.)*
2. **Named human approval.** A person reads the **tests** closely, **skims** the implementation, and **watches sensitive files** (dependencies, `package.json`, infra). They own the merge and the accountability — more than today, not less. No agent marks its own work done.

The PRD references only the *outcome* of this gate (a review trail + a named approval on the PR), never this mechanism — the mechanism lives here and, later, in `.github/workflows/`. It is **out of the architecture spine by design** (it's CI/delivery infra, not feature structure); the spine carries only the "delivery discipline" convention and defers here.

**Spike model choice (2026-07-03).** The breadth pass uses **GitHub's out-of-the-box models** (whatever GitHub Agentic Workflows provides by default) — good enough to demonstrate the gate. This **carves the Azure-only / UK-residency constraint out of the review *tooling*** for the spike only; the real engagement must revisit whether review models have to be Azure-hosted (e.g. Azure OpenAI / AI Foundry, UK). The carve-out applies to the *review tool*, not to the app.

**This gate is a build task, sequenced first.** Standing up the GitHub Agentic Workflow is an **enabler story / tech task that precedes all feature stories** — the feature's own dev-story PR must flow *through* this gate to prove the ways-of-working. It is authored in `bmad-create-epics-and-stories` as its own item (not a story under the double-booking feature) and scaffolded via `bmad-testarch-ci`.

---

## Iterating the PRD — the change cycle (after the first pass)

The choreography above runs **once** to stand the product up. After that the product evolves by **iterating the PRD**, not rebuilding it. This is where the real risk lives: BMAD has **no native guard** against a moving PRD silently orphaning already-built work (research 2026-07-06 — confirmed open BMAD issues [#1930] `correct-course` rewrites the AC of already-merged stories, [#1638] epic/story decomposition is strictly one-way with no feedback path, [#446] stories go stale with no detection). So we impose the discipline below. It is the append-an-epic / living-PRD model BMAD recommends, hardened with three conventions every serious spec-driven framework converges on (append-only requirement IDs, deltas-not-edits, immutable shipped work).

**The cycle is two PRs, never collapsed into one:**

**PR-A — Planning change** (`bmad-prd` update → `bmad-create-epics-and-stories`). Touches `prd.md` + `epics.md` only; **no application code**. New scope enters as `planned` FRs (a legitimate backlog line, not a defect — this is "outstanding"). `bmad-create-epics-and-stories` regenerates the FR Coverage Map so nothing is left uncovered by accident.

- **Gates on PR-A** (all planning-file checks — there is no code here, so "spec+code together" does *not* apply):
  1. **Coverage complete** — every `FRn` in `prd.md` appears in the coverage map in `epics.md` (i.e. epics were regenerated, not left stale). Checked with **`bmad-check-implementation-readiness`** — an **LLM-judgement step in our process, not a hard GitHub status check** (it has no deterministic pass/fail). A note in this doc + a reviewer eyeball, not CI.
  2. **FR IDs append-only** — diff check on `prd.md`: an existing FR's text changed ⇒ fail, unless it's a new `FR<n+1>` or the old ID is marked `[DEPRECATED]` (constitution rule 6). Mechanical; could be trivial CI later, a process step for now.
  3. **No frozen story touched** — no story that is `in-progress`/`done` in `sprint-status.yaml` has its AC modified in this diff (the [#1930] guard; constitution rule 7). Mechanical; process step for now.

**Ticket creation happens on merge of PR-A, and is manual by design.** The TL runs **`sync-stories-to-linear`** (Story 1.1a skill) to mirror the new `planned` stories to Linear. We deliberately do **not** auto-run this from a GitHub Agentic Workflow on merge: (a) it would bypass the **reconcile-and-confirm human gate** we built into Story 1.1b (agents-draft-humans-approve — the board is never written silently), and (b) an on-merge Action is post-merge and non-interactive, so the "do we decompose these now or not?" decision has nowhere to live — a manual invocation *is* that decision point. A path-filtered Action that merely **comments "PRD changed — run `sync-stories-to-linear`"** is an acceptable nudge; anything that writes the board is not.

**PR-B — Implementation** (`bmad-create-story` → ATDD → `bmad-dev-story` → review). This is the existing **dev-cycle** row. `bmad-create-story` runs **just-in-time** — when a dev picks the story up and moves it to `in-progress`, *not* at PR-A time. Touches code + the story file; gate is "spec+code together" (story file in the diff) flowing through the review gate above.

**The freeze line — one boundary, three things at once.** An AC is editable only while the story is `backlog`. At move-to-`in-progress`, `bmad-create-story` writes the full file, the red ATDD tests are authored from the AC, **and the AC freezes** (constitution rule 7). This single transition is why "which deltas are done vs outstanding" stays answerable: `planned` FR = outstanding, `in-progress`/`done` FR = frozen and traceable to a story + tests.

**Ownership of the iteration cycle:**

| Step | Model | Accepts (gate) | Also reviews |
|---|---|---|---|
| **PR-A — PRD iteration** (`bmad-prd` update → `bmad-create-epics-and-stories`) | **Collaborative** | **KPMG PO** _(same gate as the PRD row)_ | Simon (KPMG Tech Lead) |
| **Ticket sync** (`sync-stories-to-linear`, on PR-A merge) | solo | **Nearform TL** _(runs + confirms the reconcile gate)_ | — |
| **PR-B — per-story implementation** | solo | Nearform peer review _(the review gate above)_ | — |

---

## Caveat to carry into the real engagement

Collaborative mode needs the KPMG PO's **synchronous time**. Senior client stakeholders often won't give live hours, which may force PRD / stories into **draft-then-review** even though collaborative is better for ownership. Rule: **collaborative if the PO can sit with us; fall back to draft-then-review if not.** (In the spike this is free — the TL plays both hats.)

---

## Log as we go

Fill in what *actually* happened at each stage — where the PO really needed to come in, what their duties turned out to be, where the hypothesis above was wrong.

- **PRD — instance #1 (2026-07-01).** Ran `bmad-prd` collaboratively via **mechanism (a) — PRD-as-a-reviewed-PR** (PR #1). What actually happened:
  - **Collaborative-via-PR worked.** Draft committed → PO took the three open-question decisions → folded back as commits → a PR comment records the decision trail. The loop (PO comment → TL commit) is real and self-documenting. Pete wore TL + PO (free in the spike).
  - **"Agents draft, humans approve" fell out *structurally*, not just as a rule.** The agent literally could not open the PR — `gh` was authed as read-only accounts; a write-capable human identity (`piotr-nearform`) was required. Lesson for the real engagement: provision a write-scoped token/identity for the agent, and keep the agent's PR-opening identity **distinct** from the human who approves/merges, so the approval gate stays a genuine separation rather than one identity rubber-stamping itself.
  - **PRD-first held.** No scaffold yet; the first story pulls React/Node/DB into existence. Q3 (UI in scope) means that first story also scaffolds a front end.
  - **Hypothesis mostly right, one part untested.** Collaborative mode was cheap here only because one person wore both hats; the caveat below (it needs the PO's synchronous time) stays unproven until a real KPMG PO is in the loop.
- **Iteration process — instance #3 (2026-07-06).** Before running any PRD change, worked out *how* we iterate the PRD without orphaning built work — the stakeholder question "how does the agent know which PRD deltas are done vs outstanding, and how do we not silently override old functionality?". Ran deep research across BMAD's repo/issues + adjacent spec-driven frameworks (OpenSpec, ReqToCode, Kiro, Spec Growth Engine): **BMAD has no native answer** — decomposition is one-way ([#1638]), `correct-course` can rewrite merged stories' AC ([#1930]), stories go stale undetected ([#446]). Corrected an earlier wrong belief that `bmad-testarch-trace` guards this (it's *test*-coverage traceability, not change-preservation — refuted in the research). Added the two-PR change cycle + three PR-A gates + the freeze line above, and encoded the two load-bearing rules (append-only FR IDs, AC-frozen-at-`in-progress`) into `project-context.md` (rules 6, 7). Ticket creation stays **manual** (`sync-stories-to-linear`) on purpose — auto-on-merge would bypass Story 1.1b's reconcile-and-confirm gate. Lesson: the iteration guardrails are *our* convention layered on BMAD, not something BMAD gives us; the freeze line (`backlog` → `in-progress`) is the single boundary that makes done-vs-outstanding answerable.
- **Sequencing — instance #2 (2026-07-02).** Caught before running epics: the draft choreography had **Epics & stories before Architecture**, but BMAD (`bmad-create-epics-and-stories` is `preceded-by: bmad-architecture`) and dependency reality put **Architecture first** — the guard story can't be written cleanly until the concurrency mechanism the PRD deferred is decided. Corrected the table order. Lesson: trust the BMAD phase ordering over a hand-drawn choreography unless there's a concrete reason to diverge; a home-grown sequence is easy to get subtly wrong. Also surfaced the **UX/design step** (`bmad-ux`) as the ingestion point for Antoine's designs — added to the table and Notes.
