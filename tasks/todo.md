# data-layer plugin Phase 1 — PLAN (TDD+ PLAN entry, T2)

**Status:** [PROPOSED] — awaiting agent-architect persona + workspace rules staging; no RED started
**Date:** 2026-11-15
**Repo:** `claude-for-customer-success` (branch `claude/crm-data-normalization-layer-Cbmij`)
**Ask (verbatim):** "One concern raised, and that we partially address with skills, is data hygiene; since we will be pulling from at least 2-3 record sources I was considering whether it would be feasible to create a thin data normalization layer that the MCPs connected to the CRM, Sales, etc., feed into. For solo deployment we could keep it local with SQLite or the equivalent. We could run skills to clean the data if it is needed (and could write it back if allowed). We could also use it for dashboards, to add persistence and to house the outcome catalog & value registry"
**Elaboration (chat exchanges 2026-11-14 → 2026-11-15, folded into plan):**
- Source-data mapping + dry-run per source as part of config/startup
- Rule-based transforms/data quality checks; insulate from upstream schema changes
- Storage scope question — hybrid mirror (structured rows; bulky payloads reference-only)
- Deal→MSP→outcomes→renewal traceability; MSP as deal-derived materialized normalization path; revenue-observability write-point
- Supabase as the first-class cloud-hosted Postgres option (optional alongside self-hosted); lean-in to Auth + RLS + Realtime + Storage
**Plan artifact:** `docs/plans/crm-normalization-layer-plugin.md` (864 lines)
**Iteration boundary:** T2 — Phase 1 only this cycle; Phases 2 and 3 are planned-but-gated.

## Phase 1 scope (from the plan, Rollout Phases § Phase 1)

- 4 tables: `accounts`, `signals_snapshot`, `signals_history` (90d retention), `calls` (metadata only)
- 4 skills: `cold-start-interview`, `data-sync` (--dry-run default on first run), `data-hygiene-check`, `portfolio-view`
- Pre-wired consumers: `csm/managed-agents/health-watcher`, `csm/managed-agents/churn-signal-digest`, `csm/skills/account-research`, `csm/skills/renewal-readiness`, `csm/skills/call-prep`
- Backend: SQLite only (no ABC until Phase 2); file at `~/.local/share/cs-data-layer/cs.sqlite`, `chmod 600`
- Adapters: YAML specs for Salesforce, HubSpot, configured CS Platform, Gong (calls metadata)
- `inspect → map → dry-run → commit` loop on cold-start + every adapter YAML edit
- Adapter-rename auto-update: Levenshtein > 0.85 + same-type offers single-confirm YAML update
- Security baseline: provenance quintet, chmod 600, secret-scrub on ingest, sanitize freetext, audit-log dry-runs
- GDPR columns ship in P1 schema; delete-flow skills are stubs until Phase 2
- No write-back in Phase 1

## Blockers before RED can start

- [ ] **GOV-1** Workspace-level `CLAUDE.md` (parent of `agent-building/CLAUDE.md`) — needed for workspace folder semantic contract and L1 NEVER rules
- [ ] **GOV-2** Agent-architect persona — `agents/agent-agent-architect/CLAUDE.md` plus methodology/session-startup
- [ ] **GOV-3** Rule files: `rules/shared/session-start.md`, `rules/shared/compaction-mitigation.md`, `rules/shared/linear-sync.md`, `rules/shared/behavioral-defaults.md`, `rules/shared/tdd-plus-default.md`, `rules/agent-building/folder-contracts.md`, `rules/agent-building/never-rules.md`, `rules/agent-building/persona-load.md`
- [ ] **GOV-4** TDD+ spec: `tdd-plus-methodology/tdd-plus-development-methodology.md`
- [ ] **GOV-5** `_context.md` (project state) — bootstrap this session's entry
- [ ] **GOV-6** `context/INDEX.md` manifest (bootstrap)
- [ ] **GOV-7** Linear issue ID for this work (if applicable) — linear-sync anchor
- [ ] **AUTH-1** Explicit user authorization of the full plan scope (per workspace L1 NEVER: "NEVER let a subagent's recommendation expand the original task scope without explicit user authorization"). The plan grew from "data normalization layer" to 3 phases + Supabase + lifecycle spine — all with user input, but a session-start re-ground + explicit sign-off is still owed before any code.

## Assumptions (named, will re-verify after governance staged)

- A1 `data-layer/` is a new peer plugin under this repo (sibling of csm/, rev-ops/). Follows build conventions in `bin/manifests.yaml` and `bin/build-plugin.sh` (per the 2026-09-08 release precedent in `tasks/archive/todo-2026-09-08-cfcs-hooks-quoting-release.md`).
- A2 The plugin ships Python under `data-layer/lib/` — new convention for this repo (no existing plugin has Python). `bin/preflight.py` CHECKs 1–9 may need extension to cover Python modules; defer decision until RED.
- A3 Tests live in `data-layer/evals/` and run via `pytest`. The repo has no pytest harness today — bootstrap a minimal `pyproject.toml` or `conftest.py` at RED time.
- A4 SQLite backend path `~/.local/share/cs-data-layer/cs.sqlite` is the default; configurable in `data-layer/CLAUDE.md`. `chmod 600` enforced in `connect()`.
- A5 Branch `claude/crm-data-normalization-layer-Cbmij` is the target; no force-push, no touching `main`.
- A6 Attribution per session system reminder (Co-Authored-By: Claude Opus 4.7).

## TDD+ 11-stage decomposition for Phase 1 (gated — do not start without governance)

### 1. RESEARCH
- [ ] R1 Re-read `shared/cs-domain-model.md` §1 (health bands), §6 (source attribution), §G1–G7 (guardrails) — confirm P1 schema honors them
- [ ] R2 Re-read `shared/revops-domain-model.md` — any P1-relevant constraints
- [ ] R3 Enumerate the 5 pre-wired consumer skills' current MCP call patterns; identify the exact shell-out seam each one needs
- [ ] R4 Read `bin/preflight.py` CHECKs 1–9 — confirm the new plugin passes without new CHECKs; if a new CHECK is needed, scope it
- [ ] R5 Read `bin/manifests.yaml` schema + `bin/generate-manifests.py` — identify new entry shape for `data-layer`
- [ ] R6 Read `bin/build-plugin.sh --strict` + canonical frontmatter validator — identify any blocker for a Python-carrying plugin
- [ ] R7 Confirm `managed-agent-cookbooks/data-sync/` layout precedent against `value-map-system/`

### 2. DESIGN
- [ ] D1 Freeze the 4 P1 tables' DDL (columns, types, indexes) — write to `docs/specs/data-layer-phase1-schema.md`
- [ ] D2 Write the 4-step ingestion pipeline sequence as pseudocode — `docs/specs/data-layer-phase1-ingestion.md`
- [ ] D3 Write the `cache_helper.sh` contract (function signature, exit codes, JSON shape) — `docs/specs/data-layer-cache-helper.md`
- [ ] D4 Adapter YAML schema v1 (fields, transforms stub, quality stub) — `docs/specs/adapter-yaml-v1.md`
- [ ] D5 ADR-001: "Why SQLite-only in Phase 1 (no ABC yet)" — `docs/adr/ADR-001-sqlite-only-phase1.md`
- [ ] D6 ADR-002: "Why CLI shell-out is the integration seam (not MCP shim)" — `docs/adr/ADR-002-cli-seam.md`
- [ ] D7 ADR-003: "Phase 1 defers write-back, OCV import, GDPR-active, lifecycle spine" — `docs/adr/ADR-003-phase1-deferred.md`

### 3. PLAN (this file)
- [x] P1 todo.md decomposed (this document)

### 4. RED
- [ ] RED-1 `evals/test_schema_contract.py` — PRAGMA assertions for 4 tables + provenance quintet columns
- [ ] RED-2 `evals/test_identity_reconciliation.py` — two-phase upsert; HubSpot→Salesforce match-key collision resolves to one canonical account_id
- [ ] RED-3 `evals/test_adapter_conformance_salesforce.py` — seed JSON → upserted rows match expected normalized shape
- [ ] RED-4 `evals/test_adapter_conformance_hubspot.py` — same, HubSpot
- [ ] RED-5 `evals/test_secret_scrub.py` — poisoned fixture with fake AWS key → scrubbed in raw_blob, logged
- [ ] RED-6 `evals/test_sanitize.py` — `<script>` + SQL fragment in internal_notes → sanitized normalized column, raw_blob preserved
- [ ] RED-7 `evals/test_signals_history_retention.py` — backdate rows past 90d → prune deletes them, snapshot survives
- [ ] RED-8 `evals/test_cache_helper_contract.py` — Bash helper: cache hit returns row + confidence; miss returns JSON {hit: false, fallback: "mcp"}; 2s timeout → fallback
- [ ] RED-9 `evals/test_dry_run_in_memory.py` — dry-run runs full pipeline against in-memory SQLite, emits report, writes no persistent rows
- [ ] RED-10 `evals/test_file_permissions.py` — SQLite file forced to 0600 on connect; connect refuses if looser

### 5. GREEN
- [ ] G1 `data-layer/reference/schema.sql` — 4 tables + source_account_map
- [ ] G2 `data-layer/lib/sqlite_backend.py` — direct calls (no ABC)
- [ ] G3 `data-layer/lib/identity.py` — two-phase upsert
- [ ] G4 `data-layer/lib/ingestion.py` — pipeline orchestration
- [ ] G5 `data-layer/lib/adapter_executor.py` — YAML hydrate
- [ ] G6 `data-layer/lib/normalize.py` — health band + confidence stamp
- [ ] G7 `data-layer/lib/secret_scrub.py`
- [ ] G8 `data-layer/lib/sanitize.py`
- [ ] G9 `data-layer/lib/retention.py` — signals_history 90d prune
- [ ] G10 `data-layer/lib/cli.py` — inspect, dry-run, sync, get, prune verbs
- [ ] G11 `data-layer/lib/cache_helper.sh`
- [ ] G12 `data-layer/adapters/{salesforce,hubspot,gong}.adapter.yaml` + `adapter.schema.yaml`
- [ ] G13 `data-layer/skills/{cold-start-interview,data-sync,data-hygiene-check,portfolio-view}/SKILL.md`
- [ ] G14 `data-layer/CLAUDE.md`, `README.md`, `ARCHITECTURE.md`, `.mcp.json`, `.gitignore`
- [ ] G15 `data-layer/migrations/001_init.sql` + backfill-checker template
- [ ] G16 `data-layer/samples/{seed-crm,seed-csp,seed-calls}.json`
- [ ] G17 Pre-wire snippets in the 5 consumer skills (6-line cache_helper.sh source + miss fallback)
- [ ] G18 `managed-agent-cookbooks/data-sync/{agent.yaml, README.md, subagents/}`
- [ ] G19 `bin/manifests.yaml` — add data-layer v0.1.0 entry
- [ ] G20 `README.md` + badges — plugin 7, link row
- [ ] G21 `shared/cross-skill-registry.md` — 4 new commands
- [ ] G22 `CHANGELOG.md` — 0.1.0 entry

### 6. REFACTOR-QC
- [ ] QC1 Simplify helpers; remove dead branches; extract shared literals
- [ ] QC2 Type hints + docstrings where non-obvious
- [ ] QC3 Verify no backward-compat shims or dead code

### 7. REFACTOR-VX
- [ ] VX1 `bin/preflight.py --all .` passes (all CHECKs including any new one for Python)
- [ ] VX2 `pytest data-layer/evals/ -v` — all green
- [ ] VX3 `bin/build-plugin.sh` staged build passes `--strict` with canonical validator
- [ ] VX4 Extract archive; `hooks.json` + `plugin.json` parse; preflight PASS on extracted tree
- [ ] VX5 `bin/generate-manifests.py --check` clean

### 8. COMPLETE
- [ ] C1 Commit per `bin/commit-release-*.sh` convention: one commit for plugin scaffold, one for manifest/release, one for consumer-skill pre-wires
- [ ] C2 `git status` clean except known-untracked (tasks/, design/)

### 9. CAPTURE
- [ ] CAP1 Session entry in `_context.md`
- [ ] CAP2 Any lessons learned → `context/lessons/LESSON-<slug>-<date>.md`

### 10. DOCUMENT
- [ ] DOC1 `data-layer/README.md` final — install, cold-start, "how skills opt in" snippet
- [ ] DOC2 `data-layer/ARCHITECTURE.md` final — design rationale + "v1 is SQLite-only" caveat
- [ ] DOC3 Update `deployment-cookbooks/solo-csm-cookbook.md` — add §0.9 "Stand up the data layer"
- [ ] DOC4 Update `csm/skills/cold-start-interview/SKILL.md` — inheritance question for data-layer activation
- [ ] DOC5 `docs/reports/data-layer-phase1-gate-report.md` — gate evidence

### 11. JUDGEMENT
- [ ] J1 Walk each DOD rubric row below; produce PASS/FAIL per criterion with evidence pointer

### 12. ADVERSARIAL_REVIEW (terminal gate — mandatory per T2)
- [ ] AR1 Attack 1: "schema matches the plan" — diff `reference/schema.sql` against `docs/plans/crm-normalization-layer-plugin.md` schema sketch
- [ ] AR2 Attack 2: "Phase 1 scope discipline" — assert no Phase 2/3 tables/skills/code landed in commits
- [ ] AR3 Attack 3: "the 5 pre-wired consumer skills still work with cache absent" — delete SQLite file, run each skill, confirm MCP-fallback path exercises cleanly
- [ ] AR4 Attack 4: "ingestion sandboxing" — verify dry-run never writes to the real SQLite path (prove by watching inode + chmod state)
- [ ] AR5 Attack 5: "adapter YAML rot auto-update does not drift silently" — simulate a Levenshtein-close rename; confirm the single-confirm prompt fires

## DOD Rubric (dimensional)

### D1 Correctness of the Phase 1 cache
- [ ] REQUIRED: 4 P1 tables created with provenance quintet + audit/deletion trio — evidence: PRAGMA output
- [ ] REQUIRED: two-phase upsert resolves HubSpot+Salesforce for same account to one canonical account_id — evidence: RED-2 passes
- [ ] REQUIRED: signals_history 90d retention sweep removes aged rows, preserves snapshot — evidence: RED-7 passes

### D2 Build integrity
- [ ] REQUIRED: `bin/preflight.py --all .` exits 0 — evidence: output
- [ ] REQUIRED: `bin/build-plugin.sh --strict` builds `dist/data-layer-v0.1.0.plugin` — evidence: PASS line
- [ ] REQUIRED: extracted archive passes preflight independently — evidence: output

### D3 Integration safety (consumer pre-wires)
- [ ] REQUIRED: 5 pre-wired consumer skills unchanged behavior when `~/.local/share/cs-data-layer/cs.sqlite` absent — evidence: AR3 passes
- [ ] REQUIRED: cache_helper.sh 2s timeout → MCP fallback — evidence: RED-8 passes

### D4 Scope discipline
- [ ] REQUIRED: zero Phase 2/3 code lands (no DuckDB, no Postgres/Supabase, no lifecycle tables, no write-back, no OCV import) — evidence: AR2 passes
- [ ] REQUIRED: commits touch only `data-layer/`, `managed-agent-cookbooks/data-sync/`, `bin/manifests.yaml`, `README.md`, `CHANGELOG.md`, `shared/cross-skill-registry.md`, `deployment-cookbooks/solo-csm-cookbook.md`, the 5 pre-wired consumer skill files, `docs/`, `tasks/`, `context/` — evidence: `git show --stat` per commit

### D5 Security baseline
- [ ] REQUIRED: SQLite file enforced chmod 600 — evidence: RED-10 passes
- [ ] REQUIRED: secret-scrub redacts AWS/GitHub/Slack/OAuth/JWT patterns — evidence: RED-5 passes
- [ ] REQUIRED: sanitize escapes HTML + SQL fragments in freetext normalized columns — evidence: RED-6 passes
- [ ] REQUIRED: dry-run runs against in-memory SQLite only — evidence: RED-9 + AR4 pass

### D6 Governance compliance
- [ ] REQUIRED: GOV-1 through GOV-6 all staged before RED starts — evidence: files present at expected paths
- [ ] REQUIRED: AUTH-1 explicit user sign-off on scope captured in `_context.md` — evidence: `_context.md` entry
- [ ] REQUIRED: session-start protocol executed verbatim at every session start — evidence: `_context.md` entries

## Findings surfaced during PLAN (not acted on — outside Phase 1 scope)

- F1 The 2026-09-08 release precedent (`tasks/archive/todo-2026-09-08-cfcs-hooks-quoting-release.md`) used `bin/commit-release-*.sh` helper scripts committed-untracked. Same pattern will apply here.
- F2 No pytest harness exists in this repo today. Bootstrap may need `pyproject.toml` at root or inside `data-layer/`. Keep decision for RED stage.
- F3 `bin/preflight.py` CHECK 9 was added 2026-09-06 for non-scalar top-level frontmatter. Verify data-layer's SKILL.md frontmatter stays scalar.
- F4 `bin/build-plugin.sh --strict` could not validate skill-less plugins per 2026-09-08 lesson (handoff, auq-resilience); data-layer HAS skills so this should not block it. Verify at VX3.

## Review
_(deferred to COMPLETE + JUDGEMENT + ADVERSARIAL_REVIEW stages)_
