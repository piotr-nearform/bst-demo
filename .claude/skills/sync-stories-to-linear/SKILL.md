---
name: sync-stories-to-linear
description: 'One-way repo → Linear mirror of BMAD story specs. Backfills the backlog and idempotently upserts each story to a linked Linear ticket, never creating duplicates. Use when the user says "sync stories to Linear", "mirror the backlog to the board", or "run the story→board sync".'
---

# Sync Stories to Linear Workflow

**Goal:** Mirror every BMAD story spec in this repo to a linked Linear ticket in the `BST Spike` project — a strictly one-way sync (repo → Linear). Backfill any story with no ticket, update a ticket whose spec changed, and no-op an unchanged story. Never create a duplicate.

**Your Role:** You are the sync mechanism. You execute the deterministic contract below as an explicit, order-fixed procedure so runs are repeatable. You draft; a human owns and approves — you never `git push`, never merge, and you write the repo-side key only as a working-tree edit.

## Scope (what this skill IS and IS NOT)

- **IS:** the one-way mirror — enumerate backlog → match by marker → hash-compare → create/update → repo-side key write-back. Direction is strictly **repo → Linear**.
- **IS NOT:** drift detection, change-preview, or the reconcile-and-confirm human gate — those are **Story 1.1b** and build on top of this. Do not add a confirm gate here beyond the create/update safety already specified. There is **no** GitHub Issues sync (explicitly excluded).
- Source of truth: the BMAD story spec in version control. Linear is a downstream mirror; edits made only in Linear are non-authoritative and never flow back to the repo (except the ticket key, cached repo-side).

## Conventions

- Bare paths resolve from the project working directory (`{project-root}`).
- `story_key` = the `development_status` key in `sprint-status.yaml` (e.g. `1-1a-story-board-sync-one-way-repo-to-linear-mirror-enabler`). This IS the durable story id and the match marker value.

## Fixed targets (verified 2026-07-06 via Linear MCP)

- **Team:** `BST` — id `2b81e3e8-53d4-4c8f-8b78-6280d1dfb0a7`
- **Project:** `BST Spike` — id `504c2cf9-d132-4ec4-a183-b74c5f73eddd` (url `https://linear.app/bst-demo/project/bst-spike-f3dd8e86a0cc`)
- **Canonical repo (for links):** `piotr-nearform/bst-demo` on GitHub. Note the local dir is `bst-spike` but the remote/board name is `bst-demo`.
- **Sprint status file:** `_bmad-output/implementation-artifacts/sprint-status.yaml`
- **Per-story files:** `_bmad-output/implementation-artifacts/<story_key>.md`
- **Epics fallback:** `_bmad-output/planning-artifacts/epics.md`

**Legacy tickets `BST-5` and `BST-6` are OUT OF SCOPE.** They are hand-created placeholders from Linear setup (2026-07-03) that carry **no** `bmad-story-id` marker. Never match, reuse, retire, or relink them here — matching is by marker only, so they are invisible to this sync. Creating a fresh mirror ticket that overlaps a placeholder's topic is expected and acceptable; leave placeholder cleanup for 1.1b or a manual pass.

## Linear MCP tools used

- `mcp__claude_ai_Linear__list_issues` — list tickets in `BST Spike` for marker matching (read).
- `mcp__claude_ai_Linear__get_issue` — read a single ticket's current description/hash (read).
- `mcp__claude_ai_Linear__save_issue` — create (no `id`) and update (with `id`) tickets (write).

## The deterministic contract (pins behaviour)

1. **Marker (durable match key):** the `story_key`, stamped into the ticket **description** at creation as `<!-- bmad-story-id: <story_key> -->` on its own line, alongside `<!-- bmad-content-hash: <hash> -->`. Both land in the same create call → crash-safe (a ticket can never exist without its id marker).
2. **Matching:** match ONLY on the `bmad-story-id` marker. Never match on title, date, or the repo-side cached key.
3. **Idempotency (hash-based, not prose-diff):** compare the freshly computed content hash to the ticket's stored `bmad-content-hash`. Equal → **no write**. Differ → **update the same ticket** (rewrite body + hash). No marker match → **create**. Never decide by byte-diffing rendered prose — Linear re-renders markdown, so a rendered compare false-positives "changed" and would break the zero-writes no-op.
4. **Repo-side key cache:** on create, write the Linear key (e.g. `BST-12`) into the story file's `Linear:` header line as a **working-tree edit** — never a `git push`. Written **only for stories that have a per-story `.md` file**. Cache only; matching never depends on it.
5. **One-way:** nothing flows Linear → repo except the key identifier.

## Execution

<workflow>

  <step n="1" goal="Enumerate the backlog">
    <action>Read the FULL `sprint-status.yaml` and parse the `development_status:` mapping.</action>
    <action>Build the story set = every key, EXCLUDING keys matching `epic-*` and `*-retrospective`. The remaining keys (e.g. `1-1a-…`, `1-3-…`) are the story ids. This is the whole backlog — most stories have no per-story file yet; that is expected.</action>
    <action>For each `story_key`, derive its short number `X.Y`: take the first two dash-segments and join with a dot (e.g. `1-1a-…` → `1.1a`, `1-3-…` → `1.3`).</action>
  </step>

  <step n="2" goal="Source each story's content">
    <action>For each `story_key`, prefer the per-story file `_bmad-output/implementation-artifacts/<story_key>.md` if it exists (richest). Otherwise fall back to the `### Story X.Y` section of `_bmad-output/planning-artifacts/epics.md`.</action>
    <action>Extract these fields from the source:
      - **title** — from a per-story file: the H1 (`# …`) line, stripped of the leading `# `. From epics.md: the `### Story X.Y: …` heading text after the colon (or the full heading text if you keep the story number — be consistent across runs).
      - **user_story_block** — the "As a … / I want … / So that …" paragraph.
      - **acceptance_criteria** — the Acceptance Criteria section body (the Given/When/Then text, verbatim; a faithful mirror, do not paraphrase).</action>
    <action>Compute the **canonical repo link**:
      - if the per-story file exists: `https://github.com/piotr-nearform/bst-demo/blob/main/_bmad-output/implementation-artifacts/<story_key>.md`
      - else: `https://github.com/piotr-nearform/bst-demo/blob/main/_bmad-output/planning-artifacts/epics.md` (anchor to the story heading where practical).</action>
  </step>

  <step n="3" goal="Compute a deterministic content hash">
    <critical>Compute the hash with a real tool (sha256), NOT by eyeballing. This is what makes re-runs deterministic.</critical>
    <action>Build the hash input as the concatenation, in this fixed order, separated by a single newline: `title` + `user_story_block` + `acceptance_criteria`. (Do NOT include the marker lines, the repo link, or the `Linear:` key — those are either constant or self-referential.)</action>
    <action>Canonicalize the hash input deterministically: strip trailing whitespace on every line, collapse runs of 2+ blank lines to a single blank line, and strip leading/trailing blank lines from the whole string.</action>
    <action>Compute `hash = sha256(canonicalized_input)` as lowercase hex using a tool, e.g. write the canonicalized input to a scratch file and run `sha256sum`, or `python3 -c "import hashlib,sys;print(hashlib.sha256(sys.stdin.buffer.read()).hexdigest())"`. Use the full 64-char hex digest. The same source must always yield the same hash.</action>
  </step>

  <step n="4" goal="Match existing tickets by marker">
    <action>Call `mcp__claude_ai_Linear__list_issues` for project `BST Spike` (id `504c2cf9-d132-4ec4-a183-b74c5f73eddd`). Page through all results.</action>
    <action>For each returned ticket, read its description and look for `<!-- bmad-story-id: <value> -->`. Build a map `story_key → { issueId, storedHash }`, reading `storedHash` from that ticket's `<!-- bmad-content-hash: … -->` marker (use `get_issue` if the list payload does not include the full description).</action>
    <action>Tickets with no `bmad-story-id` marker (including legacy `BST-5`/`BST-6`) are ignored entirely.</action>
  </step>

  <step n="5" goal="Decide create / update / no-op per story">
    <action>For each `story_key` in the enumerated set, classify:
      - **no marker match in the map** → CREATE
      - **marker match AND storedHash == computed hash** → NO-OP (zero writes)
      - **marker match AND storedHash != computed hash** → UPDATE (same ticket id)</action>
    <action>Render a short plan table (story_key → CREATE/UPDATE/NO-OP) before writing, so the run is auditable.</action>
  </step>

  <step n="6" goal="Build the ticket body">
    <action>For CREATE and UPDATE, the ticket description is exactly:</action>
    <template>
<!-- bmad-story-id: {story_key} -->
<!-- bmad-content-hash: {hash} -->

**Canonical spec (repo — source of truth):** [`{repo_relative_path}`]({repo_link})

One-way mirror of the BMAD story spec. The repo is canonical; edits made here are not authoritative and are overwritten on the next sync.

---

{user_story_block}

## Acceptance Criteria

{acceptance_criteria}
    </template>
    <action>The **title** field of the ticket is the story `title` from Step 2.</action>
  </step>

  <step n="7" goal="Apply — create or update (Linear writes)">
    <critical>Do exactly one write per CREATE and per UPDATE. Do ZERO writes for NO-OP stories. Never create a second ticket for a story that already has a marker match.</critical>
    <action>For CREATE: call `mcp__claude_ai_Linear__save_issue` with NO `id`, `team` = `BST` (id `2b81e3e8-53d4-4c8f-8b78-6280d1dfb0a7`), `project` = `BST Spike` (id `504c2cf9-d132-4ec4-a183-b74c5f73eddd`), the title, and the description from Step 6. Capture the minted key (e.g. `BST-12`) from the response.</action>
    <action>For UPDATE: call `mcp__claude_ai_Linear__save_issue` WITH the existing `id`, and the new title + description (which carries the new hash). Same ticket, never a new one.</action>
    <action>For NO-OP: do nothing.</action>
  </step>

  <step n="8" goal="Repo-side key write-back (working-tree edit only)">
    <critical>NEVER `git push`, commit, open, or modify a PR here. The write-back is a plain file edit that rides the next human-approved PR.</critical>
    <action>For each story that was CREATED **and** has a per-story `_bmad-output/implementation-artifacts/<story_key>.md` file: edit that file's `Linear:` header line to `Linear: <MintedKey>` (e.g. `Linear: BST-12`), replacing the placeholder comment. One key per line, machine-parseable.</action>
    <action>Fileless stories (no per-story `.md`) get NO repo cache — that is correct, because matching is by the Linear-side marker. Do NOT create story files just to hold a key.</action>
    <action>Treat the cached key as a convenience only; an absent or stale key never causes a duplicate because matching is marker-based.</action>
  </step>

  <step n="9" goal="Report">
    <action>Summarise the run: N created (with keys), N updated, N no-op, and any story that could not be sourced. Note that repo-side `Linear:` edits are uncommitted working-tree changes for a human to review and land via PR.</action>
  </step>

</workflow>

## Crash-safety note

If a run is interrupted after `save_issue` creates a ticket but before the repo-side key is written back, the next run still matches that ticket by its `bmad-story-id` marker (Step 4) and classifies it NO-OP or UPDATE — never CREATE. So an interrupted run never produces a duplicate. The repo-side key is only a cache.

## How to run (and the manual first backfill)

This skill is agent-executed **in-session**, where the Linear MCP auth already exists (no new secret — honours NFR3). Invoke it on demand: "sync stories to Linear".

**First backfill (Story 1.1a, Task 6 — HUMAN-performed):** because the sync did not exist when the current backlog was authored, the first run is done manually by a human operator who reviews the plan table (Step 5) before the writes in Step 7 land. Run the skill against the current backlog; confirm it creates exactly one marker-stamped mirror ticket per enumerated story (including 1.1a itself), each carrying the `bmad-story-id` + `bmad-content-hash` markers and the canonical repo link, and that it does NOT touch `BST-5`/`BST-6`.

**Evidence to capture (Task 7 — attach to the PR):**
- (a) **Backfill** created N marker-stamped tickets including 1.1a (list the keys).
- (b) **No-op re-run:** running again with no repo changes makes zero writes / creates zero duplicates (the plan table is all NO-OP).
- (c) **Single-ticket update:** edit one story spec, re-run, and confirm only that one ticket updates (hash changed) while the rest stay NO-OP.
- (d) **Crash-safe re-find:** simulate an interrupted run (ticket created, key not yet written back) and confirm the next run re-finds it by marker — no duplicate.
