# Web-Verification Review — Architecture Spine (Double-Booking Guard)

**Reviewer role:** Verify every load-bearing technical claim in the architecture spine is current and correct as of mid-2026 (2026-07-02), not asserted from stale training data.

**Target reviewed:** `_bmad-output/planning-artifacts/architecture/architecture-bst-spike-2026-07-02/ARCHITECTURE-SPINE.md`

**Method:** Web search against primary sources (Microsoft Learn / Azure docs, official PostgreSQL documentation) plus corroborating practitioner sources.

---

## Verdict

**PASS — all four load-bearing claims check out.** Every technical assertion in the spine is current and correct as of mid-2026. No out-of-date, incorrect, or unsupported claims found. One deliberate, subtle design choice (`'[]'` inclusive bounds) is not just correct but is the *load-bearing* detail on which the FR3 clash semantics depend — it is verified correct and worth calling out as a strength, not a defect.

---

## Claim-by-claim findings

### Claim 1 — Azure PG Flexible Server supports `btree_gist` + EXCLUDE constraints, enabled via `azure.extensions` allow-list + `CREATE EXTENSION` (spine AD-1, AD-5)

**Status: VERIFIED — correct.**

- `btree_gist` is on Azure Flexible Server's supported extensions list (Microsoft Learn: "List of the PostgreSQL Extensions and Modules … for Azure Database for PostgreSQL Flexible Server").
- Azure requires the extension to be allow-listed via the `azure.extensions` server parameter *before* `CREATE EXTENSION` will succeed — set via Portal (Settings → Server parameters → `azure.extensions`) or CLI (`az postgres flexible-server parameter set --name azure.extensions --value btree_gist,...`). Only after allow-listing does the standard `CREATE EXTENSION` statement work.
- This is exactly the two-step sequence AD-5 describes ("`azure.extensions` must list it … **and** a migration runs `CREATE EXTENSION IF NOT EXISTS btree_gist`. On Azure Flexible Server, `CREATE EXTENSION` is rejected unless the allow-list parameter is set first"). The ordering and the rejection behaviour are both accurately stated.

*Sources:* [How to allow extensions (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-allow-extensions); [Extensions and modules by engine (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/postgresql/extensions/concepts-extensions-by-engine); [Introducing allow-list for extensions (Microsoft Community Hub)](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/introducing-ability-to-allow-list-extensions-in-postgresql/ba-p/3219124)

### Claim 2 — Supported PG versions 16/17/18 on Azure Flexible Server; UK South is a valid region (spine Stack table, deployment envelope)

**Status: VERIFIED — correct.**

- Azure Flexible Server currently supports PostgreSQL 18, 17, 16, 15, 14, 13 (older majors are being retired per the version policy). 16, 17, and 18 are all supported, matching the spine's "16/17/18 all supported" and its pin to 17. PG 17 is GA including in-place major version upgrade.
- UK South is a valid Azure Flexible Server region (appears across Azure PG docs for feature/region availability, Premium SSD v2, HA, geo-redundant backup, and third-party UK South pricing pages). No issue with the deployment envelope's "Azure — UK South".

*Sources:* [Supported versions of PostgreSQL (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-supported-versions); [Reliability in Azure Database for PostgreSQL (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/reliability/reliability-postgresql-flexible-server); [Premium SSD v2 (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/postgresql/configure-maintain/concepts-storage-premium-ssd-v2)

*Note (informational, not a defect):* the spine pins **17** and lists 16/17/18 as the supported band. This is a safe, current choice. If the spike is provisioned much later, re-confirm the minor and that no listed major has entered retirement — but as of 2026-07-02 the statement is accurate.

### Claim 3 — The SQL prevents per-person overlaps; violating INSERT raises SQLSTATE 23P01; `'[]'` inclusive bounds make 3–7 Aug clash with 7–10 Aug (spine AD-1, AD-4)

**Status: VERIFIED — correct, and the inclusive-bounds detail is the crux.**

- `EXCLUDE USING gist (person_id WITH =, daterange(start_date,end_date,'[]') WITH &&)` genuinely enforces "no two rows with the same `person_id` may have overlapping dateranges." The `WITH =` scopes comparison to the same person; the `WITH &&` (overlap) rejects any pair sharing a data point. This is the canonical Postgres pattern for double-booking prevention.
- A violating INSERT/UPDATE raises **SQLSTATE 23P01 `exclusion_violation`** — distinct from `23505` (unique) and `23503` (FK). The spine's `23P01 → 409` mapping is on the correct error code.
- **The `'[]'` choice is load-bearing and correct.** PostgreSQL's *default* daterange canonical form is `[)` (inclusive lower, exclusive upper). Under `[)`, two ranges that merely touch at an endpoint do **not** overlap — so `[3 Aug, 7 Aug)` and `[7 Aug, 10 Aug)` would **not** clash. The spine explicitly uses `'[]'` (inclusive **both** bounds), under which 7 Aug is a member of both ranges, so `&&` returns true and the two clash. This is precisely FR3's stated rule ("any shared day, endpoints included; 3–7 Aug clashes with 7–10 Aug"). The spine has chosen the one bound-spec that makes the requirement true, and AD-4 correctly names this as the single source of the clash rule. **Had the spine used the default `[)`, the requirement would be silently violated at the shared endpoint** — the author avoided exactly that trap.

*Sources:* [PostgreSQL 18 docs — Range Types (§8.17, inclusive/exclusive bounds and canonicalization)](https://www.postgresql.org/docs/current/rangetypes.html); [PostgreSQL 18 docs — btree_gist (F.8)](https://www.postgresql.org/docs/current/btree-gist.html); [PostgreSQL GiST exclusion constraint for double bookings (amitavroy)](https://amitavroy.com/articles/postgresql-gist-exclusion-constraintthe-database-evel-answer-to-double-bookings); [What goes into an exclusion constraint (danielclayton.co.uk)](https://blog.danielclayton.co.uk/posts/overlapping-data-postgres-exclusion-constraints/)

### Claim 4 — `btree_gist` is what enables an equality (`=`) predicate alongside a range (`&&`) predicate in one GiST EXCLUDE constraint (spine AD-1, Stack table)

**Status: VERIFIED — correct.**

- Scalar/discrete types (`uuid`, `integer`, `text`) have only B-tree operator classes by default and no native GiST operator class, so they cannot participate in a GiST index / EXCLUDE `WITH =` on their own. `btree_gist` adds GiST operator classes for those scalar types, which is exactly what lets `person_id WITH =` sit in the same GiST exclusion constraint as `daterange(...) WITH &&`. Without `btree_gist`, the `person_id WITH =` term would fail to create. The spine's "requires `btree_gist`" and "enables the EXCLUDE constraint" framing is accurate, and its "Load-bearing" status label is justified.

*Sources:* [PostgreSQL 18 docs — btree_gist (F.8)](https://www.postgresql.org/docs/current/btree-gist.html); [Exclusion constraints beyond UNIQUE (CYBERTEC)](https://www.cybertec-postgresql.com/en/postgresql-exclusion-constraints-beyond-unique/)

---

## Severity summary

| # | Finding | Severity | Fix |
| --- | --- | --- | --- |
| 1 | `btree_gist` + `azure.extensions` allow-list + `CREATE EXTENSION` sequence (AD-1/AD-5) | None — verified correct | No action |
| 2 | PG 16/17/18 supported + UK South valid region | None — verified correct | (Optional) re-confirm minor version + retirement band at provisioning time if far in the future |
| 3 | EXCLUDE SQL prevents overlaps; 23P01 on violation; `'[]'` makes shared-endpoint 3–7/7–10 Aug clash | None — verified correct; the `'[]'` choice is the crux and is right | No action — preserve `'[]'` exactly; do not "canonicalize" to `[)` |
| 4 | `btree_gist` enables `=` alongside `&&` in one GiST EXCLUDE | None — verified correct | No action |

**Nothing flagged as out-of-date, incorrect, or asserted-without-basis.** The spine's load-bearing decisions were reality-checked and hold up against primary sources as of 2026-07-02.

---

## One watch-item for downstream (not a spine defect)

The correctness of FR3 hinges entirely on the literal `'[]'` in the migration. When this is implemented, the acceptance test for the shared-endpoint case (3–7 Aug vs 7–10 Aug → clash) is the one that will catch any accidental drift to the default `[)`. The spine already routes FR3 to `db/migrations/` with the constraint as its spec (spec-as-contract), so this is covered — flagged only so implementers know which single character is doing the work.
