# PIPELINE.md - the locked 9-phase pipeline

Every RelationalAI demo built from this template moves through these nine
phases in order. Each phase has explicit **exit criteria**. You do not move
to the next phase until the current phase is green.

Phase boundaries should also be `TaskCreate` task boundaries - one task per
phase, marked `in_progress` when you enter and `completed` when exit criteria
are met. Phase N+1 is `blockedBy` Phase N.

The reference demos are at:
- `/Users/piotrkraus/rai-repos/supply_chain_demo` (the 10-question shape)
- `/Users/piotrkraus/rai-repos/airplanes_demo` (the 5-act shape - newer, more polished)

When a phase says "copy the pattern from X", treat X as ground truth.
[REFERENCES.md](REFERENCES.md) maps every file in both demos to its phase.

---

## Phase 1 - Position RelationalAI in the domain

**Goal.** Decide what makes this domain a RelationalAI demo (vs. a plain
SQL demo). Land on the question shapes that justify each reasoner.

**Steps.**
1. Read `BRIEF.md` (the intake output).
2. Invoke `/rai-discovery` (via Skill tool) and pass it the domain summary
   plus the reasoner mix from Q4 of the intake.
3. Reference [DEMO_QUESTION_CATALOG.md](DEMO_QUESTION_CATALOG.md) for the
   archetypes that work cleanly in each reasoner family.
4. Draft `DEMO_QUESTIONS.md` (use `airplanes_demo/DEMO_QUESTIONS.md` as the
   shape - plain-English questions, one per act, each labelled with the
   reasoner type and what makes it interesting).

**Exit criteria.**
- `DEMO_QUESTIONS.md` exists with N questions (N from intake Q3: 3, 5, or 10).
- Each question is tagged with its reasoner family.
- Each question has a one-sentence "why this is a RelationalAI question, not
  a SQL question" justification.
- At least one question per reasoner family the user picked in intake Q4.
- The set as a whole tells a coherent narrative (Act 1 sets up the problem,
  Act 2 reveals a hidden dependency, Act 3 ranks risk, Act 4 prescribes,
  Act 5 adds a rule and re-solves - adapt the arc to N).

**Anti-patterns.**
- A graph question with only one hop (that's a join).
- A prescriptive question with no real constraint (that's a sort).
- A persistent-rule question that doesn't change the answer when added.

---

## Phase 2 - Invent and load synthetic data into Snowflake

**Goal.** Create a defensible synthetic dataset, small enough to be fast
(target: each query under 30 s warm) and rich enough to make every Phase 1
question land. Load it into the SE Snowflake account.

**Steps.**
1. Set up the venv: `uv venv && uv add relationalai pandas plotly jupyterlab kaleido`.
2. Verify the snow CLI: `snow connection test -c rai`. If it fails, invoke
   `/rai-setup`.
3. Design the schema. Aim for the shape supply_chain used: ~15-20 tables, a
   mix of dimensions / masters / junctions / events / time. Not normalised
   to 3NF - leave the kind of redundancy a Snowflake schema actually has.
4. Hand-pick the "anchored numbers" - the specific named entities and counts
   the talk track will hinge on (supply_chain's "Suzhou battery shortage",
   airplanes' "KL1234 cascade of 6 flights", "KLG handler with 7 TOBT
   violations"). Bake these into the generator with `seed=42`.
5. Write the generator under `data/build_<domain>_data.py`. Make it
   reproducible.
6. Write the loader under `data/load_to_snowflake.sh` (idempotent - DROP
   TABLE IF EXISTS then CREATE then COPY INTO from stage).
7. Run the loader. Verify row counts.
8. Add Snowflake-native metadata: COMMENTs on database/schema/tables/columns,
   tags in `<DB>.META` for `DATA_DOMAIN`, `TABLE_ROLE`, `GRAIN`, `DEMO_AREA`.
   Pattern is in `supply_chain_demo/annotate_and_doc.py`.
9. Generate `DATA_DICTIONARY.md` from the Snowflake metadata.
10. Enable change tracking on every table (PyRel requires it):
    `ALTER TABLE <T> SET CHANGE_TRACKING = TRUE` for each table.

**Exit criteria.**
- `data/build_<domain>_data.py` runs end-to-end with `seed=42` and is idempotent.
- `data/load_to_snowflake.sh` succeeds from a clean database.
- Row counts match the generator's expected counts (assert this).
- The anchored numbers from Phase 1 reproduce as raw SQL - add the queries
  to `data/<domain>_demo_validation.sql` and assert their outputs.
- Change tracking is enabled on every source table.
- `DATA_DICTIONARY.md` is regenerated.

**Anti-patterns.**
- More than ~500K rows in the largest table (slow + costly).
- Synthetic identifiers like "Product 1, 2, 3" (industry experts will roll
  their eyes - use real-sounding SKUs / callsigns / claim numbers).
- A "perfect" dataset with no problem in it - Phase 1 questions need to
  have actual answers, including violations, cascades, and infeasibilities.

---

## Phase 3 - Build the PyRel ontology

**Goal.** Translate the Snowflake schema into a RelationalAI model with
concepts, properties, relationships, subtypes, and derived properties.

**Steps.**
1. Invoke `/rai-build-starter-ontology` and point it at the loaded tables.
   This produces a first-pass `rai_code/manual/<domain>.py`.
2. Invoke `/rai-ontology-design` to enrich: add subtypes (e.g.
   `Departure(Flight)`, `Arrival(Flight)`), add derived relationships
   (e.g. `single_sourced(Product)`, `TOBTViolation(Flight)`).
3. If you'll add a persistent-rule act (Phase 4 Q5), declare the concept
   that the rule will hang off - see `PreservedFlight(Flight)` in
   `airplanes_demo/rai_code/manual/eham_acdm.py`.
4. Copy the `_build_config()` pattern from
   `supply_chain_demo/rai_code/manual/supply_chain.py` - it auto-detects
   Snowsight vs. local and uses named engines.
5. Run a smoke test that just loads the model and prints `inspect.schema(model)`.

**Exit criteria.**
- `rai_code/manual/<domain>.py` imports cleanly with `.venv/bin/python -c "from rai_code.manual import <domain>; print(<domain>.model)"`.
- All Snowflake tables from Phase 2 are mapped to a concept.
- Junction tables become concepts with relationships, not just two
  reference properties - pattern from `SupplierProduct` /
  `BomEntry` / `Lane` in supply_chain.
- At least 3 derived properties / relationships exist (rules that the
  ontology computes, not just data it stores).
- Both engines (`<domain>_logic_l`, `<domain>_prescriptive_m`) resume to
  READY without error.

**Anti-patterns.**
- One concept per table verbatim with no enrichment - that's just a SQL view.
- Using `relationalai.connect_sync(...)` directly (low-level; no SQL executor).
  Always go through `from relationalai.semantics import Model`.

---

## Phase 4 - Author demo queries

**Goal.** Write one PyRel query per Phase 1 question, executable locally.

**Steps.**
1. **Invoke `/rai-querying` first.** Mandatory - it's the syntax authority
   for the current PyRel version. Your training-data knowledge is stale.
2. For each question, identify the reasoner family (rules / graph /
   heuristic / prescriptive / persistent rule) and invoke the matching skill:
   - Rules → `/rai-rules-authoring`
   - Graph → `/rai-graph-analysis`
   - Heuristic → `/rai-querying` (no separate skill - express as derived properties)
   - Prescriptive → `/rai-prescriptive-problem-formulation` then `/rai-prescriptive-solver-management`
   - Persistent rule → `/rai-rules-authoring` (the rule lives on the model, the query is a re-solve)
3. Write each query as a top-level function in `rai_code/manual/demo_queries.py`,
   named `qN_<slug>()` and returning a `pandas.DataFrame`.
4. For prescriptive queries, also return the solver status - needed by
   `/rai-prescriptive-results-interpretation` and by the Phase 7 agent.
5. Add a `main()` that runs all queries top-to-bottom and prints shape
   + first 3 rows of each. This is your smoke test.

**Exit criteria.**
- `.venv/bin/python rai_code/manual/demo_queries.py` runs end-to-end from
  the project root with the engines warm.
- Each query returns a non-empty DataFrame.
- The anchored numbers from Phase 1 / Phase 2 reproduce in the query
  outputs (assert them in `main()`).
- LP / MIP queries return `OPTIMAL` status. If `INFEASIBLE`, root-cause via
  `/rai-prescriptive-results-interpretation` and fix the formulation -
  do not move on with a broken solver.
- No pandas in the query logic itself (sort / format on DataFrame is fine).
  All filtering / aggregation expressed in PyRel.

**Anti-patterns.**
- Querying with stale syntax - `/rai-querying` exists because the API
  changes between PyRel minor versions.
- Hardcoded magic numbers from Phase 2 - read them from the schema.

---

## Phase 5 - Local Jupyter notebook with Plotly visualisations

**Goal.** A `.ipynb` that runs all queries with narrative markdown between
them and produces a Plotly figure for each.

**Steps.**
1. Create `rai_code/manual/<domain>_demo.ipynb`.
2. Cell structure (from `airplanes_demo/rai_code/manual/eham_acdm_demo.ipynb`):
   - Markdown: intro (title, table of acts, runtime estimate).
   - Code: imports + the PyRel event-loop workaround (`nest_asyncio.apply()`
     or similar - see the reference).
   - Per question: 4 cells - a markdown lead-in, the query call, a Plotly
     viz, a markdown "what to look at".
   - Markdown: closing.
3. Charts: use `plotly.express` for bars / scatters and `plotly.graph_objects`
   for networkx-backed graphs (Phase 1 graph question).
4. Notebooks must run top-to-bottom without manual intervention. Use
   `papermill` or `jupyter nbconvert --execute` to verify.

**Exit criteria.**
- `.venv/bin/jupyter nbconvert --execute --to notebook --inplace rai_code/manual/<domain>_demo.ipynb` succeeds.
- Every query has a corresponding Plotly figure.
- Markdown cells are not empty placeholders - they're the talk track in
  miniature.

---

## Phase 6 - Snowsight notebook

**Goal.** The same notebook, but inside Snowflake.

**Steps.**
1. Same `<domain>.py` + `demo_queries.py` (no edits). The `_build_config()`
   pattern detects the Snowsight session.
2. Build a Snowsight-flavoured `.ipynb`. Differences from local:
   - No `nest_asyncio` (Snowsight handles it).
   - Inline `from snowflake.snowpark.context import get_active_session`
     in the first cell.
   - Plotly works the same but figure sizes need explicit width / height.
3. Upload to the SE account:
   - Stage: `<DB>.NOTEBOOKS.<DOMAIN>_NOTEBOOK_STAGE`
   - Notebook: `<DB>.NOTEBOOKS.<DOMAIN>_DEMO`
   - Files: notebook + `<domain>.py` + `demo_queries.py` under a subfolder
     so Snowsight's Files view shows them as a workspace.
4. `snow notebook execute <DB>.NOTEBOOKS.<DOMAIN>_DEMO` to run it remotely
   and verify it works in Snowsight too.

**Exit criteria.**
- The notebook is uploaded and visible in Snowsight Files at
  `<DB>.NOTEBOOKS.<DOMAIN>_DEMO`.
- `snow notebook execute` runs it end-to-end without errors.
- Result figures appear inline.

**Skip if** intake Q3 was "Just the ontology + notebook".

---

## Phase 7 - Snowflake Intelligence (Cortex) agent

**Goal.** Deploy a Cortex agent so a non-technical audience can ask the
demo questions in natural language inside Snowsight.

**Steps.**
1. Invoke `/rai-cortex-integration`. It scaffolds the `agent/deploy.py`
   pattern - copy and adapt from `airplanes_demo/agent/deploy.py`.
2. Write `agent/queries.py` as the `QueryCatalog`. One function per Phase 4
   query plus `_chart` variants where useful - see
   `airplanes_demo/agent/queries.py`. The `_chart` wrapper attaches a
   `chart_hint` dict that lets the agent suggest "click the chart icon to
   visualise this as a bar / scatter / etc.".
3. Configure:
   - Agent name: `<domain>` (lowercase)
   - Database: `<DB>` (the demo database)
   - Schema: `RAI_AGENT`
   - Warehouse: `RAI_XS`
4. `.venv/bin/python -m agent.deploy deploy` to register.
5. Smoke test: `.venv/bin/python -m agent.deploy chat "What's the answer to question 1?"`. Round-trip should be ~60-90 s warm.
6. Test each demo question via chat. Expect 60-90 s for rules / graph /
   heuristic; 2-3 min for prescriptive (the LP solve dominates).

**Exit criteria.**
- `.venv/bin/python -m agent.deploy status` reports the agent deployed.
- All N demo questions answer correctly via `chat` with reasonable latency.
- Chart hints render in Snowsight (manual visual check).

**Skip if** intake Q3 was "Just the ontology + notebook".

---

## Phase 8 - `prep_demo.py` gate + `RUNNING.html` runbook

**Goal.** The single pre-flight script the user runs 10 minutes before a
live demo, and the speaker-facing static runbook.

**Steps.**
1. Copy `airplanes_demo/prep_demo.py` and adapt:
   - Replace `acdm_logic_l` / `acdm_prescriptive_m` with the demo's engines.
   - Replace `ACDM_DEMO.EHAM` with `<DB>.<SCHEMA>`.
   - Replace the anchored-number SQL checks with this demo's anchored numbers.
   - Replace the smoke-test sentinels with values you know reproduce.
2. Run `prep_demo.py` cold (engines suspended). Verify it suspends-resumes-runs
   green end-to-end. Budget ~8 minutes cold, ~6 minutes warm.
3. Build `build/generate_demo_figures.py`. One PNG per query, saved to
   `build/figures/`. Use `kaleido` for static export. Patterns in
   `airplanes_demo/build/generate_demo_figures.py`.
4. Write `RUNNING.html` from the template at
   [RUNBOOK.template.html](RUNBOOK.template.html). Embed each figure either
   as a base64 data URI or as a relative path to `build/figures/`.
5. Wire `prep_demo.py --skip-figures` to skip step 3 for fast iteration.

**Exit criteria.**
- `.venv/bin/python prep_demo.py` finishes with all checks GREEN.
- Cold-start (both engines suspended) completes in under 10 minutes.
- `RUNNING.html` opens in a browser and shows every embedded figure.
- The "anchored numbers" section in `RUNNING.html` matches the talk track.

---

## Phase 9 - Talk track + handoff briefing

**Goal.** The two markdown documents that let the next human or agent run
the demo without you.

**Steps.**
1. Write `<DOMAIN>_TALK_TRACK.md`. Shape: opening pitch (90 s), Act-by-act
   speaker beats with expected outputs and timing, fallback notes for when
   something is slow / breaks. Pattern from
   `airplanes_demo/A-CDM-Decision-Hub-Talk-Track.md`.
2. Write `SNOWSIGHT_DEMO.md` if Phase 7 happened. Shorter version (3
   questions, agent UI focus). Pattern from `airplanes_demo/SNOWSIGHT_DEMO.md`.
3. Write `HANDOFF_BRIEFING.md` from the template at
   [HANDOFF.template.md](HANDOFF.template.md). Include: domain choice
   rationale, key design decisions made + why, user preferences observed
   during the run, anchored numbers, known limitations, where to look when
   things break.
4. Commit (do NOT push - `git push` is denied).

**Exit criteria.**
- All three markdown files exist and are non-empty.
- A fresh reader can run the demo from `RUNNING.html` + the talk track
  without asking you anything.
- `git status` shows the repo is clean except for the build cache and venv.

---

## When you finish all phases

1. Mark all 9 tasks `completed`.
2. Write a final summary in chat (not in a file): what you built, how long
   each phase took, what surprised you, what you'd improve.
3. Stop. Do not start another loop.
