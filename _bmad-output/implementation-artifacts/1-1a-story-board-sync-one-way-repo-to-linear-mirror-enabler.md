---
baseline_commit: 4406657ed0bff79ac0bf8cf07146c36275986219
---

# Story 1.1a: Story→board sync — one-way repo → Linear mirror (enabler)

Status: review
Linear: BST-7

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a delivery team,
I want a one-way sync (repo → Linear) that mirrors each BMAD story spec to a linked Linear ticket — runnable on demand — backfilling the existing backlog and never creating duplicates,
so that the board mirrors the canonical version-controlled stories from a single source of truth, without a parallel GitHub Issues copy. (NFR6)

_Depended on by: Story 1.1b (drift detection + reconcile-and-confirm builds on this one-way mirror). This story ships as its own small PR — it is a pipeline enabler, not part of the accumulating feature PR (epic §Story→PR mapping)._

## Acceptance Criteria

Reproduced verbatim from `_bmad-output/planning-artifacts/epics.md` § "Story 1.1a". These are the contract; do not paraphrase them into the PR (Story 1.2 requires ACs restated verbatim with an AC→evidence table).

**AC1 — Backfill / create**

> **Given** the sync runs against a backlog that predates it (e.g. the stories from this session, none yet on the board)
> **When** it runs for the first time
> **Then** it backfills — creating a linked Linear ticket for every story spec that has no ticket yet, including this story itself
> **And** the first run for this spike's backlog is performed manually, since the sync does not yet exist when these stories are authored.

**AC2 — Idempotent upsert**

> **Given** the sync has already run
> **When** it runs again with no repo changes
> **Then** it makes no changes — an idempotent upsert, never a duplicate ticket
> **And** a story whose spec changed has its ticket updated; an unchanged story is a no-op.

**AC3 — Crash-safe match key**

> **Given** a ticket may be created but the run interrupted before any repo-side key is recorded
> **When** the sync runs again
> **Then** it re-finds the existing ticket by the **repo story id stamped into the Linear ticket** (external id / marker) — the durable match key lives on the Linear side, so an interrupted run never produces a duplicate
> **And** the Linear-minted key cached in the repo is a convenience, not the source of matching.

**AC4 — One-way direction + key lifecycle**

> **Given** Linear mints the ticket key on creation
> **When** the key is recorded against the story spec in the repo
> **Then** it is written as a working-tree edit that rides the next human-approved PR (never a direct push, honouring agents-draft-humans-approve) — a linking identifier, not content flowing back
> **And** changes made only in Linear are not authoritative and never flow back into the repo.

## Tasks / Subtasks

- [x] **Task 1 — Scaffold the sync mechanism as a Claude Code skill** (AC: 1,2,3,4)
  - [x] Implement as a Claude Code skill/command `sync-stories-to-linear` under `.claude/skills/` (**decided** — Linear MCP-driven, no new secret; see Dev Notes "Implementation form"), invokable on demand. 1.1b will add the sprint-planning hook and the confirm gate on top of it.
  - [x] The skill enumerates the full backlog from `sprint-status.yaml` (**decided** — see Dev Notes "Enumerator"), resolves each story's Linear ticket by marker, and creates-or-updates. No GitHub Issues involved.
- [x] **Task 2 — Define the durable match key (marker) and matching** (AC: 3)
  - [x] Stamp the **repo story id** (the sprint-status key, e.g. `1-1a-story-board-sync-one-way-repo-to-linear-mirror-enabler`) into each Linear ticket at creation time, so the marker lands atomically with the ticket (crash-safe). Recommended: HTML-comment marker lines in the ticket description — `<!-- bmad-story-id: <story_key> -->` (the match key) and `<!-- bmad-content-hash: <hash> -->` (the idempotency key, Task 4).
  - [x] **Matching keys on `bmad-story-id` only** (never title, never the repo-side cached key): list issues in the `BST Spike` project and find the one whose description contains that id marker. The content-hash marker is used only to decide no-op-vs-update, never to match.
- [x] **Task 3 — Create / backfill** (AC: 1)
  - [x] For every story with no matching ticket, create one in team `BST` / project `BST Spike` with: title from the story's heading (H1 of the per-story file, or the `### Story X.Y:` heading when sourced from epics.md), description = marker (id + content-hash, Task 2) + canonical-repo-file link (`<story_key>.md` if it exists, else `epics.md#Story-X.Y`) + the story's user-story block + its ACs (a faithful mirror).
  - [x] Include this story (1.1a) itself in the backfill.
- [x] **Task 4 — Idempotent upsert (no-op via content hash)** (AC: 2)
  - [x] Decide no-op vs update by the **content hash in the marker**, NOT by diffing rendered prose (Linear re-renders markdown on save — whitespace/link/list normalization means a byte compare of `get_issue` output would falsely report "changed" every run and violate AC2). Compute a stable hash of the canonicalized source spec; if it equals the ticket's stored `bmad-content-hash` → **no-op (zero writes)**; if it differs → update the ticket body in place and rewrite the hash marker (same ticket, never a new one).
  - [x] Canonicalize the source before hashing (e.g. trim + collapse trailing whitespace) so the hash is deterministic across runs. Re-running with no repo changes must make zero writes.
- [x] **Task 5 — Repo-side key write-back (one-way link)** (AC: 4)
  - [x] After a ticket is created, write its Linear key back into the story file as a working-tree edit (the `Linear:` line in this header) — never a direct `git push`. It rides the next human-approved PR.
  - [x] Write the repo cache **only for stories that have a per-story `.md` file** (today just 1.1a). Fileless backfilled stories get no repo cache — that is fine, because matching is by the Linear-side marker, not the cache. Do not create story files just to hold a key.
  - [x] Treat the repo-side key as a cache only; nothing flows Linear → repo except this identifier.
- [ ] **Task 6 — Perform the manual first backfill run** (AC: 1) — _HUMAN-performed (AC1 + guardrail): the outward-facing Linear writes are left for Pete. See skill "How to run"._
  - [ ] Once the skill exists, run it manually against the current backlog (bootstrap: the sync did not exist when these stories were authored). Confirm it creates one mirror ticket per story, each carrying the marker + repo link.
- [ ] **Task 7 — Capture verification evidence** (verification — see Testing)
  - [ ] Record a manual-walkthrough evidence note (for the PR) proving: (a) backfill created N marker-stamped tickets incl. 1.1a; (b) a second run is a clean no-op (no duplicates); (c) editing one story spec updates only that ticket; (d) a simulated interrupted run (ticket created, key not yet written back) is re-found by marker on the next run — no duplicate.

## Dev Notes

### What this story is (and is NOT)

- **IS:** the one-way mirror mechanism — backfill + idempotent upsert + crash-safe marker matching + repo-side key write-back. Direction is strictly **repo → Linear**.
- **IS NOT:** drift detection, the change-preview, and the reconcile-and-confirm human gate — those are **Story 1.1b**, which depends on this. It also is **not** the sprint-planning hook (1.1b) and **not** a GitHub Issues sync (explicitly excluded). Do not build 1.1b's confirm gate here; keep this the raw upsert capability. [Source: epics.md#Story-1.1b; memory: spike-story-board-sync-oneway]
- **Boundary note on AC2 "update":** 1.1a performs updates directly (idempotent upsert). The human-confirm-before-overwrite safety is deliberately deferred to 1.1b. Keep updates faithful mirrors of the repo spec so that deferral is safe.

### Concrete Linear targets (verified 2026-07-06 via Linear MCP)

- **Team:** `BST` (id `2b81e3e8-53d4-4c8f-8b78-6280d1dfb0a7`).
- **Project:** `BST Spike` (id `504c2cf9-d132-4ec4-a183-b74c5f73eddd`), url `https://linear.app/bst-demo/project/bst-spike-f3dd8e86a0cc`. Its own description already states the repo is canonical and the board mirrors it.
- **Canonical repo:** `piotr-nearform/bst-demo` (GitHub remote). Note the local dir is `bst-spike` but the remote/board name is `bst-demo`.
- **Pre-existing tickets — READ THIS (out of scope, confirmed):** the board already holds `BST-5` ("Enabler E2 — Multi-model PR review gate") and `BST-6` ("Enabler E1 — Linear ↔ repo sync"). These are **legacy placeholders hand-created during Linear setup (2026-07-03), before the 1.1a/1.1b/1.2 split — they carry NO marker and are NOT per-story mirrors.** The backfill must NOT match or reuse them; it creates fresh marker-stamped mirror tickets. Retiring/relinking the legacy placeholders is **confirmed out of scope for 1.1a** (it is a 1.1b drift concern or a manual cleanup). Creating a fresh mirror ticket that overlaps a legacy placeholder's topic is expected and acceptable for this story; flag it for 1.1b.

### Implementation form (DECIDED: Claude Code skill, Linear MCP-driven)

Build **a Claude Code skill/command** `sync-stories-to-linear` under `.claude/skills/`, driven by the **Linear MCP tools** already connected in-session:
- `mcp__claude_ai_Linear__list_issues` (project=`BST Spike`) to fetch existing tickets for marker matching.
- `mcp__claude_ai_Linear__save_issue` to create and to update tickets (create when no id, update when id known).
- `mcp__claude_ai_Linear__get_issue` to read a single ticket's current content for the idempotency compare.

Rationale (decided 2026-07-06): both invocation paths (on-demand now, sprint-planning hook in 1.1b) run **in-session where MCP auth exists**; a skill needs **no new secret** (avoids the NFR3 surface a standalone script's `LINEAR_API_KEY` would add) and composes with the conductor/sprint-planning model. The deterministic contract below is what pins behaviour, even though execution is agent-driven — write the skill's steps as an explicit, order-fixed procedure (enumerate → match by marker → diff → create/update → write-back) so runs are repeatable.

A standalone Node script with a `LINEAR_API_KEY` was considered and **rejected** for this spike (adds a secret + a second Linear client for no benefit the in-session skill lacks).

### The deterministic contract (pins behaviour regardless of form)

1. **Marker (durable match key):** the repo story id, stamped into the Linear ticket **description** at creation. Recommended literal: `<!-- bmad-story-id: <story_key> -->` on its own line, alongside `<!-- bmad-content-hash: <hash> -->`. Both land in the same `save_issue` create call as the ticket → crash-safe (a ticket can never exist without its id marker). [Source: epics.md AC3; memory: spike-story-board-sync-oneway]
2. **Matching:** list issues in `BST Spike`, select the one whose description contains the `bmad-story-id` marker for the target story. Title/date/repo-cached-key are never the match source.
3. **Idempotency (hash-based, not prose-diff):** compare the canonicalized source spec's hash to the ticket's stored `bmad-content-hash`. Match → **no write**. Differ → **update the same ticket** (rewrite body + hash). No id-marker match → **create**. Do NOT decide by byte-diffing `get_issue` prose — Linear re-renders markdown, so a rendered compare would false-positive "changed" and break the zero-writes no-op. A no-change re-run performs zero writes and creates zero tickets.
4. **Repo-side key cache:** on create, write the Linear-minted key (e.g. `BST-12`) into the story file's `Linear:` header line as a **working-tree edit** (no direct push); it rides the next human-approved PR. Written **only for stories that have a per-story `.md` file** (fileless backfilled stories get no cache — matching is by the Linear-side marker). Cache only — matching never depends on it, and it may be absent/stale after an interrupted run without causing a duplicate.
5. **One-way:** nothing flows Linear → repo except the key identifier. Linear-side edits are non-authoritative.

### Enumerator: which stories get mirrored (DECIDED: full backlog from sprint-status)

Enumerate the story set from `_bmad-output/implementation-artifacts/sprint-status.yaml` → `development_status` keys, **excluding** `epic-*` and `*-retrospective` keys. These keys ARE the durable story ids (they match the marker), and this backfills the **whole** backlog (all of 1.1a…1.7) even though most have no per-story file yet — matching "backfilling the existing backlog." (Decided 2026-07-06: full sprint backlog, not just stories with per-story files.)

For each story's **content**, source from the per-story file `_bmad-output/implementation-artifacts/<story_key>.md` if it exists (richest), else fall back to the matching `### Story X.Y` section in `epics.md`. Both source and marker key off the same `<story_key>`.

### Story-file header convention (new)

The current story template (`.claude/skills/bmad-create-story/template.md`) has no field for the Linear key. This story introduces the `Linear:` line directly under `Status:` in the header (see the top of this file) as the repo-side cache. Keep it machine-parseable (one key per line, e.g. `Linear: BST-12`). Absent/comment value = not yet synced.

### Constitution & NFR guardrails that apply here

- **Agents draft; a human approves (rule 5).** The sync itself must never push to git or auto-merge. The key write-back is a working-tree edit on the next PR. No agent marks work done. [Source: project-context.md#Critical-Implementation-Rules]
- **No secrets in code or git (NFR3, rule 2).** If the script alternative is chosen, the Linear token is an env var only. The recommended skill/MCP form introduces no new secret.
- **Spec-as-contract (NFR5, rule 4).** This story's behaviour and this spec change together in the same PR.
- **Review-tooling carve-out.** Azure-only is waived for pipeline/review tooling only (Linear + GitHub) — this sync is such tooling. [memory: spike-review-tooling-carveout]

### Testing standards summary

Per the epic's **Delivery & verification conventions**: enablers (1.1a/1.1b/1.2) are **verified by manual walkthrough / CI dry-run + evidence, NOT Playwright** — a scoped, documented exception to NFR4 ATDD, not a waiver. There is no app scaffold yet (1.3 pulls that into existence), so there is no `tests/e2e` or `tests/api` target for this story. The verification for 1.1a is the Task-7 evidence note (backfill / no-op re-run / single-ticket update / crash-safe re-find), attached to the PR. [Source: epics.md#Delivery-&-verification-conventions]

### Project Structure Notes

- No app source (`ui/`, `api/`, `db/`) is touched — those belong to 1.3+. This story adds pipeline tooling only: the sync skill/command (recommended `.claude/skills/sync-stories-to-linear/`) plus this story file and its `Linear:` write-back.
- The structural seed in the architecture spine (`ui/ api/ db/ tests/`) is app-only and irrelevant to this enabler.

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story-1.1a] — ACs (verbatim), one-way direction, crash-safe marker, key lifecycle.
- [Source: _bmad-output/planning-artifacts/epics.md#Delivery-&-verification-conventions] — enabler PR mapping + manual-walkthrough verification exception.
- [Source: _bmad-output/project-context.md] — Board = Linear (2026-07-03); one-way link via Linear key; constitution rules 2 (secrets), 4 (spec-as-contract), 5 (agents draft / humans approve).
- [Source: docs/wiki/topics/ways-of-working.md] — conductor model; on-demand/in-session invocation; headless/cron cannot use interactively-authenticated MCP (why the skill runs in-session).
- [Source: git 0092afe] — the 1.1→1.1a/1.1b split and the crash-safe-match-key / one-way clarifications.
- Memories: [[spike-story-board-sync-oneway]], [[spike-phase-zero-enablers]], [[spike-review-tooling-carveout]], [[gh-pr-account]] (PRs on bst-demo need `gh auth switch --user piotr-nearform`).

## Decisions (resolved 2026-07-06)

1. **Implementation form:** a Claude Code skill/command `sync-stories-to-linear`, driven by the Linear MCP tools — no new secret, composes with the sprint-planning hook. (Standalone script rejected.)
2. **Mirror set:** the **full sprint backlog** enumerated from `sprint-status.yaml` (all of 1.1a…1.7), content sourced from per-story file when present else the `epics.md` section.
3. **Legacy tickets `BST-5`/`BST-6`:** **out of scope** for 1.1a — backfill creates fresh marker-stamped mirrors; reconciling/retiring the placeholders is 1.1b or a manual cleanup.

## Dev Agent Record

### Agent Model Used

Claude Opus 4.8 (1M context) — `claude-opus-4-8[1m]` (bmad-dev-story workflow).

### Debug Log References

- Read-only dry-run validated the deterministic contract without any Linear write: enumerating `sprint-status.yaml` `development_status` (excluding `epic-*` / `*-retrospective`) yields 8 stories (1.1a, 1.1b, 1.2, 1.3–1.7); the `X.Y` derivation from the story-key prefix matches the `### Story X.Y` headings in `epics.md` exactly; only 1.1a has a per-story file (so only 1.1a gets a repo-side key write-back).

### Completion Notes List

- **Built the skill `sync-stories-to-linear`** at `.claude/skills/sync-stories-to-linear/SKILL.md` — an agent-executed, order-fixed procedure (enumerate → source content → hash → match by marker → decide create/update/no-op → apply → repo-side write-back) implementing the story's deterministic contract and Tasks 1–5. Linear MCP-driven (`list_issues`/`get_issue`/`save_issue`), no new secret (NFR3).
- **Match key** = `<!-- bmad-story-id: <story_key> -->` in the ticket description, stamped atomically at creation (crash-safe, AC3). Matching is on that marker only — never title, date, or the repo-side cached key. Legacy `BST-5`/`BST-6` carry no marker and are explicitly out of scope (never matched/retired here).
- **Idempotency** = `<!-- bmad-content-hash: <sha256> -->`; hash computed with a real tool over the canonicalized (title + user-story + ACs) source, NOT by diffing Linear's re-rendered prose. Equal → zero-write no-op (AC2); differ → in-place update of the same ticket; no match → create.
- **One-way + key lifecycle (AC4):** on create, the minted key is written to the story file's `Linear:` header as a working-tree edit only (no push/PR/merge by the agent) — and only for stories that have a per-story `.md` file. The `Linear:` header convention is already seeded in this file.
- **Guardrails honoured:** no real Linear tickets created/updated/deleted (Task 6 backfill left for the human per AC1); no `git push`/PR/merge; story not marked `done`; no secrets introduced.
- **Deferred to the human (outward-facing):** Task 6 (manual first backfill) and Task 7 (evidence capture: backfill / no-op re-run / single-ticket update / crash-safe re-find). Exact steps + evidence checklist are documented in the skill's "How to run" section. The first backfill has since been run manually (Task 6): it minted tickets BST-7…BST-14 and wrote back `Linear: BST-7` for this story; all four Task-7 evidence checks passed (recorded on PR #6).
- **Verification method:** per the epic's Delivery & verification conventions, enablers are verified by manual walkthrough + evidence, not Playwright — there is no app scaffold yet. The read-only dry-run above confirms the deterministic (repo-side) half of the contract.

### File List

- `.claude/skills/sync-stories-to-linear/SKILL.md` (new) — the sync skill.
- `_bmad-output/implementation-artifacts/1-1a-story-board-sync-one-way-repo-to-linear-mirror-enabler.md` (modified) — baseline_commit frontmatter, task checkboxes, Dev Agent Record.
- `_bmad-output/implementation-artifacts/sprint-status.yaml` (modified) — 1.1a status ready-for-dev → in-progress → review.
