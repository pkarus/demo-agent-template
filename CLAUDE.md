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

## RelationalAI skills are pre-loaded

The plugin `rai@RelationalAI` is registered in [.claude/settings.json](.claude/settings.json)
and provides 15 slash-skills you should reach for at the right phases:

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

**`/rai-querying` is mandatory at Phase 4** - its SKILL.md says: *"Load this
BEFORE writing any PyRel query, even your first one - your prior knowledge of
the syntax is likely stale."* Treat that as binding.

Skills auto-load on session start, but invoke them explicitly when you enter
each phase using the `Skill` tool - that signals progress to the user.

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
- Role: `RAI_DEVELOPER` (granted) for PyRel; `ACCOUNTADMIN` for DDL on the demo database
- Native App: RelationalAI is installed on this account
- PyRel auto-discovers credentials from `~/.snowflake/connections.toml` - no `raiconfig.yaml` needed
- Engine sizing convention from the reference demos:
  - Logic engine: `<domain>_logic_l` at `HIGHMEM_X64_L` (rules + graph + heuristic)
  - Prescriptive engine: `<domain>_prescriptive_m` at `HIGHMEM_X64_M` (LP / MIP)
  - Both are **named** engines so they persist across runs

The exact `_build_config()` pattern that works in both local Python AND inside
a Snowsight notebook is in `supply_chain_demo/rai_code/manual/supply_chain.py`
- copy it.

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
3. PyRel source at `/Users/piotrkraus/rai-repos/PyRel` is the authoritative
   reference for API signatures.
4. If still stuck after 15 minutes, write the question in `BRIEF.md` under
   `### Open questions` and proceed with the next independent task. Surface
   the open questions in your end-of-session summary.

## End of file. Start with INTAKE.md.
