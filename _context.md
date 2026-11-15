# _context.md — data-layer plugin build

**Project:** claude-for-customer-success — data-layer plugin (new peer, Phase 1)
**Workspace (eventual home on user's machine):** `/Users/toddceby/claude-cowork/projects/agent-building/claude-for-customer-success/`
**This session's working directory:** `/home/user/claude-for-customer-success/` (remote execution container; cloned fresh)
**Branch:** `claude/crm-data-normalization-layer-Cbmij`

---

## Current state (2026-11-15, session start)

**Stage:** Pre-RESEARCH — governance scaffolding in flight.
**Last completed step:** Phase 1 PLAN decomposition written to `tasks/todo.md`.
**Next action:** Wait for user to stage GOV-1 through GOV-6 (workspace-level CLAUDE.md, agent-architect persona, rules/, TDD+ spec, _context.md's first real session entry, context/INDEX.md). RED cannot start until governance is present.
**Open blockers:**
- GOV-1..6: governance files not yet in-repo (user to stage via upload or commit)
- AUTH-1: explicit user sign-off on the full plan scope, captured here

**Plan artifact:** `docs/plans/crm-normalization-layer-plugin.md` (864 lines, seeded from user upload 2026-11-15)
**Plan status:** [DRAFT] — ask-grew-via-conversation; needs session-start re-ground + explicit scope authorization per workspace L1 NEVER rules.

---

## Session history

### 2026-11-15 — Session 1 (this session)
- User uploaded `agent-building/CLAUDE.md` revealing governance model (TDD+, folder contracts, L1 NEVER rules, agent-architect persona)
- User uploaded the plan file (`crm-normalization-layer-plugin.md`, 864 lines) covering Phase 1 cache + Phase 2 warehouse + Phase 3 lifecycle spine + Supabase cloud transition
- Agent re-grounded per NEVER-rule § Orchestration Discipline: flagged that the plan scope has grown substantially from the original "CRM data normalization layer" ask
- User authorized "I produce artifacts in agent-building folder layout *inside this repo*"
- User chose "Pause — you'll set up governance first" for proceed-mode
- Agent created agent-building folder layout inside this repo (`docs/{plans,specs,adr,reports,reference}/`, `context/`, `tasks/archive/`)
- Agent archived prior session's PLAN (2026-09-08 CfCS hooks-quoting release, VALIDATED) to `tasks/archive/todo-2026-09-08-cfcs-hooks-quoting-release.md`
- Agent placed plan at `docs/plans/crm-normalization-layer-plugin.md`
- Agent wrote this session's PLAN (`tasks/todo.md`) decomposing Phase 1 into TDD+ stages; marked [PROPOSED] pending governance
- Agent wrote this `_context.md` and `context/INDEX.md`
- Agent stopped here, awaiting governance staging

**Note on prior-conversation history:** This is a continuation of prior conversation threads where the plan was built iteratively through panel reviews (minimalist, buy-vs-build, boundaries, operator) and three rounds of elaboration on transforms/quality rules, storage model, lifecycle traceability, Supabase migration, and MSP-as-normalization-path. Those exchanges happened before the agent-building governance was visible; the plan file captures their output.

---

## Re-ground notes (per workspace L1 NEVER rules)

**Original ask (verbatim, from prior session):**
> "One concern raised, and that we partially address with skills, is data hygiene; since we will be pulling from at least 2-3 record sources I was considering whether it would be feasible to create a thin data normalization layer that the MCPs connected to the CRM, Sales, etc., feed into. For solo deployment we could keep it local with SQLite or the equivalent. We could run skills to clean the data if it is needed (and could write it back if allowed). We could also use it for dashboards, to add persistence and to house the outcome catalog & value registry"

**Plan delta vs original ask (all with user input through iterative elaboration — but sign-off still owed):**
- Original: thin SQLite layer, skills clean data, write-back if allowed, dashboards, OCV + value registry housing
- Plan now adds: 3-phase rollout (SQLite → DuckDB → Postgres/Supabase); Phase 3 lifecycle spine (deals, discovery, MSP, signoffs, handoff, lifecycle events, lineage traces, revenue_attribution); source-mapping inspect + dry-run; transforms + quality-rules engine; hybrid storage model (structured mirror vs reference); Supabase cloud deployment path with Auth/RLS/Realtime/Storage; MSP-as-materialized-normalization-path

**Reflection checkpoint owed before COMPLETE:** original ask was X (thin SQLite layer + skills + dashboards + OCV housing); output must be Y (Phase 1 ships exactly that; Phases 2 and 3 are planned-but-gated). Keep Y = X for Phase 1.

---

## Session-start checklist (to execute at every resume)

Per `agent-building/CLAUDE.md` § Session Resume Protocol:

1. Read `_context.md` (this file)
2. Read `context/INDEX.md`
3. Read `tasks/todo.md`
4. Load the lead agent persona per the **Lead Agent** section of `agent-building/CLAUDE.md` (requires GOV-2 staged first)
5. State aloud: current task, last completed step, next action — before doing any work
