---
project_name: 'BST spike (staff-to-job scheduling)'
user_name: 'Piotr'
date: '2026-06-29'
sections_completed: ['technology_stack', 'critical_rules']
existing_patterns_found: 0
---

# Project Context for AI Agents

_Critical rules every AI agent must follow when implementing code here. Focus on unobvious things an agent would otherwise get wrong. This file is the project constitution; `CLAUDE.md` / `AGENTS.md` hold operational detail (commands, architecture) and must not restate these rules._

_This is a throwaway spike to rehearse the BST ways of working. The app is disposable; the pipeline is the point. Prefer the smallest thing that works; no speculative abstraction._

---

## Technology Stack & Versions

Mark the confirmed-vs-indicative split and keep it; do not let indicative harden into fact.

- **Hosting: Azure only.** Confirmed hard constraint. No other cloud.
- **Frontend: React.** Indicative (expected BST direction, not formally confirmed). Version: TBD once scaffolded.
- **Backend: Node.** Indicative. Version: TBD once scaffolded.
- **Acceptance/E2E tests: Playwright.** Authored from story AC.
- **Unit tests / lint / type-check:** TBD once scaffolded; record exact commands in `CLAUDE.md` when they exist.
- **Code, CI, code review: GitHub** (Actions; GitHub Agentic Workflows for multi-model PR review).
- **Board: Linear** (spike decision, 2026-07-03), linked to GitHub via the Linear issue key in the branch/PR/story so work items stay in sync. _(Supersedes the earlier Azure Boards `AB#<id>` intent for the spike — chosen for the easiest MCP-driven board sync; the real engagement may revert to Azure Boards / an Atlassian tool.)_

## Critical Implementation Rules

1. **Azure only, UK data residency.** Anything hosted or stored targets Azure in a UK region. Never introduce another cloud or a non-UK region.
2. **No secrets in code or git.** Use environment variables / Azure Key Vault. Validate and sanitise all external input.
3. **ATDD is non-negotiable and test-first.** Playwright acceptance tests are written from the story AC *before* implementation (`bmad-dev-story`). Never weaken, skip, or rewrite a test to make it pass; a red AC test means the code is wrong, not the test.
4. **Spec is the contract.** If behaviour changes, update the corresponding BMAD story/PRD in the same change. Code that drifts from its story is treated like a failing test.
5. **Agents draft; a human owns and approves.** No agent marks its own work "done". Every artifact and code change lands via a PR a human approves.
6. **FRs are append-only; never silently re-meaned.** Functional requirements carry stable IDs (`FR1`, `FR2`, …). Never reword the meaning of a shipped FR in place, and never renumber or reuse an ID. A *changed* requirement gets a **new** FR ID (or a dated addendum entry that references the original); a *removed/superseded* one is marked `[DEPRECATED → superseded by FRn]` with its text preserved, never deleted. This is what keeps the FR Coverage Map trustworthy across an evolving PRD — BMAD does not enforce it (see `../docs/wiki/topics/iterating-with-bmad.md`).
7. **Acceptance criteria freeze at `in-progress`.** A story's AC is editable only while it is `backlog`. The moment it moves to `in-progress`, three things happen together — `bmad-create-story` writes the full story file, red ATDD tests are authored from the AC (rule 3), and the **AC freezes**. After that, scope change is a *new delta story*, or a deliberate reset-to-`backlog` via `bmad-correct-course` **with the tests re-derived** — never an in-place AC rewrite. Never edit a `done` story's AC (it is the historical record of what was built).
8. **Honour these constraints in every workflow.** They apply across PRD, architecture, story, dev, and review. If a constraint blocks the obvious approach, surface it, do not silently work around it.
