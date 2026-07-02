# Rubric-Walker Review — ARCHITECTURE-SPINE (Double-Booking Guard)

- **Reviewer role:** rubric walker (good-spine checklist)
- **Subject:** `ARCHITECTURE-SPINE.md` (FEATURE altitude, layered)
- **Source PRD:** `prd-bst-spike-2026-07-01/prd.md` (FR1–FR5, AC1–AC4) + `addendum.md`
- **Date:** 2026-07-02
- **Verdict:** **PASS-WITH-FIXES**

Framing honoured: this is a deliberately throwaway 1–2 day spike whose point is the ways-of-working pipeline, and the constitution forbids speculative abstraction. "Minimal" is judged as correct. Findings below are only genuine gaps where two units *could* diverge, a dimension is silent, or a rule is not actually enforceable — not requests for structure for its own sake.

---

## Checklist walk

### 1. Does it fix the real divergence points for the level below (epics/stories)?

Largely yes, and well. The one genuinely load-bearing decision — *where the clash rule is allowed to live* — is nailed by AD-1 (constraint is the enforcement) and AD-2 (service translates, never detects). This is the exact place a story could otherwise diverge (one dev adds a "helpful" SELECT-then-check), and the spine forbids it explicitly. AD-4 pins the clash semantics (inclusive `[]` daterange, `&&`) in one place. AD-3 fixes the loser contract at both layers. The Capability→Architecture map ties every FR/AC to a location and a governing AD. Good.

One divergence point is under-specified — see Finding H1 (concurrency test harness / "how do two writes actually race").

### 2. Is every AD's Rule enforceable and does it actually prevent its stated divergence?

- **AD-1** — Enforceable and self-enforcing: the constraint physically makes overlap unpersistable; `23P01` is the observable failure. Strongest possible enforcement. ✔
- **AD-2** — Enforceable in review ("no SELECT-then-decide"), and AD-1 makes any app-level check redundant rather than authoritative. Prevents the second-source-of-truth drift. ✔
- **AD-3** — Enforceable via the API 409 (AC4 pins it) and the stable test hook. ✔ Minor: status is unlabelled — see L1.
- **AD-4** — Enforceable; the `'[]'` bound literal is the single encoding and AD-1 consumes it. ✔
- **AD-5** — Enforceable and correctly identifies the real Azure Flexible Server trap (`azure.extensions` allow-list must be set *before* `CREATE EXTENSION` succeeds). This is the one that most often bites in a fresh environment; capturing both steps in version control is right. ✔

### 3. Could anything under Deferred let two units diverge?

Reviewed each deferred item:

- **App host (Container Apps vs App Service)** — single app, no cross-unit contract; safe to defer. ✔
- **JS stack versions** — pulled in by first story; indicative per constitution. Safe. ✔ (but see M1 — one dimension the *deferral itself* leaves silent.)
- **UI message copy & surfacing** — AD-3 already fixes the invariant (stable hook + plain language + never-a-500); copy/surfacing are genuinely story-local and cannot cause divergence because tests key on the hook, not the copy. ✔
- **Out-of-PRD scope** — correctly excluded. ✔

No deferred item lets two units diverge. ✔

### 4. Is named tech verified-current?

- **PostgreSQL 17 on Azure Database for PostgreSQL Flexible Server, UK South** — plausible and the version note ("16/17/18 all supported") is appropriately hedged. Author states web-verification of Postgres `EXCLUDE`/`btree_gist` on Azure UK. `EXCLUDE USING gist` + `btree_gist` for equality-plus-range is a long-standing, stable Postgres capability, and `btree_gist` is on the Azure Flexible Server allow-list. No red flags. ✔
- **`btree_gist` bundled with the PG version** — correct; it is a contrib extension shipped with the server, gated by the allow-list (AD-5). ✔
- **Playwright / Node LTS / React** — Playwright confirmed by constitution; Node/React correctly kept indicative. ✔

No stale or invented version claims found.

### 5. If a spec drove it, does it cover that spec's capabilities? (FR1–FR5, AC1–AC4)

| Item | Covered? | Where |
| --- | --- | --- |
| FR1 create assignment (UI) | ✔ | `ui/`+`api/`, AD-4 |
| FR2 non-clashing succeeds | ✔ | AD-1 (constraint admits) |
| FR3 clash rejected, inclusive overlap | ✔ | AD-1 + AD-4 |
| FR4 concurrency atomic, exactly-one-wins | ✔ | AD-1 (`23P01` on loser) |
| FR5 deterministic loser contract | ✔ | AD-2 + AD-3 |
| AC1 concurrency via UI | partial | `tests/e2e/`; harness under-specified — H1 |
| AC2 happy path | ✔ | `tests/e2e/` |
| AC3 sequential + boundary (7–10 Aug) | ✔ | AD-4 boundary example matches PRD |
| AC4 API 409 | ✔ | `tests/api/`, AD-3 |

Full FR/AC coverage. The only softness is AC1's *mechanism of racing* (H1), not its coverage.

### 6. Is every dimension the altitude owns decided, deferred, or an open question? (esp. operational/environmental envelope)

Dimensions owned at FEATURE altitude and their status:

- **Data model / persistence** — decided (AD-1, AD-4, conventions). ✔
- **Clash enforcement** — decided (AD-1/AD-2). ✔
- **API contract** — decided (AD-3, 409). ✔
- **UI contract** — decided to the invariant level (AD-3). ✔
- **Deployment & environments** — *partially* decided. The deployment envelope diagram (Azure UK South, PaaS app, Flexible Server, Key Vault) and AD-5 cover provider + region + the extension-provisioning path. This is a real strength — the operational envelope is **not** silent. **But two operational sub-dimensions are silent** — see M1 and M2.
- **Infra/provider strategy** — decided (Azure UK South, Flexible Server). ✔
- **Operations (migrations, secrets)** — decided (migrations in VC, secrets via Key Vault/env, AD-5). ✔ for a spike.
- **Test strategy** — decided (isolation convention, hooks, P0 pin). ✔

No *whole* dimension is silent, and the operational/environmental envelope is explicitly addressed — the checklist's highest-priority worry is satisfied. Two narrower operational gaps remain (M1, M2).

---

## Findings

### H1 — [HIGH] AC1's concurrency harness (how the two writes actually race) is not fixed, and it is a real divergence point
AD-1 guarantees "exactly one wins" *at the database*, and that is correct and load-bearing. But the PRD/addendum are explicit that the P0 test drives **two browser contexts** firing near-simultaneously, and that "two near-simultaneous clicks are not perfectly simultaneous" — so the addendum flags that a thin service/DB-level check may be needed to pin the race deterministically. The spine's conventions table says "a service/API-level test pins AC4 (409) deterministically," but AC4 is the *sequential* API contract, not the race. Nothing in the spine fixes *how AC1's race is made deterministic* — leaving two stories free to diverge (one asserts on the UI double-submit and flakes; another adds an ad-hoc transaction-timing hack). This is the single highest-risk feature in the PRD (concurrency is "the one place depth is warranted") and the spine should state the invariant for how the race is exercised/pinned, even if the copy/mechanics are story-local.
**Fix:** add one line to conventions or an AD note: the P0 race is exercised via two concurrent writes (Playwright `Promise.all` over two contexts) *and* pinned deterministically at the API/DB layer (two overlapping inserts in-flight → exactly one `23P01`); AC1 asserts the observable outcome, the API/DB test guarantees determinism.

### M1 — [MEDIUM] Migration tooling / ownership is silent — the constraint (the spec) has no named home
`db/migrations/` is named and "migrations in version control" is a convention, but *what runs them* is unstated (Flyway? Prisma Migrate? node-pg-migrate? raw SQL + psql?). Because the spine declares "the constraint **is** the spec for FR3," the mechanism that applies and re-applies that constraint in a fresh Azure environment is operationally load-bearing, and AD-5 depends on a migration executing `CREATE EXTENSION`. Two stories could pick different tools or hand-run SQL. For a spike this can stay lightly-held, but it should be a named decision or an explicit open question, not silent.
**Fix:** name the migration approach (even "raw SQL migrations applied by a single tracked script/tool, chosen at first scaffold") or list it as an open question rather than leaving it implicit.

### M2 — [MEDIUM] No environment/config contract for the DB connection across local vs Azure
Secrets are covered (Key Vault/env, never in git), and test isolation mandates a fresh DB. But the spine does not fix *how the app is pointed at a database* — the connection-string/env-var contract that both the running app and the Playwright acceptance run depend on. The test-isolation convention ("fresh, isolated database/schema") implies a local/CI Postgres, while the deployment envelope implies Azure Flexible Server; nothing states the single config seam that makes both work. This is small but it is the operational seam most likely to cause two stories (app wiring vs test wiring) to diverge.
**Fix:** one convention line naming the DB-connection env contract (e.g. a single `DATABASE_URL`-style var, sourced from env locally/CI and Key Vault on Azure) so app and tests share one seam.

### L1 — [LOW] AD-3 carries no ADOPTED/status label; AD-4 likewise
AD-1 and AD-2 are tagged `[ADOPTED]`; AD-3, AD-4, AD-5 have no status marker. All read as adopted and are bound in the map, so this is cosmetic, but the inconsistency invites "is AD-3 provisional?" questions at story time.
**Fix:** label AD-3/AD-4/AD-5 `[ADOPTED]` for parity.

### L2 — [LOW] `end_date`/`start_date` column names are assumed but never stated as the schema convention
AD-1 and AD-4 both write `daterange(start_date, end_date, '[]')`, implicitly fixing two column names, but the conventions table (which pins uuid PKs and entity names) doesn't record them. Harmless given the AD text, but the single source of column naming is the constraint expression rather than the convention list.
**Fix:** optional — note `assignment.start_date` / `end_date` (or a single `during daterange` column) in conventions to remove the ambiguity between "two date columns" and "one range column."

---

## What is deliberately and correctly minimal (not findings)

- No ports/adapters, no domain core, no repository layer — correct for a throwaway spike; the constitution forbids the abstraction.
- UI copy/surfacing, host choice, JS versions deferred — correct; none can cause divergence.
- No auth/roles, no person/job management — correctly out of scope per PRD.
- Single-app structural seed with four folders — right-sized.

## Summary

A tight, genuinely spine-shaped document: it isolates the one load-bearing decision (clash rule lives only at the data layer), enforces it with the strongest possible mechanism, and — notably — does **not** leave the operational/environmental envelope silent (Azure UK South, Flexible Server, `btree_gist` provisioning via AD-5 is a real strength). It passes with fixes. The one HIGH gap is that the *deterministic pinning of the P0 race* — the highest-risk thing in the whole spike — is not fixed as an invariant, leaving stories free to diverge on how AC1 is made non-flaky. Two MEDIUM operational seams (migration tooling, DB-connection config contract) are silent within an otherwise-addressed operations dimension. All are cheap to close with one or two lines each.
