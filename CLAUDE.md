# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A throwaway spike: a small staff-to-job scheduling app, loosely shaped like BST, built to practise the BMAD v6 ways of working before the real engagement starts. The app is not the point; the pipeline is. Engagement and cross-repo context lives in the workspace wiki at `../docs/wiki/` (start at `overview.md`). The spike is scoped in the wiki's delivery proposal (`../docs/wiki/topics/bmad-delivery-proposal.md`).

## Stack (assumed, mirrors expected BST)

- **Frontend:** React.
- **Backend:** Node.
- **Hosting:** Azure.
- **Code / CI / review:** GitHub (GitHub Actions, GitHub Agentic Workflows for multi-model review).
- **Board:** Azure Boards, linked to GitHub so commits and PRs update tickets.

## Commands

- **Install:** `[command]`
- **Dev / run:** `[command]`
- **Build:** `[command]`
- **Test (all):** `[command]`
- **Test (single):** `[command]`
- **Lint:** `[command]`
- **Type-check:** `[command]`

[Document the real commands once they exist. Do not invent them.]

## Architecture

[Fill in once the app takes shape. Keep it high-level.]

## Conventions

Agent rules live in a clear layering: `project-context.md` (BMAD's constitution, the hard rules every agent obeys) alongside this `CLAUDE.md` / `AGENTS.md` (the operating manual). One of these is the named source of truth; keep the others in sync with it. Part of the spike is deciding which and proving they do not drift.

## Testing

Risk-based per the engagement method (see `../docs/wiki/concepts/tea-test-architect.md`): test depth scales with business risk (P0-P3). Acceptance tests are written first from the story AC (Playwright), before `bmad-dev-story` runs.

## Spec-as-contract

This codebase is delivered spec-driven (see `../docs/wiki/concepts/spec-driven-development.md`). If code changes, update the corresponding BMAD story/PRD. Drift from the spec is a failure, like a failing test.
