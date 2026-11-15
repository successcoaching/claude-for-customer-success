# data-layer plugin, Phase 1: Continuation Brief

**Status:** [DRAFT] (snapshot 2026-11-15; not yet carried across a take)
**Lane (tentative slug):** `data-layer`. The outgoing session has no roster access in this container; correct this slug against `roster.py show` on your Mac before the handoff goes live.
**Repo:** `claude-for-customer-success`
**Branch:** `claude/crm-data-normalization-layer-Cbmij` (HEAD `ecfdb88`)
**Workspace target:** `/Users/toddceby/claude-cowork/projects/agent-building/claude-for-customer-success/`

---

## 1. How to start the next session

1. Open a Claude Code desktop window on the Mac inside `~/claude-cowork/projects/agent-building/`.
2. Read the root router: `./CLAUDE.md` (project-level) and its parent workspace `CLAUDE.md`. Load the lead persona per the `Lead Agent` section, which points at `agents/agent-agent-architect/CLAUDE.md`.
3. Claim coord: `python3 ~/claude-cowork/coordination/bin/coord.py claim --lane data-layer` (adjust lane slug if the roster uses a different value).
4. Read, in order: `claude-for-customer-success/_context.md`, `claude-for-customer-success/context/INDEX.md`, this brief, then `claude-for-customer-success/tasks/todo.md`.
5. First file to open after re-grounding: `claude-for-customer-success/docs/plans/crm-normalization-layer-plugin.md` sections "Rollout phases" and Phase 1 details, then `claude-for-customer-success/tasks/todo.md` Blockers.

## 2. Read this brief as a snapshot, not as truth

Every claim below is the outgoing session's best reading of the state at commit `ecfdb88`. Before building on any of it, re-ground against (a) the live working tree, (b) `tasks/todo.md` checkboxes, and (c) `roster.py show` for the lane's current fields. A wrong carried assumption costs a whole session; one grounding pass costs minutes.

## 3. Landed state (DONE, by commit hash + path)

- `ecfdb88 chore(data-layer): bootstrap agent-building governance scaffolding for Phase 1`
  - `_context.md` (repo root): current state, session history, re-ground notes with original ask verbatim
  - `tasks/todo.md`: Phase 1 TDD+ PLAN, 11 stages, DOD rubric, blockers listed
  - `tasks/archive/todo-2026-09-08-cfcs-hooks-quoting-release.md`: prior VALIDATED release PLAN preserved
  - `docs/plans/crm-normalization-layer-plugin.md`: 864-line 3-phase plan (Phase 1 + Phase 2 + Phase 3 + Supabase cloud path)
- `context/INDEX.md` written on disk but gitignored per repo convention (`.gitignore:35 context/`). Useful; not committed.
- Plan iteration history (not in a commit, lives in prior chat threads): panel review by four adversarial reviewers (minimalist, buy-vs-build, boundaries, operator) folded in as phased rollout; three rounds of elaboration on transforms/quality rules, hybrid storage model, lifecycle spine, Supabase migration, and MSP-as-normalization-path.

## 4. In flight (mid-way; next edit)

Nothing in a code file is mid-way. The lane is **pre-RESEARCH**. The exact next edit, once blockers clear, is tasks/todo.md stage R1: re-read `shared/cs-domain-model.md` sections 1, 6, and G1-G7, and confirm the 4 Phase 1 tables honor the health band + source attribution + guardrails before writing the DDL.

## 5. Open waits

- **GOV-1..6 staging** on the user. Workspace-level `CLAUDE.md`, agent-architect persona, 8 rule files, TDD+ spec, prior-session `_context.md` + `context/INDEX.md` from the Mac. Upload paths given in the 2026-11-15 chat.
- **AUTH-1** on the user. Explicit scope authorization. The plan grew from the original "thin SQLite layer" ask to 3 phases + Supabase + lifecycle spine via iterative chat. Per the workspace L1 Orchestration Discipline NEVER rule, RED cannot start without a "yes, Phase 1 scope as written in `tasks/todo.md` is authorized" acknowledgement, captured in `_context.md`.

## 6. Hard-won gotchas (one sentence each)

- G1. This repo's prior session pattern commits `tasks/todo.md` as tracked (not the archived note's assumption of untracked). Check `git log -- tasks/todo.md` before deciding whether an edit needs `git add`.
- G2. `context/` is in `.gitignore` (line 35). `context/INDEX.md` lives on disk but never ships in a commit; it needs to be rsync'd or re-created on the receiving side.
- G3. The `agent-building/CLAUDE.md` NEVER rule "no writing agent YAML manifests to `output/`" applies to this build too: the `managed-agent-cookbooks/data-sync/` cookbook belongs under `managed-agent-cookbooks/`, not `output/`.
- G4. Remote origin moved to `successcoaching/claude-for-customer-success` but pushes still succeed against `t0ddc3by/claude-for-customer-success`. When the new session pushes, watch for the same "repository moved" notice; it is informational, not an error.
- G5. The 2026-09-08 release precedent shows one-commit-per-logical-change on this repo; follow that shape when Phase 1 lands (scaffold, release manifest, consumer pre-wires as separate commits).
- G6. `bin/preflight.py` CHECK 9 fails on non-scalar top-level frontmatter in a SKILL.md. Keep the 4 new SKILL.md files' frontmatter scalar.

## 7. Hard constraints (lane standing rules)

- No AI attribution on commits (per `/continue-lane` working contract). The scaffolding commit `ecfdb88` already shipped with `Co-Authored-By` lines; do not repeat that on future commits in this lane.
- Coordinate through the roster + coord scripts; no out-of-band moves.
- Never stage a file this session did not create or edit.
- Never write to a database file in the repo working tree (no `.sqlite` files in commits; `.gitignore` covers this).
- Never print env values, keys, or secrets.
- Branch policy for this session: develop on `claude/crm-data-normalization-layer-Cbmij`; push only to that branch; no force-push; do not touch `main`.
- Lane-specific: scope discipline. Phase 1 only this iteration. No DuckDB code, no Postgres or Supabase code, no lifecycle-spine tables, no write-back, no OCV import. Any drift is a scope expansion that needs a new AUTH.
- Lane-specific: TDD+ ordering. RED tests precede GREEN code for every item in `tasks/todo.md` sections 4 and 5.
- Working contract: TL;DR on every update; rulings only through AskUserQuestion with four options and one marked Recommended; journal rulings before putting them; no em dashes anywhere; status signals on every artifact; task list stays current; roster `touch` at every handoff; coord claim on session start.

## 8. Baton

**Not yet offered.** The outgoing session is in a remote cloud container with no `~/claude-cowork/coordination/bin/roster.py` available; `roster.py handoff --session self --to-kind <kind> --confirm <lane>` cannot run from here. The user runs it on their Mac to produce the actual baton block (token, offered time, kind). Until that block exists, this brief stands in as the state half of the handoff; the mechanical half is deferred.

When the user runs `handoff` on their Mac, paste the block's title + BATON + Take + Name + Confirm lines into Section 8 here, then into the kickoff prompt below in place of the placeholder.

## 9. Suggested first step

Read this brief, then re-ground: `git log --oneline -5`, `cat tasks/todo.md | head -60`, `roster.py show` for this lane, `coord.py status`. Then TL;DR back to Todd covering (a) current stage, (b) blockers that remain, (c) proposed first action under the TDD+ cycle, (d) one AskUserQuestion with four options for how to proceed if more than one path is open.
