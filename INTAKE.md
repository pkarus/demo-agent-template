# INTAKE.md - the 5-question intake protocol

You are the agent. The user has just told you "start a new RelationalAI demo"
(or something like it) and `BRIEF.md` is missing.

Run this intake before doing anything else. Do not ask the user about
RelationalAI itself - that's your job to figure out. Ask only the things
you cannot infer.

## Hard rules

- Use the `AskUserQuestion` tool. Do not type questions in plain text.
- Send all 5 questions in **one** `AskUserQuestion` call so the user sees them
  as a single intake form, not five drip-fed turns.
- After answers come back, write `BRIEF.md` (see template in
  [BRIEF.template.md](BRIEF.template.md)) and immediately move to Phase 1 of
  [PIPELINE.md](PIPELINE.md). Do not ask permission to proceed.
- If the user provided a PDF, CSV, or other attachment in the same message,
  read it first and let it pre-fill answers - only ask what's still unclear.

## The 5 questions

Send these together via `AskUserQuestion`. Wording can be tightened but the
information they elicit must not change.

### 1. Domain (header: "Domain")
*What's the domain of the demo, in one phrase?*

Open-text. Examples: "pharma cold-chain logistics", "retail markdown
planning", "telco network capacity planning", "insurance claims fraud",
"investment portfolio rebalancing". You will turn this into the model name,
database name, agent name, and the talk-track frame.

Offer 3 short options that span common verticals plus an "Other" path:
- Supply chain / logistics
- Financial services / risk
- Healthcare / pharma

(The user can override with Other - that's the point.)

### 2. Inputs available (header: "Inputs", multiSelect: true)
*What did you bring with you?*

Options:
- A real or representative schema (CSV/DDL/PDF) - pre-load it before generation
- A problem statement document (PDF/markdown) - use it to seed scenarios
- Nothing - invent the data from domain knowledge
- An existing Snowflake database I want to demo against

This routes Phase 2 (data invention vs. real-schema-shaping vs.
demo-against-existing-tables).

### 3. Demo length and depth (header: "Length")
*How long is the demo and how deep do we go?*

Options:
- 10 minutes / 3 questions - Snowsight-only, rules + graph + one solver
- 20 minutes / 5 acts - full reasoner sweep (the airplanes_demo shape)
- 30+ minutes / 10 questions - broad survey (the supply_chain_demo shape)
- Just the ontology + notebook (no Cortex agent, no runbook)

This determines how many questions you author in Phase 4 and whether Phases
6, 7, 8, 9 happen at all.

### 4. Reasoner mix you want to showcase (header: "Reasoners", multiSelect: true)
*Which RelationalAI reasoners must appear?*

Options:
- Rules (derived properties, classification, validation)
- Graph (reachability, cascades, paths)
- Heuristic / scoring (deterministic fit-score over the ontology)
- Prescriptive (LP / MIP via HiGHS or Gurobi)
- Persistent rule (operator adds a rule, ontology re-solves with it)
- Predictive (GNN - early access; only if explicitly requested)

Pre-select Rules + Graph + Prescriptive as the default - that's the
combination both reference demos use and the one that lands cleanest.
Predictive (GNN) is preview; don't add it unless asked.

### 5. Audience (header: "Audience")
*Who is the live audience on demo day?*

Options:
- Snowflake SE peers (technical, RAI-familiar)
- Snowflake customer prospects (technical, RAI-curious)
- Industry / domain experts (skeptical of synthetic data)
- Executive / non-technical decision makers

This drives the level of domain rigor (the airplanes_demo `HANDOFF_BRIEFING.md`
explains how skeptical-domain-expert framing forced real ICAO milestone names
and published rate numbers - repeat that pattern when answer is "industry
experts").

## After answers come back

1. **If any answer is ambiguous and would block Phase 1**, ask one focused
   follow-up via `AskUserQuestion` - only one. If the user typed "Other" and
   gave a custom domain, that's not ambiguous; proceed.

2. **Write `BRIEF.md`** by copying [BRIEF.template.md](BRIEF.template.md) and
   filling in the answers plus your derivations:
   - Model name: snake_case version of domain (e.g. "cold_chain_logistics")
   - Database name: `PK_<DOMAIN>` uppercased (matching supply_chain's `PK_SUPPLY_CHAIN`)
   - Agent name: same as model name
   - Engine names: `<model>_logic_l` and `<model>_prescriptive_m`
   - Schema name: `RAI_AGENT`
   - Pre-stated "anchored numbers" placeholder section that Phase 2 will fill

3. **Pitch the demo back to the user** in 4-6 sentences in chat (not in
   `BRIEF.md`). Cover: what the data will look like, the 5 (or 3 or 10)
   questions you intend to author, which reasoner each maps to, and the one
   thing you're least sure about. This is the only place in the whole run
   you check in with the user. Tell them you'll proceed unless they object
   in the next reply.

4. **Create a `TaskCreate` task list** with one task per phase from
   [PIPELINE.md](PIPELINE.md), in order, with the right `blockedBy`
   dependencies (Phase N+1 blocked by Phase N).

5. **Mark Phase 1 in_progress** and start.

## Don't ask

Do not ask any of the following in the intake - they're either pre-decided
by the template or things you should figure out yourself:

- Snowflake account / warehouse / role (always `ajb85638` / `RAI_XS` / `RAI_DEVELOPER`)
- Whether to use `uv` (yes)
- Where to put files (the layout in [CLAUDE.md](CLAUDE.md) is fixed)
- Whether to deploy a Cortex agent (depends on Q3; don't double-ask)
- Whether to use named engines (yes)
- What to call concepts / properties (your job; copy patterns from references)
- Whether the user wants a runbook (yes, if Q3 includes anything beyond
  "Just ontology + notebook")

## End of intake protocol.
