# BRIEF.md - demo specification

> Filled in by the agent during the intake protocol ([INTAKE.md](INTAKE.md)).
> Once written, this file is the single source of truth for the demo's
> identity (domain, names, scope, audience) for every downstream phase.

## Domain

**Pitch (one phrase):** _e.g. "pharma cold-chain logistics"_

**Why RelationalAI here, in one paragraph:** _what reasoner types are
naturally present in this domain - graph dependencies, rule cascades,
optimisation under constraint - that would be awkward in pure SQL._

## Inputs

- [ ] Schema (CSV / DDL / PDF): _path or "none"_
- [ ] Problem statement document: _path or "none"_
- [ ] Existing Snowflake database to demo against: _name or "none"_
- [ ] Otherwise: invent the data from domain knowledge

If a schema or problem doc was provided, paste a 5-line summary of what's
in it here:

> _summary_

## Scope

- **Length / depth:** _10 min / 3 Qs · 20 min / 5 acts · 30 min / 10 Qs · ontology-only_
- **Number of demo questions:** _N_
- **Reasoners showcased:** _rules · graph · heuristic · prescriptive · persistent rule · predictive (GNN)_
- **Cortex agent (Phase 7):** _yes / no_
- **Runbook + prep_demo gate (Phase 8):** _yes / no_

## Audience

_e.g. "Snowflake SE peers - technical, RAI-familiar" or "Pharma industry
experts - skeptical of synthetic data, will challenge cold-chain rate
assumptions"_

**Implication for data rigor:** _e.g. "use real ICAO milestones and
published 22% CTOT rate" or "stay loose, this audience won't dig"._

## Names (derived, lock these now)

| Thing | Value |
|---|---|
| Model name (PyRel) | `<snake_case_domain>` |
| Database name (Snowflake) | `PK_<DOMAIN_UPPER>` |
| **Demo role (security harness)** | `RAI_DEMO_<DOMAIN_UPPER>` |
| Schema for sources | `<DB>.<SCHEMA>` (typically same as DB or `EHAM`-style sub-schema) |
| Schema for agent | `<DB>.RAI_AGENT` |
| Logic engine | `<model>_logic_l` (HIGHMEM_X64_L) |
| Prescriptive engine | `<model>_prescriptive_m` (HIGHMEM_X64_M) |
| Notebook stage | `<DB>.NOTEBOOKS.<DOMAIN>_NOTEBOOK_STAGE` |
| Snowsight notebook | `<DB>.NOTEBOOKS.<DOMAIN>_DEMO` |
| Cortex agent | `<domain>` (lowercase) |

## Snowflake security harness

> Filled in during intake step 3 (bootstrap). The role + DB created here are
> the only Snowflake objects the agent ever touches outside of the demo
> itself. See CLAUDE.md > "Snowflake security harness" for the full model.

- **Bootstrap SQL run:** `data/00_bootstrap.sql` (committed to the repo for
  review). Run on: _date_ as: _profile default role_ against: _account_.
- **Demo role:** `RAI_DEMO_<DOMAIN_UPPER>`. Granted to user
  _CURRENT_USER_ and to ROLE SYSADMIN.
- **Demo database:** `PK_<DOMAIN_UPPER>`, owned by the demo role.
- **RelationalAI Native App:** _app name_. Granted application roles:
  _list of app role names_.
- **Snowflake Intelligence:** `CREATE AGENT` on
  `SNOWFLAKE_INTELLIGENCE.AGENTS` granted.
- **Warehouse:** `RAI_XS` with `USAGE` + `OPERATE` only (no MODIFY).

**Confirmed limitations of the demo role:**
- Cannot CREATE / DROP / ALTER any user.
- Cannot CREATE / DROP / ALTER any database, schema, table, or warehouse
  outside `PK_<DOMAIN_UPPER>`.
- Cannot create programmatic access tokens (PATs).
- Cannot GRANT or REVOKE any privilege.
- Cannot modify the RelationalAI native app or any integration.

**To tear down the demo entirely** (manual, by the user, not the agent):
```sql
USE ROLE ACCOUNTADMIN;
DROP DATABASE IF EXISTS PK_<DOMAIN_UPPER>;
DROP ROLE IF EXISTS RAI_DEMO_<DOMAIN_UPPER>;
```

## Anchored numbers

> Phase 2 fills these in. They are the specific named entities and counts
> the talk track will hinge on. The pattern is the
> `airplanes_demo/CLAUDE.md` "Anchored numbers (from `<domain>_demo_validation.sql`)"
> table - every cell reproduces from raw SQL. Add SQL to
> `data/<domain>_demo_validation.sql` and assert outputs in `prep_demo.py`.

| Metric | Value | SQL source |
|---|---|---|
| _TBD_ | _TBD_ | _TBD_ |

## Phase log

> Append a one-paragraph entry after each phase exits green. Keeps a
> tidy trail for the handoff document at Phase 9.

### Phase 1 - Positioning
_pending_

### Phase 2 - Data
_pending_

### Phase 3 - Ontology
_pending_

### Phase 4 - Queries
_pending_

### Phase 5 - Local notebook
_pending_

### Phase 6 - Snowsight notebook
_pending_

### Phase 7 - Cortex agent
_pending_

### Phase 8 - Gate + runbook
_pending_

### Phase 9 - Talk track + handoff
_pending_

## Design decisions

> Anything you decided that was non-obvious - schema grain, why you picked
> a particular reasoner for a particular question, why you ruled out a
> question, why the anchored numbers are what they are. This becomes the
> backbone of `HANDOFF_BRIEFING.md` at Phase 9.

## Open questions

> Things you couldn't resolve and proceeded around. Surface in the final
> summary.

## Autonomy issues

> Times the pre-tuned permissions interrupted you when they shouldn't have,
> or vice versa. Helps tune the template settings.
