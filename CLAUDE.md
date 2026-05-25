# CLAUDE.md - agent orientation for this RelationalAI demo

You are about to build (or work on) a RelationalAI demo against the sales-
engineering Snowflake account. This file is the entry point. Read it fully
before doing anything else.

## What this repo is

A blank slate cloned from `demo-agent-template`. The end state is a full
end-to-end RelationalAI demo following the shape of the two reference demos:

- `/Users/piotrkraus/rai-repos/supply_chain_demo` - 10 questions, LP + knapsack
- `/Users/piotrkraus/rai-repos/airplanes_demo` - 5 acts, MIP + persistent rule

The shape is described in [PIPELINE.md](PIPELINE.md). Both reference demos
live as sibling directories and are mapped file-by-file in [REFERENCES.md](REFERENCES.md).

## First action (always)

Check whether `BRIEF.md` exists at the repo root.

- **If `BRIEF.md` is missing or is still the unfilled template**, you must run
  the intake protocol in [INTAKE.md](INTAKE.md) before anything else. The
  intake is five questions delivered via the `AskUserQuestion` tool, then
  you write `BRIEF.md` from the answers. Do not skip this - every downstream
  phase depends on the brief.
- **If `BRIEF.md` is filled in**, read it, then read [PIPELINE.md](PIPELINE.md)
  and resume from the first phase whose exit criteria are not met. Do not
  ask the user where to resume - the file system tells you.

After the intake, proceed through [PIPELINE.md](PIPELINE.md) one phase at a
time. Each phase has explicit exit criteria. You do not move to the next
phase until the current phase is green. Use the `TaskCreate` / `TaskUpdate`
tools to track phase progress so the user can watch.

## RelationalAI skills (how they load, how to use them)

Two skill sources, in order of priority.

### 1. The `rai@RelationalAI` marketplace plugin (15 skills, auto-loaded)

The plugin is registered in [.claude/settings.json](.claude/settings.json) via:

```json
"extraKnownMarketplaces": {
  "RelationalAI": {"source": {"source": "github", "repo": "RelationalAI/rai-agent-skills"}}
},
"enabledPlugins": {"rai@RelationalAI": true}
```

Claude Code fetches the plugin from GitHub on session start and exposes 15
slash-skills automatically. Confirm they loaded by checking
`/Users/piotrkraus/.claude/plugins/` (cache) or by typing `/rai-` in chat
and seeing what completes. If nothing completes, the plugin failed to load -
run the user-facing command `/plugin install rai@RelationalAI` once, or
verify network access to github.com.

| Phase | Primary skill | Secondary |
|---|---|---|
| 1 - Position | `/rai-discovery` | `/rai-ontology-design` |
| 2 - Data | *(no RAI skill - `snow` CLI)* | `/rai-setup` to validate |
| 3 - Ontology | `/rai-build-starter-ontology` then `/rai-ontology-design` | `/rai-pyrel-coding` |
| 4 - Queries | `/rai-querying` (always first), `/rai-rules-authoring`, `/rai-graph-analysis`, `/rai-prescriptive-problem-formulation` → `/rai-prescriptive-solver-management` → `/rai-prescriptive-results-interpretation` | `/rai-discovery` to classify each question |
| 5 - Local notebook | `/rai-querying`, `/rai-pyrel-coding` | - |
| 6 - Snowsight notebook | `/rai-querying`, `/rai-pyrel-coding` | `/rai-setup` for stage upload |
| 7 - Cortex agent | `/rai-cortex-integration` | `/rai-querying` for the catalog |
| 8 - Gate + runbook | `/rai-health` | `/rai-setup` |
| 9 - Talk track + handoff | `/rai-prescriptive-results-interpretation` | - |

The full set: `rai-setup`, `rai-discovery`, `rai-build-starter-ontology`,
`rai-ontology-design`, `rai-pyrel-coding`, `rai-querying`, `rai-rules-authoring`,
`rai-graph-analysis`, `rai-prescriptive-problem-formulation`,
`rai-prescriptive-solver-management`, `rai-prescriptive-results-interpretation`,
`rai-cortex-integration`, `rai-health`, `rai-predictive-modeling` (early
access, GNN), `rai-predictive-training` (early access, GNN).

**`/rai-querying` is mandatory at Phase 4** - its SKILL.md says: *"Load this
BEFORE writing any PyRel query, even your first one - your prior knowledge of
the syntax is likely stale."* Treat that as binding.

Skills auto-load, but invoke them explicitly when you enter each phase using
the `Skill` tool - that signals progress to the user.

### 2. Project-local skills (the pre-GA-feature pattern)

For pre-GA features that aren't in the marketplace plugin yet, the reference
demos add a project-local skill at `.claude/skills/<skill-name>/SKILL.md`.
Claude Code picks these up automatically from the active project.

The canonical example is
`supply_chain_demo/.claude/skills/rai-pathfinder/SKILL.md` - a 417-line
authoritative reference for path queries via
`relationalai.semantics.std.paths` (codename "Pathfinder"). The marketplace
`/rai-graph-analysis` skill does NOT cover path enumeration; rai-pathfinder
does.

**Rule for your demo:** if Phase 4 has a question that uses
`relationalai.semantics.std.paths` (path enumeration, BOM walks,
multi-leg routing), copy that SKILL.md into your repo at
`.claude/skills/rai-pathfinder/SKILL.md` and load it before writing the
query. Same pattern for any future pre-GA feature - the agent invents a
local skill so the rules are written down once and the next session
inherits them.

## PyRel source-of-truth checkout

The PyRel repo is checked out locally at `/Users/piotrkraus/rai-repos/PyRel`.
Treat it as ground truth when the marketplace skills or public docs are
thin. The installed `relationalai` package in `.venv/` may lag the checkout -
run `uv pip show relationalai` and compare against `PyRel/pyproject.toml`.

| Path | What's in it |
|---|---|
| `PyRel/src/relationalai/semantics/` | Authoritative API surface. `std/paths` is the pathfinder source. `reasoners/prescriptive/` is the LP/MIP layer. |
| `PyRel/example/paths/` | Runnable path-query examples. **Read before authing a Phase 4 graph-path question.** |
| `PyRel/example/prescriptive/` | Runnable LP/MIP examples. Read before Phase 4 prescriptive. |
| `PyRel/example/cortex/` | Runnable Cortex agent examples. Read at Phase 7. |
| `PyRel/tests/end2end/` | Tests as documentation - each test is a small runnable PyRel program. Grep here for unfamiliar idioms. |
| `PyRel/tests/reasoners/` | Reasoner-specific tests. Same as above, narrower scope. |
| `PyRel/docs/` | Markdown documentation. Less polished than the public site but more current. |
| `PyRel/notes/` | Design notes and unreleased-feature discussion. |

Read access to `/Users/piotrkraus/rai-repos/PyRel/**` is pre-allowed in
[.claude/settings.json](.claude/settings.json). When you cite an API
signature in a comment or docstring, prefer pasting it from PyRel source
over your training data.

## Environment is pre-configured for autonomous runs

[.claude/settings.json](.claude/settings.json) pre-allows the commands you
need (`snow sql`, `snow connection`, `snow notebook`, `uv`, `.venv/bin/python`,
`.venv/bin/rai`, `.venv/bin/jupyter`, read-only git, common file utilities)
and pre-accepts file edits inside this repo. You should be able to run for an
hour without an interrupt during normal phase execution.

**Things that will still ask** (intentionally):
- `git push`, `git reset --hard`, `git checkout --force`
- `rm -rf` of anything
- `agent/deploy.py teardown`
- Any SQL that contains `DROP DATABASE`, `TRUNCATE`, `DELETE FROM` of source data

If something interrupts you that shouldn't, log it in `BRIEF.md` under
`### Autonomy issues` and continue - don't relitigate it with the user.

## Snowflake conventions (sales-engineering account)

- Connection profile: `rai` in `~/.snowflake/connections.toml`
- Account: `ajb85638`
- Warehouse: `RAI_XS` (auto-resume; cheap)
- **Role: `RAI_DEMO_<DOMAIN>` (created at intake)** - scoped to the demo DB only.
  Every `snow sql` call passes `--role RAI_DEMO_<DOMAIN>` explicitly. See
  the "Snowflake security harness" section below for the full model.
- Native App: RelationalAI is installed on this account
- PyRel auto-discovers credentials from `~/.snowflake/connections.toml` - no `raiconfig.yaml` needed
- Engine sizing convention from the reference demos:
  - Logic engine: `<domain>_logic_l` at `HIGHMEM_X64_L` (rules + graph + heuristic)
  - Prescriptive engine: `<domain>_prescriptive_m` at `HIGHMEM_X64_M` (LP / MIP)
  - Both are **named** engines so they persist across runs

The exact `_build_config()` pattern that works in both local Python AND inside
a Snowsight notebook is in `supply_chain_demo/rai_code/manual/supply_chain.py`
- copy it.

## Snowflake security harness

The sales-engineering account is shared. The `rai` connection profile's
default role on this machine has broad privileges (likely `ACCOUNTADMIN`).
Running the agent unsupervised against that profile is unsafe. The harness
has two layers.

### Layer 1: a demo-specific role (the real defense)

At intake, the agent generates `data/00_bootstrap.sql` from
[BOOTSTRAP.template.sql](BOOTSTRAP.template.sql) and asks the user to
review and confirm. The bootstrap, run **once** as the rai profile's
default role:

1. Creates `RAI_DEMO_<DOMAIN>`, owned by SYSADMIN, granted to the current user.
2. Creates the demo database `<DEMO_DB>` and grants the demo role ALL on it
   (current + future schemas, tables, views, stages, procs, notebooks).
3. Grants the demo role `USAGE` + `OPERATE` on warehouse `RAI_XS`. No
   `MODIFY`, so the role cannot alter or drop the warehouse.
4. Grants the RAI Native App's `RAI_DEVELOPER` / `RAI_USER` application
   roles to the demo role (so PyRel works).
5. Grants minimal Snowflake Intelligence access for Cortex agent deploy.

After bootstrap, the demo role can do **anything inside the demo DB** and
nothing outside it. Snowflake itself enforces this. The agent cannot:

- DROP / ALTER any database, schema, table, or warehouse outside `<DEMO_DB>` (no grants).
- CREATE / DROP / ALTER any user (not an account-level role).
- Create programmatic access tokens (PATs) - requires `CREATE USER` or `ALTER USER`.
- GRANT or REVOKE any privilege (no `MANAGE GRANTS` on the role).
- Touch the share, network policies, integrations, or replication (no grants).

### Layer 2: Claude Code Bash deny list (belt and braces)

[.claude/settings.json](.claude/settings.json) blocks risky one-liners at
the shell layer, before Snowflake even sees them:

- `USE ROLE ACCOUNTADMIN` / `SECURITYADMIN` / `SYSADMIN` in `snow sql -q`
- `--role ACCOUNTADMIN` / `SECURITYADMIN` / `SYSADMIN` on the CLI
- `CREATE USER`, `ALTER USER`, `DROP USER`
- `PROGRAMMATIC ACCESS TOKEN` (PAT creation)
- `GRANT ... ON ACCOUNT`, `MANAGE GRANTS`, `IMPORTED PRIVILEGES`
- `CREATE WAREHOUSE`, `DROP WAREHOUSE`, `ALTER WAREHOUSE`
- `CREATE ROLE` / `DROP ROLE` outside the demo-role pattern (`RAI_DEMO_*`)

These would be denied by Snowflake anyway under the demo role, but the
shell-layer deny stops the agent from accidentally hand-typing them when
the user has temporarily switched into ACCOUNTADMIN for a one-off.

### Rules of the road for the agent

1. **Every `snow sql` call** explicitly passes `--role RAI_DEMO_<DOMAIN>`
   and `-c rai`. Never rely on the profile default. Example:
   ```
   snow sql --role RAI_DEMO_SUPPLY_CHAIN -c rai -q "SELECT 1"
   snow sql --role RAI_DEMO_SUPPLY_CHAIN -c rai -f data/02_schema.sql
   ```
2. **DDL files live under `data/` and are reviewable.** When you write
   bootstrap, schema, or grant SQL, save it to a file and run via
   `snow sql -f`, not via `-q`. The user reviews; that is the gate.
3. **If you think you need ACCOUNTADMIN**, you don't. Either the demo
   role is missing a grant (fix the bootstrap, ask the user to re-run it),
   or you're operating outside the demo's scope (you shouldn't be).
4. **Engines are named and managed by the demo role.** The reasoner
   resume/suspend calls go through `.venv/bin/rai reasoners ...` which
   uses the demo role via the profile + `--role` flag.
5. **At teardown**, the agent's `agent/deploy.py teardown` only un-registers
   the Cortex agent. To drop the demo DB itself, the user runs a
   one-line ACCOUNTADMIN `DROP DATABASE <DEMO_DB>; DROP ROLE
   RAI_DEMO_<DOMAIN>;` manually - the agent does not.

## Python environment

- Python 3.13, pinned in [.python-version](.python-version)
- Project venv at `.venv/` (create with `uv venv` if missing)
- Install packages with `uv add <pkg>` or `uv pip install <pkg>`
- Global CLIs (`snow`, `uv` itself) are not in the venv - they're system-wide
  via `uv tool install <pkg>` or pipx
- `relationalai` package (PyRel) - install with `uv add relationalai`
- `PyRel` source is checked out at `/Users/piotrkraus/rai-repos/PyRel` - use
  it as ground truth when public docs are thin

You will create `.venv/` early in Phase 2. Don't put it off.

## Repo layout (target - phase 1 starts empty)

```
.
├── CLAUDE.md                       # this file
├── INTAKE.md                       # the 5-question intake protocol
├── PIPELINE.md                     # the locked 9-phase pipeline
├── BRIEF.md                        # written after intake (does not exist yet)
├── BRIEF.template.md               # the brief format
├── BOOTSTRAP.template.sql          # the demo-role + DB bootstrap SQL (intake step 3)
├── REFERENCES.md                   # file-by-file pointers into the two demos
├── DEMO_QUESTION_CATALOG.md        # question archetypes per reasoner
├── RUNBOOK.template.html           # phase 8 deliverable
├── HANDOFF.template.md             # phase 9 deliverable
├── README.md                       # human-facing intro
├── .python-version
├── .gitignore
├── .claude/
│   └── settings.json               # marketplace + plugin + allowlists
├── rai_code/manual/                # ontology + queries + notebook live here
├── agent/                          # Cortex Agent deployment + queries catalog
├── data/                           # synthetic data generator + loader
└── build/                          # PyRel runtime cache (gitignored)
```

You will populate the empty directories as you move through phases. Do not
create files outside this shape without a reason - the reference demos
followed it strictly and it pays off in Phase 8 when you wire `prep_demo.py`.

## Style and tone

- No em-dashes in generated documents - the user dislikes them.
- No emojis in code or docs.
- Plain prose in markdown - minimal headers, no bullet-spam.
- When you have a choice between asking the user and inferring from
  `BRIEF.md` + the references, infer. The intake was
  there so you would not have to interrupt later.
- Truth over approval. If a demo idea is weak (no graph traversal, no
  optimisation, no rule-cascade), say so in `BRIEF.md`
  and propose a stronger one.

## When you finish a phase

1. Update the `TaskCreate` task for that phase to `completed`.
2. Append a one-paragraph "Phase N done - X" note in `BRIEF.md` under
   `## Phase log`.
3. Immediately start the next phase. Do not ask "ready to proceed?".

## When you get stuck

1. Check `/rai-health` (engine state) and `/rai-setup` (connection state) first.
2. Grep the two reference demos for the symptom. Almost everything you'll
   hit they've already hit.
3. Grep the PyRel checkout. In order:
   - `PyRel/example/<reasoner>/` for a runnable answer
   - `PyRel/tests/end2end/<area>/` for a test that exercises the same path
   - `PyRel/src/relationalai/semantics/` for the actual API signature
4. If the issue is path-query specific, read
   `supply_chain_demo/.claude/skills/rai-pathfinder/SKILL.md` end-to-end -
   it captures every footgun the marketplace skills don't cover.
5. If still stuck after 15 minutes, write the question in `BRIEF.md` under
   `### Open questions` and proceed with the next independent task. Surface
   the open questions in your end-of-session summary.

## End of file. Start with INTAKE.md.
