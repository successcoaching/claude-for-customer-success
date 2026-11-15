# Plan — `data-layer/` plugin: thin normalization layer for CRM/CS data

## Context

The suite pulls from 2–3 record sources per skill (CRM, CS Platform, call recording, document storage) via per-plugin `.mcp.json` connectors. There is no shared cache, no queryable mirror, and no enforced normalization at ingestion — every skill talks to MCPs directly, applies domain-model rules at read time, and writes its outputs as YAML Value Maps or Markdown artifacts.

That works for one-account-at-a-time skill calls, but it strains in three places:

1. **Data hygiene** — the same `account_id` can mean different things across Salesforce/HubSpot, Gainsight/Totango, Gong; staleness, confidence, and source attribution are reapplied skill-by-skill rather than enforced once at ingestion.
2. **Dashboards & cross-account queries** — `health-watcher` and `churn-signal-digest` re-pull every account every cycle; there is no place to run "show me all Red accounts with a renewal in 60 days" without N MCP calls.
3. **Persistence for the OCV and Value Registry** — currently Markdown + YAML; queryable joins ("which accounts have realized value against outcome OCV-003?") are not possible.

This plan introduces a **thin `data-layer/` plugin** that sits between MCPs and skills as a **read-through cache + queryable mirror**. Source-of-truth ownership does not change: YAML Value Maps remain canonical for account state; Markdown OCV remains canonical for the outcome catalog. SQLite is the default backend for solo deployments; a pluggable backend interface allows team deployments to swap in Postgres later (a v2 effort — not v1).

---

## Decisions (confirmed with user)

| Question | Decision |
|---|---|
| Role of the layer | **Cache + query mirror** (YAML/Markdown stay canonical) |
| Deliverable | **Working reference implementation** (code + schema + sample dashboard) |
| Placement | **New top-level plugin** `data-layer/` (peer of csm, rev-ops, etc.) |
| Scale target | **Solo-first with pluggable backend** (SQLite default; ABC is a forward contract for v2) |
| Rollout | **Phased — see Rollout Phases below.** Phase 1 ships in v1; Phases 2 and 3 are designed but gated on demonstrated demand. |

---

## Rollout phases (adversarial review folded in)

An adversarial review by four expert reviewers (pragmatist/minimalist, buy-vs-build platform engineer, boundaries architect, operator/CS leader) converged on a single critique: **the original scope is wrong-sized for a solo CSM tool.** All four independently proposed roughly the same Phase 1; three of four argued the revenue lifecycle spine should not ship in v1 because the source artifacts (discovery findings, signed MSPs, customer signoffs linked to renewals) don't structurally exist for ~70% of SaaS accounts.

The plan responds by **phasing the rollout** rather than discarding the design. Phases 2 and 3 keep their existing architecture (described in the sections below) but are explicitly gated on Phase 1 demand evidence. This protects against the "build a spine for data that doesn't exist" failure mode while preserving the design for the day it's earned.

### Phase 1 — Read-through cache and the smallest queryable mirror (ship in v1)

Smallest thing that materially helps. Adoption wedge is the two highest-frequency consumers (`health-watcher`, `churn-signal-digest`) that re-pull the full portfolio every cycle today.

**Tables (4):** `accounts`, `signals_snapshot`, `signals_history` (with 90d retention), `calls` (metadata only — `transcript_uri` references Gong/Chorus). All carry the provenance quintet. No lifecycle tables yet.

**Skills (4):** `cold-start-interview` (inherits from `/csm:cold-start-interview`), `data-sync` (with `--dry-run` default on first run), `data-hygiene-check`, `portfolio-view` (the Monday-morning canned-query collection, not a full ad-hoc SQL surface).

**Pre-wired consumers:** `csm/managed-agents/health-watcher`, `csm/managed-agents/churn-signal-digest`, `csm/skills/account-research`, `csm/skills/renewal-readiness`, `csm/skills/call-prep`. Each sources `cache_helper.sh` and soft-falls to MCP on miss/timeout. Existing skill behavior unchanged when cache absent.

**Backend:** SQLite. Single file at `~/.local/share/cs-data-layer/cs.sqlite`, `chmod 600`. No pluggable backend ABC yet — the abstraction layer is deferred to Phase 2.

**Adapters:** YAML specs for Salesforce, HubSpot, the configured CS Platform, and Gong (calls metadata only). The `inspect → map → dry-run → commit` loop fires on cold-start and on every adapter YAML edit. **New mitigation against adapter rot (from operator review):** when `inspect` detects a high-confidence field rename (Levenshtein > 0.85 on the field name, same type), it offers to auto-update the YAML with a single confirmation prompt rather than requiring manual edit.

**Security baseline:** provenance quintet, `chmod 600`, secret-scrub on ingest, sanitize freetext, audit-log dry-runs. GDPR delete flow ships behind a `--enable-gdpr` config flag — schema columns exist (`is_deleted`, `deletion_requested_at`) but the delete skills are stubs until Phase 2.

**No write-back in Phase 1.** Write-back staging, AUQ-fallback hardening, allowlist management, and rate limits all defer to Phase 2. A solo CSM with the cache reads-only is materially helped; write-back is a separable, higher-risk surface.

**Storage budget Phase 1:** ~30 MB for a 200-account portfolio (accounts ~0.4 MB, signals snapshot ~4 MB, signals history 90d ~25 MB, calls metadata ~6 MB minus other-tables-absent reductions). Comfortable for SQLite without DuckDB substitution pressure.

**Phase 1 success criteria (gates Phase 2):**
- Adoption: at least 3 of the 5 pre-wired skills show >50% cache-hit rate within 30 days of install.
- Drift survival: at least one adapter YAML field rename auto-handled without manual edit.
- No data corruption: zero `data_quality_violations` of `severity: error` go unaddressed for >7 days.
- User-reported value: the CSM running it agrees, unprompted, that the cache is materially helping (qualitative gate — bypass anyway if quantitative gates clear).

### Phase 2 — Queryable warehouse + transforms + write-back (ship when Phase 1 demand is proven)

Adds the analytics surface: ad-hoc dashboard queries, declarative transforms + quality rules, write-back staging with full hardening, OCV Markdown↔table bridge, GDPR delete activation.

**Backend substitution (from buy-vs-build review):** SQLite → **DuckDB** as default for Phase 2. Both are embedded, free, MIT-licensed, zero-config; DuckDB is columnar and 10–50× faster on the portfolio-aggregation queries Phase 2 introduces (`GROUP BY health_band`, `JOIN signals × renewals`, retention prunes on `signals_history`). DuckDB's SQL dialect is Postgres-compatible, simplifying the v3 Postgres lift. The `DataLayerBackend` ABC is introduced in this phase (its abstraction earns its keep once two implementations exist and a third is in the planning window). Phase 1 SQLite users migrate via a single `python -m data_layer.lib.cli migrate-to-duckdb` command.

**Tables added in Phase 2:** `contacts`, `opportunities`, `renewals`, `outcomes`, `value_registry_entries`, `write_back_stage`, `gdpr_deletions`, `data_quality_violations`. View `value_map_summaries` materialized.

**Skills added in Phase 2:** `dashboard-query` (now justified by demonstrated demand for cross-account portfolio queries), `write-back-stage` (with the hardening described later in this plan), `outcome-registry-import`, `gdpr-delete`, `gdpr-export`, `run-migrations`, `customize`.

**Transform + quality rule engine** ships fully. The two-phase identity reconciliation, the `source_account_map` junction, the YAML adapter spec with `transforms:` and `quality:` blocks — all activate.

### Phase 3 — Revenue lifecycle traceability spine (ship when source artifacts demonstrably exist)

The eight lifecycle tables (`deals`, `discovery_findings`, `mutual_success_plans`, `revenue_attribution`, `msp_commitments`, `customer_signoffs`, `handoff_events`, `lifecycle_events`, `lineage_traces`) and the four lifecycle skills (`msp-deal-desk-review`, `lineage-trace`, `revenue-postmortem`, `msp-pregate`) ship **only after** the CSM/RevOps user confirms via cold-start interview that the upstream artifacts actually exist. **MSPs are auto-materialized from the closed-won opportunity** rather than requiring a pre-existing Drive/Notion MSP template, so the cold-start gate is simpler than before — it just asks whether the deal-record is sufficiently structured (line items, ICP-tier custom field, segment field, attribution custom fields) to seed an MSP automatically. The cold-start prerequisites are:

- Discovery findings: are they captured as structured data in a queryable form? (Gong call tags, Salesforce activity fields, a discovery template doc with parseable structure.)
- Deal record structure (for MSP auto-materialization): does the closed-won opportunity carry line items, an ICP-tier or segment custom field, and at least one outcome-related custom field (or a deterministic mapping from product SKU → typical outcomes) so the MSP can be seeded automatically? If yes, the optional Drive/Notion mirror is a bonus, not a requirement.
- Customer signoffs: are they captured beyond meeting transcripts (DocuSign envelopes, explicit email confirmations, click-acceptance records)?
- Revenue-attribution lookup feasibility (warn-only, not blocking): is at least one upstream marketing/attribution source queryable at close (HubSpot Marketing first-touch, Marketo lead source, Salesforce Campaign Influence, 6sense intent, Clearbit enrichment)? If none, Phase 3 still activates but `revenue_attribution` rows will be `lookup_completeness='unavailable'` until a source is connected, and the awareness→MQL segment of every lineage trace will be empty — gap-honest, by design.

If the answer is "no" to two or more of the **blocking** prerequisites (discovery, deal-record structure, signoffs), **Phase 3 does not activate**. The cold-start writes a deferral marker; rev-ops skills continue to operate on their existing stateless model. The plan is preserved in repository for the day the user's data maturity catches up — building tables for data that doesn't exist is the worst kind of rot. The revenue-attribution question is warn-only — a "no" there triggers a deferred-feature notice but does not block Phase 3.

If the blocking prerequisites are met, Phase 3 activates with the design described in detail later in this plan. The `lineage-trace` and `msp-deal-desk-review` skills remain gap-honest: missing artifacts (and partial attribution lookups) surface explicitly rather than producing silent partial chains.

### What was considered and rejected

The panel proposed several pivots the plan deliberately does not adopt, with reasoning:

- **"Use JSON files + bash + jq, no DB"** (minimalist). Rejected because cross-account portfolio queries (Monday-morning triage, expansion candidates, GRR-at-risk views) are independently validated by the operator review as real solo-CSM needs, and joining 4 normalized tables across 200 accounts in jq is materially worse than SQL. Phase 1's 4-table SQLite footprint is itself near-minimal; further reduction loses real value.
- **"Use Airbyte + DuckDB + Metabase"** (buy-vs-build). Rejected for the ETL/BI layers: requires Docker, conflicts with the "plugin set, not deployment platform" model of this repo, and adds vendor surface area incompatible with the solo-CSM-laptop install pattern. **Partially accepted**: the DuckDB backend recommendation lands in Phase 2 (the rationale for columnar OLAP is sound and the install footprint is zero-config).
- **"Three-plugin split: data-cache, data-warehouse, revenue-lifecycle"** (boundaries). Rejected as a v1 step because three plugins to install vs. one with progressive disclosure is worse adoption ergonomics for the solo CSM, and the shared provenance/adapter/identity code would either duplicate or require a library plugin (which CLAUDE.md does not yet support cleanly). **Re-evaluated at Phase 3**: if the lifecycle spine activates, it ships as a sibling plugin `revenue-lifecycle/`, not as more tables in `data-layer/`. The architectural split becomes natural at that point rather than imposed up front.
- **"Defer the entire data layer; teach existing skills to cache MCP responses to disk"** (minimalist alternative). Rejected because that approach has no place to put portfolio queries, no place to hold normalized identity reconciliation across sources, and recreates the hygiene problem the user originally raised (skills re-applying normalization rules independently).

The phased rollout above is the synthesized response: ship the smallest thing the panel agrees survives, design the rest fully but gate its activation on evidence.

---

## Architecture

*Note on phasing: the sections below describe the **full design across Phases 1–3**. Phase 1 ships the cache + 4 tables + 4 skills + SQLite as listed above. Phases 2 and 3 are designed here so the v1 implementation does not paint itself into a corner — they are gated, not promised. Where a feature applies only to Phase 2 or 3, it is marked.*

### Plugin directory layout

Matches peer-plugin conventions (`csm/`, `rev-ops/`). Adds a `lib/` Python subtree — new but small. Skills do not import Python; they shell out via a Bash helper that wraps the CLI.

```
data-layer/
├── .mcp.json                       # Inherits the same MCP server set as csm/cs-ops
├── .gitignore                      # Excludes *.sqlite, *.sqlite-journal, /local/, /audit/
├── CLAUDE.md                       # Plugin profile (cold-start writes user copy)
├── README.md                       # Overview + "how skills opt in" snippet
├── ARCHITECTURE.md                 # Design rationale + "v1 is SQLite-only" caveat
├── agents/
│   └── data-sync.md                # Scheduled managed-agent registration
├── hooks/
│   └── hooks.json                  # Re-uses ../auq-resilience/hooks/auq_hook.py
├── skills/
│   ├── cold-start-interview/       # Configures backend, db path, schedule, allowlist
│   ├── customize/
│   ├── data-sync/                  # On-demand ingestion
│   ├── data-hygiene-check/         # Drift + staleness + adapter-spec validation
│   ├── dashboard-query/            # Canned queries + ad-hoc SQL with safety rails
│   ├── write-back-stage/           # Stage, approve, push CRM write-back
│   ├── outcome-registry-import/    # Markdown OCV ↔ value_registry_entries
│   ├── gdpr-delete/                # Right-to-be-forgotten flow
│   ├── gdpr-export/                # Data portability dump
│   ├── msp-deal-desk-review/       # CS deal-desk in-workflow review of materialized MSP
│   ├── lineage-trace/              # Walk a churn/downgrade back through MSP → discovery → attribution
│   ├── revenue-postmortem/         # Cohort-level root-cause + awareness-segment patterns
│   ├── msp-pregate/                # Pre-handoff blocker for rev-ops deal desk
│   └── run-migrations/             # Manual migration runner (--dry-run supported)
├── adapters/                       # Declarative YAML specs, not Python files
│   ├── adapter.schema.yaml         # Meta-schema for adapter YAMLs
│   ├── hubspot.adapter.yaml        # Reference: HubSpot ↔ normalized tables
│   ├── gainsight.adapter.yaml      # Reference: Gainsight ↔ normalized tables
│   ├── gong.adapter.yaml           # Reference
│   └── README.md                   # How to author a new adapter (community PRs are YAML diffs)
├── reference/
│   ├── schema.sql                  # Canonical DDL — single source of truth
│   ├── source-field-map.md         # Generated from adapters/*.yaml
│   └── confidence-band-rules.md    # When fields are stamped High / Moderate / Low
├── lib/
│   ├── __init__.py
│   ├── backend.py                  # DataLayerBackend ABC
│   ├── sqlite_backend.py           # Default implementation (~200 LOC)
│   ├── adapter_executor.py         # Hydrates adapters/*.yaml → upsert calls
│   ├── ingestion.py                # Orchestrates adapters, secret scrub, reconciliation
│   ├── identity.py                 # Two-phase upsert + canonical ID reconciliation
│   ├── normalize.py                # Health bands / confidence at write
│   ├── transforms.py               # Built-in transform op registry + custom op loader
│   ├── quality.py                  # Built-in quality rule registry + violation reporter
│   ├── retention.py                # Nightly prune sweep per CLAUDE.md retention block
│   ├── secret_scrub.py             # Redacts tokens, OAuth refresh, JWTs from raw_blob
│   ├── sanitize.py                 # HTML/SQL-injection escape pass on freetext columns
│   ├── value_map_bridge.py         # Mirror summary fields from YAML Value Maps
│   ├── ocv_parser.py               # Markdown OCV ↔ value_registry_entries
│   ├── cache_helper.sh             # Bash helper skills source — single function, low friction
│   └── cli.py                      # `python -m data_layer.lib.cli sync|query|stats|get`
├── migrations/
│   └── 001_init.sql                # Versioned schema migrations with backfill checkers
├── samples/
│   ├── seed-crm.json
│   ├── seed-csp.json
│   └── seed-calls.json
└── evals/
    ├── test_schema_contract.py     # PRAGMA-based column-presence assertions
    ├── test_sync_golden_path.py    # End-to-end seed → sync → query
    ├── test_adapter_conformance.py # Each adapter YAML produces valid rows
    └── test_injection_safe.py      # Poison fixtures verify sanitize + trust boundary
```

A matching cookbook at `managed-agent-cookbooks/data-sync/` follows the `value-map-system/` layout.

### Backend interface (`lib/backend.py`)

Single ABC, introduced in Phase 2 when a second implementation (DuckDB) earns the abstraction. **Phase 1 ships SQLite-only with no ABC** — direct SQLite calls keep the v1 LOC count down. **Phase 2 introduces the ABC and DuckDB as the default backend** (10–50× faster on portfolio aggregations and retention prunes; same install footprint as SQLite; Postgres-compatible dialect simplifies the eventual Postgres lift). **A generic Postgres backend** (self-hosted or hosted) lands when the user outgrows local; **Supabase is the recommended hosted-Postgres deployment target**, with a separate `lib/supabase_backend.py` that extends the generic Postgres backend with Auth, RLS policy generation, Realtime subscriptions, and Storage for `raw_blob`. The ABC stays Postgres-generic so self-hosted Postgres remains a supported path. Documented honestly in `ARCHITECTURE.md`. No ORM.

```python
class DataLayerBackend(ABC):
    def connect(self) -> None: ...
    def close(self) -> None: ...
    def migrate(self, migrations_dir) -> None: ...

    def upsert(self, table, rows, *, source, fetched_at) -> int: ...
    def query(self, sql, params=None) -> list[dict]: ...
    def get(self, table, *, account_id) -> dict | None: ...
    def since(self, table, *, account_id, since) -> list[dict]: ...

    def freshness(self, table, account_id) -> dict: ...   # {fetched_at, age_days, is_stale, confidence_band}

    def stage_write_back(self, target_source, target_record_id, proposed_change,
                         *, proposed_by, authority) -> str: ...
    def list_pending_write_backs(self, *, source=None) -> list[dict]: ...
    def mark_write_back(self, stage_id, status, note="") -> None: ...

    # GDPR
    def stage_deletion(self, *, account_id=None, contact_id=None, reason) -> str: ...
    def apply_deletion(self, deletion_id, *, approver) -> None: ...
```

### Normalized schema sketch (`reference/schema.sql`)

Every fact table carries the **provenance quintet** (`source`, `source_record_id`, `fetched_at`, `confidence`, `raw_blob`) plus the **audit-and-deletion trio** (`is_deleted`, `deletion_requested_at`, `created_by`) so GDPR + multi-tenant retrofits don't require a breaking migration later.

Tables annotated **[P1]**, **[P2]**, or **[P3]** ship in that phase.

Core tables:

| Table | Phase | Purpose | Key columns |
|---|---|---|---|
| `accounts` | **P1** | Canonical account record (one per reconciled account) | `account_id` PK, `account_name`, `segment`, `arr_usd`, `renewal_date`, `csm_owner`, `previous_canonical_id`, + provenance, + audit/deletion |
| `source_account_map` | **P1** | Junction: which source records resolve to which canonical account | `source`, `source_account_id`, `canonical_account_id` FK, `match_method`, `match_confidence`, `reconciled_at` |
| `signals_snapshot` | **P1** | One row per (account, signal_type) — latest only, upsert on conflict | `account_id`, `signal_type`, `signal_value`, `health_band`, `observed_at`, + provenance, PK `(account_id, signal_type)` |
| `signals_history` | **P1** | Append-only audit trail; no UNIQUE constraint; 90d retention | `signal_id` PK, `account_id`, `signal_type`, `signal_value`, `health_band`, `observed_at`, `archived_at`, + provenance |
| `calls` | **P1** | Gong/Chorus metadata + URI (never transcript body) | `call_id`, `account_id` FK, `occurred_at`, `duration_sec`, `participants`, `transcript_uri`, `highlights`, + provenance |
| `contacts` | **P2** | Stakeholders | `contact_id` PK, `account_id` FK, `full_name`, `title`, `email`, `influence_tier`, `last_engaged_at`, + provenance, + audit/deletion |
| `opportunities` | **P2** | Open and historical opps (live state; not the close snapshot) | `opportunity_id`, `account_id` FK, `type`, `stage`, `amount_usd`, `close_date`, `is_won`, + provenance |
| `renewals` | **P2** | Renewal-specific projection | `renewal_id`, `account_id` FK, `renewal_date`, `arr_at_risk_usd`, `status`, `forecast_weight`, + provenance |
| `outcomes` | **P2** | Per-account outcome observations | `outcome_id`, `account_id` FK, `ocv_entry_id` FK, `stage`, `status` (L0–L3), `evidence`, `observed_at`, + provenance |
| `value_registry_entries` | **P2** | OCV mirror (canonical Markdown remains source-of-truth) | `entry_id` PK, `name`, `customer_segment`, `primary_metric`, `value_chain_stage`, `status`, `rubric`, `source_path`, `parsed_at`, `markdown_checksum`, + provenance |
| `write_back_stage` | **P2** | Staged CRM writes awaiting approval | `stage_id` PK, `target_source`, `target_record_id`, `proposed_change` (JSON), `proposed_by`, `authority` (blue/green), `status`, `note`, `created_at`, `resolved_at`, `approver`, `diff_checksum` |
| `gdpr_deletions` | **P2** | Immutable audit of deletion events (schema columns ship in P1, skills in P2) | `deletion_id` PK, `subject_type`, `subject_id`, `requested_at`, `approved_by`, `applied_at`, `wiped_columns` (JSON) |
| `data_quality_violations` | **P2** | Outputs from adapter `quality:` rules + per-run summary | `violation_id` PK, `run_id`, `source`, `table`, `column`, `row_pk`, `rule`, `severity`, `message`, `observed_at` |

`value_map_summaries` is a **VIEW**, not a manually-mirrored table — it always reads fresh from YAML via `value_map_bridge.py`. Materialization is opt-in for Postgres only (SQLite doesn't index views).

Indexes: `(account_id, observed_at DESC)` on `signals_history`; `(account_id, occurred_at DESC)` on `calls`; `(account_id, stage)` on `opportunities`; `(signal_type, health_band, observed_at)` for aggregates.

### Identity reconciliation (the missing piece)

**Two-phase upsert** in `lib/identity.py`:

1. **Reconcile phase** — on every incoming row, compute deterministic match key `LOWER(TRIM(legal_company_name)) || domain_hash(email_domain)`. Look up `source_account_map`. If a canonical account exists for that key, reuse its `account_id`. Else assign a new one. Optional fuzzy fallback (Levenshtein > 0.85) is configurable; default off.
2. **Upsert phase** — `INSERT ... ON CONFLICT (source, source_record_id) DO UPDATE SET fetched_at=now(), confidence=?, raw_blob=?, ...`.

Reconciliation events log to `accounts.previous_canonical_id` + an entry in `write_back_stage` as an informational "merged accounts" note. **No human-in-the-loop for default deterministic matches.** Fuzzy matches above the threshold but below 1.0 go to a manual review queue surfaced by `data-hygiene-check`.

Walk-through: when HubSpot's `hubspot-12345` arrives and Salesforce already seeded canonical `ACME-001` from `sf-9991`:
- Reconcile phase finds match-key collision → reuse `account_id = 'ACME-001'`.
- Upsert phase writes `accounts(source='hubspot', source_record_id='hubspot-12345', account_id='ACME-001', …)` and adds a row to `source_account_map`.

### Ingestion + normalization flow

```
MCP tool call
    ↓
adapter_executor.py loads adapters/<source>.adapter.yaml
    ↓
secret_scrub.redact(raw_blob)        # Strip tokens, OAuth refresh, JWTs
    ↓
sanitize.clean(freetext_fields)      # HTML escape, quote escape — preserve raw_blob untouched
    ↓
identity.reconcile(rows)              # Two-phase upsert resolution
    ↓
normalize.apply_rules(rows)           # Health bands, confidence assignment
    ↓
backend.upsert(table, rows, source=..., fetched_at=now())
```

**Adapter YAML spec** — adapters are declarative, not Python:

```yaml
# adapters/hubspot.adapter.yaml
source_name: hubspot
tables:
  accounts:
    source_object: company
    fields:
      account_name:    { source_field: name, type: string, required: true }
      arr_usd:         { source_field: lifecyclestage_revenue, type: float, nullable: true }
      renewal_date:    { source_field: renewal_date__c, type: date }
    match_key_fields: [name, primary_email_domain]
validation:
  integrity_checks:
    - { name: account_name_present, query: "SELECT count(*) FROM accounts WHERE account_name IS NULL" }
```

Community PRs to add Totango/ChurnZero/Vitally support are YAML diffs, not Python. `data-hygiene-check` validates adapter specs against live MCP responses and flags drift (e.g., HubSpot renames `name` → `company_name`) at ingest time, not at query time.

**Rules at write time** (deterministic): `health_score → health_band` per cs-domain-model.md §1; confidence (High = live MCP this run, Moderate = fixture, Low = user-provided); source-attribution stamp.

**Rules at read time** (clock- or period-dependent): `[stale — N days]` tag (re-evaluated every read); GRR/NRR formulas (period-dependent, never stored); forecast scenario weighting.

### Read-through cache integration — the seam

**Opt-in via a Bash helper, not an MCP shim and not a Python import.** Skills source `lib/cache_helper.sh` and call a single function:

```bash
source data-layer/lib/cache_helper.sh
ROW=$(cache_get "ACME-001" "accounts" max_age_days=7)
if cache_miss "$ROW"; then
    # Skill calls MCP directly — existing path unchanged
else
    # Use cached $ROW
fi
```

The CLI wraps a 2-second timeout and returns `{"hit": false, "fallback": "mcp"}` on timeout — skills never block on a misbehaving cache. Friction for a skill author drops from "implement CLI integration" to "source one helper, call one function."

**Pre-wired skills shipped with the cache as default:**
- `csm/managed-agents/health-watcher` — must use cache (portfolio-wide every cycle; the most expensive offender against MCP rate limits)
- `csm/managed-agents/churn-signal-digest` — same
- `csm/skills/account-research`, `call-prep`, `risk-flag`, `renewal-readiness`, `qbr-builder` — pre-wired with explicit MCP escape hatch

The remaining 70+ skills carry on calling MCPs directly. Zero forced changes.

### Write-back semantics — hardened

CRM write-back is the highest-risk surface. Hardening from the security review:

1. **Stage** — skill calls `backend.stage_write_back(...)`. Row written with `status='pending'`, `proposed_by`, `authority` (Blue/Green per Value Map model).
2. **Rate limit** — backend rejects `stage_write_back` if the same `proposed_by` has >10 pending writes in the last hour. Returns `'too_many_pending'`.
3. **Approve + push** — `/data-layer:write-back-stage --approve <id>` shows the **full diff** (current CRM value → proposed). **AUQ prose-fallback `proceeding-with-this-if-no-response` is DISABLED for write-back approval** — no response within 60s marks the write `awaiting_manual_review` and posts a Slack notification; nothing pushes.
4. **Bulk approval** — `>5` pending writes require explicit `--bulk-approve` and a second confirmation.
5. **Audit** — every approval writes approver identity (from git config / env), timestamp, and SHA256 `diff_checksum` to `write_back_stage`.

**Allowlist:** `data-layer/CLAUDE.md` carries a `write_back_allowlist:` block. Default is **not empty** — `cold-start-interview` proposes a safe default set during setup:
- `salesforce.account.csm_owner`
- `salesforce.account.next_qbr_date`
- `gainsight.account.health_score_override`
- `gainsight.account.success_plan_status`

The user accepts, edits, or empties the list. Off-by-default-but-pre-populated keeps write-back useful without making the CSM edit JSON to enable it.

### Security & privacy (folded from the security review)

- **File ACL** — `cold-start-interview` sets `chmod 600` on the SQLite file and verifies it on every `connect()`. Refuses to open the DB if permissions are looser.
- **Encrypted backups** — config option `encrypt_backups` (default true); cookbook documents exclusion from Time Machine / Backblaze unless encrypted.
- **Secure deletion on uninstall** — `/data-layer:gdpr-delete --uninstall` shreds the SQLite file (overwrite then unlink), not just `rm`.
- **Secret scrubbing** — `lib/secret_scrub.py` runs before every upsert; redacts AWS keys, GitHub PATs, Slack tokens, JWTs, OAuth refresh tokens from `raw_blob`. Redacted paths logged to `audit/scrub.log`.
- **Custom-field allowlist** — `data-layer/CLAUDE.md` carries `custom_field_allowlist:`; default empty so custom CRM fields are scrubbed unless explicitly opted-in.
- **Retrieved-content trust boundary** — `raw_blob` and any column derived from it are treated as untrusted data per the existing `csm/CLAUDE.md` "Retrieved-content trust" guardrail. `sanitize.py` escapes freetext before storing in normalized columns; tests in `evals/test_injection_safe.py` poison fixtures and assert no skill renders unescaped content.
- **GDPR deletion flow** — `/data-layer:gdpr-delete --account <id>` sets `is_deleted=true`, wipes PII columns (`full_name`, `email`, `internal_notes`, `raw_blob`) to `[redacted]`, stages a CRM write-back deletion request, and logs to immutable `gdpr_deletions` table + `audit/gdpr-deletes.log`.

### OCV sync — explicit reimport with checksum guard

The Markdown OCV stays source-of-truth, but the sync direction is now explicit:
- `outcome-registry-import` computes a SHA256 `markdown_checksum` on the file and stores it in `value_registry_entries`. On reimport, if checksum unchanged, no-op. If changed, surface a **diff confirmation** to the CSM before overwriting the table.
- `dashboard-query` and any skill that updates an entry via SQL writes `status='dirty'` on that row and surfaces a "Reverse-sync needed" warning in `data-hygiene-check`. Reverse sync (table → Markdown) is a manual `/data-layer:outcome-registry-import --reverse` flow with a diff prompt.
- Scheduled reimport runs daily and only re-parses files whose `mtime > parsed_at`.

This makes the sync rhythm visible instead of a workflow gotcha.

### Coexistence with `value-map-system/`

Authority unchanged. Blue agents (HIE, VMB) own `value-maps/{account_id}/value-map.yaml`. The data layer is a Green-equivalent reader; NEVER-rule #4 (no green writes to quadrants) preserved.

`value_map_summaries` is a VIEW that calls `value_map_bridge.py` on read, parsing the YAML fresh each time. No mirror-table staleness. Index it only under Postgres in v2.

### Cold-start inheritance — reduces friction

`/csm:cold-start-interview` is enhanced with one question: *"Activate the data layer for cross-account querying and health caching?"* If yes, it auto-generates `data-layer/CLAUDE.md` inheriting CSM thresholds, segment definitions, escalation matrix, and the safe-default write-back allowlist. The CSM sees one cold-start, not two.

`/data-layer:cold-start-interview` remains as the standalone option for users who skipped the CSM plugin or want to reconfigure later.

### Source-mapping inspection + dry-run pull (cold-start safety)

The cold-start sequence — and any later adapter or source change — runs through a four-step **inspect → map → dry-run → commit** loop before any data is persisted. This addresses three risks at once: adapter YAML drift, identity-reconciliation surprises, and CSM trust ("what's actually about to land in my cache?").

**Step 1 — Inspect.** For each MCP source configured in `.mcp.json`, the system calls a lightweight introspection query (e.g., Salesforce `describeSObject('Account')`, HubSpot `/crm/v3/properties/companies`, Gainsight schema endpoint). The live field list is captured.

**Step 2 — Map.** The live field list is diffed against the adapter YAML's declared `source_field` references:
- Fields present in YAML but missing live → **drift** (adapter needs update, e.g., HubSpot renamed `name`).
- Fields present live but not in YAML → **candidates** (offered to the CSM to add to the mapping; not added silently).
- Fields present in both → green.
- The CSM is shown a side-by-side diff. They accept, edit, or skip per-field.

**Step 3 — Dry-run pull.** With the agreed mapping, the system pulls a configurable sample (default: 5 records per table per source) and runs the **full ingestion pipeline against an in-memory SQLite** — secret scrub, sanitization, identity reconciliation, normalization. Nothing touches the real cache. The CSM sees a structured preview:

```
DRY RUN — salesforce → accounts (5 records sampled)
  ✓ Mapped fields:  account_name, segment, arr_usd, renewal_date, csm_owner
  ⚠ Reconciliation: 2 records collide with existing canonical IDs (will merge)
                    - sf-9991 (ACME Corp) → existing canonical ACME-001 (from hubspot)
                    - sf-7714 (Globex)    → existing canonical GLOBEX-002 (from hubspot)
  ⚠ Confidence:     1 record stamped Low (renewal_date missing)
  ✓ Secret scrub:   0 secrets redacted
  ⚠ Sanitization:   1 record had HTML in internal_notes (escaped in normalized column,
                    raw preserved in raw_blob)
  ✓ Health bands:   2 Red, 1 Yellow, 2 Green (per cs-domain-model.md §1)
```

**Step 4 — Commit.** CSM responds `approve`, `skip-source`, or `abort`. Only on `approve` does the full sync run against the real backend. The dry-run report is written to `audit/dry-runs/<timestamp>.json` for later review.

**Where this loop fires:**

1. **Initial cold-start** — once per configured source. The CSM never lands data they haven't previewed.
2. **Adapter YAML edits** — `/data-layer:data-sync --dry-run --source <name>` is the recommended next step after editing `adapters/<source>.adapter.yaml`. The `data-sync` skill defaults to `--dry-run=auto`, which forces a dry-run if the adapter file's checksum has changed since last sync.
3. **New custom-field allowlist entries** — any addition to `custom_field_allowlist:` triggers a dry-run on next sync so the CSM sees what's now flowing through that they weren't seeing before.
4. **Schema migrations** — `/data-layer:run-migrations --dry-run` already planned; this extends it to also dry-run a representative ingestion against the post-migration schema.
5. **Scheduled sync** — never auto-dry-runs (it would be noise); but if the scheduled sync detects adapter drift (field disappeared, type changed), it soft-fails that source and surfaces a "dry-run recommended" notice in the next `data-hygiene-check`.

**Config surface** added to `data-layer/CLAUDE.md`:
```yaml
dry_run:
  sample_size: 5                   # records per table per source
  auto_on_adapter_change: true     # checksum-gated
  retain_reports_days: 30
  block_commit_on_unresolved_drift: true   # safety default — turn off only knowingly
```

**New CLI verbs** in `lib/cli.py`:
- `python -m data_layer.lib.cli inspect --source <name>` — Step 1 + 2 only, returns mapping diff JSON.
- `python -m data_layer.lib.cli dry-run --source <name> [--sample N]` — Steps 1–3, returns dry-run report JSON, never commits.
- `python -m data_layer.lib.cli sync --source <name> [--commit]` — full pipeline; `--commit` required for real writes, otherwise behaves as dry-run.

**Why this earns its complexity:** the cost is one new CLI verb, one extension to `cold-start-interview`, and ~150 LOC in `lib/ingestion.py` to make the pipeline runnable against an in-memory backend. The benefit is that **the most expensive failure modes (silent adapter drift, bad reconciliation merges, unexpected secret exposure, schema mismatches with the source CRM) all surface before any data hits the persistent cache** — a much cheaper place to catch them than `data-hygiene-check` running a day later against a polluted store.

### Transforms and data-quality rules (declarative, in the adapter YAML)

The adapter YAML extends with two new optional blocks per table: `transforms:` (rule-based field reshaping) and `quality:` (assertions that surface during dry-run and abort persistent sync if `block_commit_on_quality_fail: true`). Both run inside the ingestion pipeline, so they are exercised by `dry-run` before any persistent write.

```yaml
# adapters/hubspot.adapter.yaml (extended)
source_name: hubspot
tables:
  accounts:
    source_object: company
    fields:
      account_name: { source_field: name, type: string, required: true }
      arr_usd:      { source_field: annual_revenue_cents, type: float }
      segment:      { source_field: lifecycle_segment, type: string }
      renewal_date: { source_field: renewal_date__c, type: date }
    transforms:
      - field: arr_usd
        op: divide_by
        arg: 100
        because: "HubSpot stores revenue in cents"
      - field: segment
        op: fallback_chain
        args: [source_value, "infer_from_arr"]   # custom infer fn registered in lib/transforms.py
      - field: renewal_date
        op: parse_date
        formats: ["%Y-%m-%d", "%m/%d/%Y"]
    quality:
      - { rule: not_null, field: account_name, severity: error }
      - { rule: range,    field: arr_usd, min: 0, severity: error }
      - { rule: regex,    field: primary_email, pattern: "^[^@]+@[^@]+\\.[^@]+$", severity: warn }
      - { rule: referential, field: csm_owner, references: "users.email", severity: warn }
      - { rule: freshness, max_age_days: 30, source_field: hs_lastmodifieddate, severity: warn }
```

**Built-in transform ops** (extensible via `lib/transforms.py` registry):
- Math: `divide_by`, `multiply_by`, `clamp`
- String: `lowercase`, `trim`, `regex_replace`, `split_first`, `concat`
- Date: `parse_date`, `relative_days`, `truncate_to_month`
- Logic: `fallback_chain`, `coalesce`, `case_when`, `enum_map`
- Domain-aware: `to_health_band` (cs-domain-model.md §1), `to_confidence_band`, `match_segment_rule`

**Built-in quality rules**:
- Structural: `not_null`, `unique`, `length_between`, `enum_in`
- Numeric: `range`, `nonzero`, `percent_change_max` (anti-spike: alert if ARR jumps >300% sync-over-sync)
- String: `regex`, `length`, `forbidden_substrings` (e.g., redacted markers in normalized columns)
- Referential: cross-table FK assertions
- Temporal: `freshness`, `monotonic` (e.g., `created_at <= updated_at`)
- Severity: `error` aborts commit (when `block_commit_on_quality_fail: true`); `warn` shows in the dry-run report and `data-hygiene-check` digest but lets sync proceed.

**Insulation effect.** When a source changes:
- A field renamed → YAML one-line edit, dry-run re-validates.
- A field type changes (string → number, cents → dollars) → add a `transform:` entry, no downstream consumer changes.
- A field becomes optional or starts returning nulls → `quality: not_null` flags it on next sync; the CSM decides whether to add a fallback or surface the regression.
- A source adds a new field → `inspect` lists it as a candidate; CSM adds it to the YAML when needed.

The Python code path stays generic (`adapter_executor.py` reads YAML, applies transforms, runs quality). Skills consuming the cache see a **stable normalized schema** even when upstream sources churn. This is the second-level insulation beyond the field-mapping layer.

**Quality-check outputs** land in `data_quality_violations` (new table): `violation_id PK, run_id, source, table, column, row_pk, rule, severity, message, observed_at`. `data-hygiene-check` summarizes the last N runs; the daily managed agent posts a digest of new errors.

### Storage model — hybrid mirror, not a full data warehouse

This is the most important scope decision. The data layer is **neither a thin summary index nor a complete data replica.** It's a hybrid:

**Fully mirrored** (structured rows, queryable, the cache is the answer for most reads):
- `accounts`, `contacts`, `opportunities`, `renewals`, `outcomes`, `value_registry_entries`
- `signals_snapshot` (one row per account/signal_type — latest)
- `signals_history` (time-series, but bounded by retention policy)
- `calls` *metadata* (call_id, account_id, occurred_at, duration, participants, summary, highlights URI)

**Reference-only** (metadata + pointer; bodies fetched on demand via MCP):
- Call **transcripts** — `calls.transcript_uri` points to Gong/Chorus; never stored.
- Document **bodies** — Drive/Notion documents: store title, URL, owner, last-modified; fetch body when a skill needs it.
- Email **threads** — store thread-level metadata; never message bodies.
- **Attachments** of any kind — URI only.

**Why hybrid:**
- Cross-account portfolio queries ("all Red accounts with renewal in 60d") need structured rows in the cache to be useful at all.
- Transcript bodies and document bodies are large, change rarely-but-substantially, are already well-served by their source's own search, and concentrating them locally multiplies PII exposure for little query value.
- Skills that need a transcript body still have to call Gong's MCP — that's fine, it's a per-call cost, not a per-query-of-the-portfolio cost.

**Retention policy** (configurable in `data-layer/CLAUDE.md`):
```yaml
retention:
  signals_history_days: 90       # time-series pruned after 90d (default; tunable)
  calls_metadata_days: 365       # call metadata kept 1 year
  outcomes_days: unlimited       # outcome observations never auto-purged
  raw_blob_days: 30              # raw_blob preserved 30d for audit, then nulled (normalized cols stay)
  data_quality_violations_days: 180
  dry_run_reports_days: 30
```

A nightly task (`data-sync-prune`) runs the retention sweep. Pruning never deletes structured columns — it only nulls `raw_blob` after 30d and trims `signals_history` past the retention horizon.

**Storage budget — what to actually expect** (200-account solo CSM portfolio):

| Table | Rows | Per-row | Subtotal |
|---|---|---|---|
| accounts | 200 | 2 KB | 0.4 MB |
| contacts | 10,000 (avg 50/account) | 1 KB | 10 MB |
| opportunities | 1,000 | 1 KB | 1 MB |
| renewals | 600 | 0.5 KB | 0.3 MB |
| calls (metadata) | 6,000 (30/account/yr) | 1 KB | 6 MB |
| signals_snapshot | 4,000 (20 types × 200) | 1 KB | 4 MB |
| signals_history (90d retention) | ~180,000 | 0.5 KB | 90 MB |
| outcomes | 2,000 | 1 KB | 2 MB |
| value_registry_entries | ~100 | 5 KB | 0.5 MB |
| raw_blob bloat (30d window) | — | — | ~25 MB |
| indexes + overhead | — | — | ~20 MB |
| **Total at steady state** | | | **~160 MB** |

That's comfortable for SQLite. Without `signals_history` retention, the table grows ~1 MB/day and would push past SQLite's comfort zone within a year. The retention policy makes the difference.

**Hard ceilings the plan accepts:**
- Cap on `raw_blob` per-row size: 256 KB (truncate with audit note above that — unusual records like 5,000-line internal_notes are flagged).
- Total file size warning at 500 MB; recommend Postgres backend (v2) above 1 GB.
- A `data-hygiene-check --storage` subcommand shows current breakdown so the CSM can see where bytes are going.

**What this is NOT.** Not a data warehouse. Not a system of record. Not a place call transcripts or document bodies live. Not a substitute for the YAML Value Maps (those remain canonical). It is a **structured, normalized, query-friendly mirror of the columns that benefit from being queryable together**, with bulky payloads referenced rather than duplicated.

### Revenue lifecycle traceability (deal → MSP → outcomes → renewal)

The most important hygiene problem the data layer solves is **making the revenue lifecycle chain queryable end-to-end**, so any non-renewal, churn, or seat-reduction event can be traced back through the MSP, the deal, and the original discovery — with evidence preserved, not lost.

**Why this is hard today.** The chain exists as scattered artifacts: discovery notes in Gong + Salesforce activity history; a deal in Salesforce; a Mutual Success Plan in a Google Doc or Markdown; customer signoff captured as a DocuSign envelope, a meeting recording timestamp, or an email thread; the renewal event as a subscription record. Each artifact lives in a different system with a different ID space. Nothing joins them, so when a CSM is asked "why did Acme churn?", the answer is reconstructed from scratch every time. The rev-ops plugin already has skills that do this reconstruction (`deal-to-outcome-tracing`, `sales-cs-handoff-quality-scoring`, `outcome-to-value-tracking`, `early-churn-downgrade-signal-detection`), but they're stateless — every invocation re-reads sources and produces a snapshot that isn't queryable as a cohort.

**Design principle.** The data layer adds the **spine** (tables); existing rev-ops and CSM skills become **consumers of the spine** that write their verdicts back. **The MSP is the normalization path**: it is auto-materialized from the Salesforce/HubSpot opportunity at close-won, becoming the canonical post-sale structured record that the CS portion of the deal desk reviews in-workflow (no CRM login required). The MSP row is the first checkpoint for **outcome and value alignment** (against the OCV outcome catalog and value register) and the **initial write-point for revenue observability** — everything upstream of the MSP (campaign, lead source, ICP tier, awareness path) is captured by *lookup queries* against marketing/CRM systems at materialization time and snapshotted into `revenue_attribution`, completing the awareness→current-state picture without requiring upstream tools to be first-class synced sources. Source-of-truth ownership for non-materialized artifacts is unchanged: signoffs live in DocuSign/email; deals live in Salesforce; the live opportunity record lives in the CRM. A Drive/Notion MSP document, when present, acts as a **human-readable mirror** that may extend the materialized row's narrative fields — the structured row remains authoritative for queries and lineage.

**New tables (folded into the schema sketch):**

| Table | Purpose | Key columns |
|---|---|---|
| `deals` | **Immutable snapshot** of a closed/won opportunity at the moment of close (separate from `opportunities`, which mirrors live state) | `deal_id` PK, `account_id` FK, `opportunity_id` (link to live record), `closed_won_at`, `acv_usd`, `tcv_usd`, `term_months`, `seat_count_committed`, `product_skus` (JSON), `ae_owner`, `closing_segment`, `pricing_terms` (JSON), `closed_won_snapshot` (full JSON of opp record at close), + provenance |
| `discovery_findings` | Pre-sales discovery distilled into structured fields | `discovery_id` PK, `account_id` FK, `opportunity_id` FK, `pain_points` (JSON), `success_criteria` (JSON), `stakeholder_ladder` (JSON), `value_hypothesis_text`, `meddic` (JSON: metrics/economic-buyer/decision-criteria/decision-process/identify-pain/champion), `evidence_uris` (JSON: Gong call IDs, doc URIs), `captured_at`, `captured_by`, + provenance |
| `mutual_success_plans` | **Normalized post-sale record** auto-materialized from the closed-won opportunity. Authoritative for queries and lineage; an optional Drive/Notion document acts as a human-readable mirror for narrative fields. | `msp_id` PK, `account_id` FK, `deal_id` FK, `discovery_id` FK, `version`, `materialization_method` (auto/manual/hybrid), `materialized_from_deal_at` (timestamp of close-won trigger), `acv_at_close_usd`, `committed_segment`, `committed_icp_tier`, `committed_product_skus` (JSON, snapshot), `revenue_attribution_id` FK → `revenue_attribution`, `deal_desk_review_status` (pending/approved/changes_requested), `deal_desk_reviewed_by`, `deal_desk_reviewed_at`, `deal_desk_notes`, `mirror_source_uri` (nullable; Drive/Notion mirror if present), `mirror_markdown_checksum`, `signed_at`, `signed_by_customer`, `signed_by_internal`, `status` (draft/dd_review_pending/dd_approved/signed/superseded/voided), `parsed_at`, + provenance |
| `revenue_attribution` | Awareness→close attribution snapshot captured at MSP materialization via lookup queries against marketing/CRM systems. One row per deal. Snapshot — not continuously synced. | `attribution_id` PK, `deal_id` FK (1:1), `account_id` FK, `awareness_source` (inbound_organic/inbound_paid/outbound_sdr/outbound_ae/referral/partner/event), `originating_campaign_id`, `campaign_name`, `lead_source`, `first_touch_at`, `last_touch_at`, `mql_at`, `sql_at`, `mql_to_sql_lag_days`, `sql_to_close_lag_days`, `icp_tier`, `icp_type`, `intent_signals` (JSON: 6sense/Bombora/etc.), `lookup_sources` (JSON: marketo/hubspot-marketing/6sense/clearbit), `lookup_completeness` (full/partial/unavailable), `looked_up_at`, + provenance |
| `msp_commitments` | Line items linking an MSP to OCV outcomes + committed value | `commitment_id` PK, `msp_id` FK, `ocv_entry_id` FK → `value_registry_entries`, `committed_outcome_text`, `expected_value_usd`, `expected_value_unit` (e.g., "hours saved/year"), `target_realization_date`, `measurement_method`, `current_status` (committed/in_progress/realized/missed/voided), + provenance |
| `customer_signoffs` | Atomic record of a customer commitment moment | `signoff_id` PK, `account_id` FK, `msp_id` FK (nullable — signoffs can predate the MSP), `commitment_id` FK (nullable), `subject_type` (deal/msp/outcome/scope_change), `signoff_method` (docusign/email/recording_timestamp/meeting_notes/click_accept), `evidence_uri`, `signed_at`, `signer_name`, `signer_role`, `signer_email`, `text_committed`, + provenance |
| `handoff_events` | Sales→CS handoff record (mirrors `sales-cs-handoff-quality-scoring` output) | `handoff_id` PK, `account_id` FK, `deal_id` FK, `handoff_at`, `quality_score` (0–100), `dimensions` (JSON: ocv_referenced, trigger_match, measurement_source, stakeholder_map, risk_flags), `pass_threshold_met` (bool), `linear_issue_uri` (nullable, for below-threshold remediation), `csm_assigned`, + provenance |
| `lifecycle_events` | Renewal/churn/downgrade/expansion events with reason codes | `event_id` PK, `account_id` FK, `event_type` (renewed/churned/downgraded/expanded/scope_changed), `event_at`, `acv_change_usd`, `seat_change`, `term_change_months`, `reason_codes` (JSON, multi-select), `reason_narrative`, `responsible_msp_id` FK (the MSP active at the time), `responsible_deal_id` FK, `linked_outcomes` (JSON: which committed outcomes were realized vs missed), + provenance |
| `lineage_traces` | Cached lineage trace results so they're reproducible + auditable | `trace_id` PK, `subject_event_id` FK → `lifecycle_events`, `traced_at`, `traced_by`, `chain` (JSON: ordered list of breadcrumbs), `gaps_found` (JSON: missing artifacts), `evidence_uris` (JSON) |

**The lineage query (the canonical join):**

```sql
-- Walk an account's churn back through MSP → discovery → awareness/attribution
SELECT
  le.event_type, le.event_at, le.acv_change_usd, le.reason_codes, le.linked_outcomes,
  msp.msp_id, msp.version, msp.signed_at, msp.mirror_source_uri,
  msp.deal_desk_review_status, msp.deal_desk_reviewed_by, msp.deal_desk_reviewed_at,
  msp.acv_at_close_usd, msp.committed_segment, msp.committed_icp_tier,
  json_group_array(json_object(
    'commitment', mc.committed_outcome_text,
    'ocv_entry', vre.name,
    'expected_value_usd', mc.expected_value_usd,
    'current_status', mc.current_status
  )) AS commitments,
  he.quality_score AS handoff_quality, he.dimensions AS handoff_dimensions,
  d.deal_id, d.closed_won_at, d.acv_usd, d.ae_owner,
  df.discovery_id, df.value_hypothesis_text, df.meddic,
  json_group_array(json_object(
    'signed_at', cs.signed_at,
    'method', cs.signoff_method,
    'evidence', cs.evidence_uri,
    'text', cs.text_committed
  )) AS signoffs,
  ra.awareness_source, ra.campaign_name, ra.lead_source, ra.icp_tier, ra.icp_type,
  ra.first_touch_at, ra.mql_at, ra.sql_at, ra.mql_to_sql_lag_days, ra.sql_to_close_lag_days,
  ra.intent_signals, ra.lookup_completeness
FROM lifecycle_events le
JOIN mutual_success_plans msp ON msp.msp_id = le.responsible_msp_id
LEFT JOIN msp_commitments mc ON mc.msp_id = msp.msp_id
LEFT JOIN value_registry_entries vre ON vre.entry_id = mc.ocv_entry_id
LEFT JOIN handoff_events he ON he.deal_id = msp.deal_id
JOIN deals d ON d.deal_id = msp.deal_id
LEFT JOIN discovery_findings df ON df.discovery_id = msp.discovery_id
LEFT JOIN customer_signoffs cs ON cs.msp_id = msp.msp_id
LEFT JOIN revenue_attribution ra ON ra.deal_id = d.deal_id
WHERE le.account_id = :account_id AND le.event_type IN ('churned','downgraded','lost_renewal')
GROUP BY le.event_id
ORDER BY le.event_at DESC;
```

This is **one SQL statement** producing the full evidentiary chain — now spanning **awareness → MQL → SQL → close → MSP (materialized + deal-desk-reviewed) → commitments → signoff → handoff → adoption outcomes → renewal/churn event**. The output is what `lineage-trace` returns and what `revenue-postmortem` aggregates across cohorts. Missing attribution branches (no marketing-lookup, no campaign attached) surface as `lookup_completeness='unavailable'` rather than silently truncating the chain — gap honesty extends to the upstream segment.

**Four new minimal skills** (in `data-layer/skills/`):

- `msp-deal-desk-review` — `/data-layer:msp-deal-desk-review --deal SF-9991 [--inspect|--approve|--request-changes "<note>"]`. The **CS deal-desk in-workflow checkpoint**: shows the auto-materialized MSP for a closed-won deal — committed ACV/segment/ICP-tier, the OCV outcomes proposed for commitment (and any unmapped), the value-register entries implied (and any unmapped), the captured revenue-attribution snapshot (campaign, lead source, ICP tier, awareness path) — and lets the reviewer approve, request changes, or annotate. Approval flips `deal_desk_review_status='approved'` and unlocks downstream handoff. No Salesforce/HubSpot login required; the materialized row is the review surface. Pre-populates a draft `msp_commitments` set from deal line items + ICP-tier outcome templates; the reviewer confirms or edits.
- `lineage-trace` — `/data-layer:lineage-trace --account ACME-001 [--event churn|downgrade|lost_renewal] [--format brief|full|markdown]`. Runs the lineage query, formats the breadcrumb chain, calls out **gaps** explicitly: "Discovery findings missing — no `discovery_id` linked to the MSP. The chain cannot prove what was promised vs. what was discovered." Also surfaces upstream-attribution gaps ("Awareness source unavailable — no marketing-lookup completed at close; awareness→MQL segment of the chain is empty"). Saves the trace to `lineage_traces` for audit.
- `revenue-postmortem` — `/data-layer:revenue-postmortem --segment SMB --period 2026Q1 [--cohort-size N]`. Cohort-level aggregate: pattern detection across recent churns/downgrades. Surfaces "Of 12 SMB churns in 2026Q1, 9 had MSP `current_status='missed'` on the same OCV entry (OCV-014 'Time-to-first-value < 30 days'). The shared discovery pattern: customer flagged 'integration speed' as a top-3 pain in 8 of those 12." Now also reads `revenue_attribution` to detect awareness-segment patterns ("7 of those 9 were inbound-organic, ICP-tier-3 — a different acquisition cohort than the rest of the SMB portfolio").
- `msp-pregate` — `/data-layer:msp-pregate --deal SF-9991`. **Pre-handoff blocker**: checks that the deal has a linked discovery_findings record, an auto-materialized MSP with **`deal_desk_review_status='approved'`**, at least N committed `msp_commitments`, a captured `revenue_attribution` snapshot (warn-only if `lookup_completeness != 'full'`), a customer_signoff with evidence_uri, and a handoff_events row from `sales-cs-handoff-quality-scoring` with `pass_threshold_met=true`. Returns block/warn/pass. The rev-ops `deal-desk-workflow-management` skill calls this as a final gate before marking a deal eligible for CS handoff.

**Coordination with the rev-ops plugin (no skill duplication):**

| Existing rev-ops skill | New role with the data spine |
|---|---|
| `deal-desk-workflow-management` | **Invokes `/data-layer:msp-deal-desk-review` as its in-workflow review surface** — the materialized MSP row is presented for CS-portion review without a CRM round-trip. Approval marks `deal_desk_review_status='approved'`. Then calls `msp-pregate` before approving handoff. Writes deal-desk approvals to `deals.pricing_terms`. |
| `sales-cs-handoff-quality-scoring` | Persists its 5-dimension verdict to `handoff_events` (quality_score, dimensions, pass_threshold_met). No more re-running to remember last week's score. |
| `deal-to-outcome-tracing` | Reads MSP commitments from `msp_commitments`; writes 30/60/90/180-day rubric assessments to `outcomes` over time. Cohort views are now SQL, not file re-parsing. |
| `outcome-to-value-tracking` | Updates `msp_commitments.current_status` and `lifecycle_events.linked_outcomes` as value is realized or missed. |
| `early-churn-downgrade-signal-detection` | When a Tier-3 signal fires, automatically runs `lineage-trace` against the account so the alert ships with the "what was promised" context, not just the "what's going wrong" signal. |
| `outcome-statement-builder` | Writes new outcome statements to `value_registry_entries` (already covered by `outcome-registry-import`). |
| `revenue-leakage-scanning` | Joins `deals` and `lifecycle_events` to detect commitment-vs-realization gaps as leakage. |

Each rev-ops skill gets a `cache_helper.sh` snippet that reads from the spine; cache-miss falls through to MCP, exactly the same opt-in pattern as the CSM skills. The verdict-write-back is the meaningful change — these skills become **producers** of spine data, not just consumers.

**Adapter spec additions** (declarative, in `adapters/*.adapter.yaml`):

- `salesforce.opportunity` / `hubspot.deal` — already mapped to `opportunities`; add a second mapping: where `is_won=true` and `closed_won_at IS NOT NULL` → `deals`. The "deal" mapping snapshots all fields at the moment `is_won` flips to true, then becomes immutable (subsequent edits to the opp update `opportunities` but not `deals`). A `quality: { rule: immutability_after_close, severity: error }` enforces it.
- **`salesforce.opportunity` / `hubspot.deal` (closed-won trigger) → `mutual_success_plans` (primary materialization path).** When the same `is_won=true` transition fires that writes `deals`, a chained materializer also creates the MSP row: `materialization_method='auto'`, `materialized_from_deal_at=now()`, `acv_at_close_usd` copied from the deal, `committed_segment`/`committed_icp_tier`/`committed_product_skus` snapshotted from the opportunity record, `status='dd_review_pending'`. Draft `msp_commitments` are seeded from deal line items joined against the `value_registry_entries` table (matched by product SKU → typical outcomes mapping, configurable per ICP tier in `data-layer/CLAUDE.md`). The auto-materialization is *deterministic and idempotent* — re-running it does not duplicate; it surfaces drift between snapshot and current opp.
- **Revenue attribution lookup chain (fires at MSP materialization).** A separate adapter `revenue_attribution_lookup.adapter.yaml` orchestrates short-lived lookup queries against marketing/CRM systems at the moment of MSP materialization, producing one `revenue_attribution` row per deal: `hubspot.contact_first_touch`, `marketo.lead_source`, `salesforce.campaign_influence`, `6sense.intent_account` (optional), `clearbit.icp_enrichment` (optional). The adapter records `lookup_completeness` (full/partial/unavailable) so gaps don't silently truncate the chain. These are **point-in-time snapshots**, not continuous syncs — `revenue_attribution` is the only spine table whose rows are deliberately stale-by-design.
- `gong.call` (pre-sales tagged) → `discovery_findings`. Adapter uses call-summary fields + custom Gong tags ("discovery", "value-hypothesis") to populate `pain_points`, `meddic`, `evidence_uris`.
- `gdrive.document` or `notion.page` (MSPs by filename pattern or template ID) → **`mutual_success_plans.mirror_*` fields only** (no longer the primary write path). When the Drive/Notion mirror exists, the parser extracts narrative extensions and writes `mirror_source_uri` + `mirror_markdown_checksum`. The structured commitment data is taken from the deal-materialized row, not from the mirror, eliminating the dual-source race.
- `docusign.envelope` + `gmail.thread` (signoff-tagged) → `customer_signoffs`. Evidence URI preserved; envelope/thread body NOT mirrored (reference-only per hybrid storage).
- `salesforce.subscription` / `stripe.subscription` / `chargebee.subscription` → `lifecycle_events`. Renewal/churn/downgrade events derived from subscription state transitions; reason codes from CSP churn-reason fields where available.

**Data-quality rules specific to the lifecycle spine:**

- `deals`: `immutability_after_close` (alert if a deal row changes post-close); `acv_usd >= 0`; `ae_owner not_null`.
- `mutual_success_plans`: `materialized_from_deal_at not_null` (every MSP must trace to a deal); `acv_at_close_usd matches deals.acv_usd` (drift between materialization and deal triggers re-materialization warning); `deal_desk_review_status='approved' before status='signed'` (review must precede signature); `signed_at not_null when status='signed'`; `mirror_markdown_checksum changed → status='superseded'` (mirror version detection only — does not invalidate the structured row).
- `msp_commitments`: `expected_value_usd > 0`; `ocv_entry_id references value_registry_entries where status='Ratified'` (no commitments against draft OCV); at MSP `deal_desk_review_status='approved'` transition: **every commitment must have a non-null `ocv_entry_id`** (no orphan commitments past deal-desk review).
- `revenue_attribution`: one row per deal; `looked_up_at within 7 days of deals.closed_won_at` (warn if attribution lookup skipped at close); `lookup_completeness in (full,partial,unavailable)`; `awareness_source not_null when lookup_completeness != 'unavailable'`.
- `customer_signoffs`: `evidence_uri not_null when signoff_method != 'click_accept'`; `signed_at within 30 days of msp.signed_at` (alert on drift).
- `handoff_events`: one per deal; if missing 30+ days after `deals.closed_won_at`, surface as a gap.
- `lifecycle_events`: `responsible_msp_id not_null when event_type IN ('churned','downgraded','lost_renewal')` — **this is the hygiene rule that makes lineage tracing possible**. Without it, post-mortems can't reach back.

**Storage impact (added to the budget):**

| Table | Rows (200-account portfolio) | Per-row | Subtotal |
|---|---|---|---|
| deals | 300 (1.5/account historic) | 5 KB | 1.5 MB |
| discovery_findings | 250 | 3 KB | 0.75 MB |
| mutual_success_plans | 350 (versioned, deal-materialized) | 3 KB | 1 MB |
| revenue_attribution | 300 (1 per deal) | 2 KB | 0.6 MB |
| msp_commitments | 1,800 (5/MSP) | 1 KB | 1.8 MB |
| customer_signoffs | 700 | 1.5 KB | 1 MB |
| handoff_events | 280 | 2 KB | 0.6 MB |
| lifecycle_events | 1,000 | 1.5 KB | 1.5 MB |
| lineage_traces | 200 (audit) | 8 KB | 1.6 MB |
| **Lifecycle subtotal** | | | **~10.5 MB** |

Lifecycle spine is ~10.5 MB on top of the ~160 MB core estimate. Well within budget.

**Why this earns its complexity.** The data layer was already pulling from the same sources (Salesforce, Gong, Drive, DocuSign, billing) for other purposes; the lifecycle spine reuses the same provenance quintet, the same identity reconciliation, the same adapter YAML pattern, the same dry-run pipeline, the same retention/security model. What's new is *tables*, not a new architecture. And the payoff is direct: when a CSM, a rev-ops lead, or a CRO asks "why did Acme churn — what did we promise them, what evidence do we have, who signed off, and what discovery insight did we miss?", the answer is **one query producing irrefutable lineage**, not a week of artifact-hunting.

### Cloud transition: Supabase as the first-class hosted deployment

When a solo CSM grows into a team, or when a single CSM wants their data accessible from multiple devices, the local DuckDB file stops being the right fit. The plan supports **two cloud-Postgres paths** — self-hosted Postgres (organizations with existing infra and DBA support) and **Supabase (recommended for teams without dedicated infra)**. Both reuse the same `lib/postgres_backend.py` against the ABC; Supabase adds a thin `lib/supabase_backend.py` subclass that wires in Supabase-specific features.

This section describes the Supabase migration design. It ships **after Phase 2 stabilizes** and is independent of Phase 3 (lifecycle spine) — the two can land in either order, and a Phase 3-active organization can migrate to Supabase without re-doing lifecycle setup.

#### Why Supabase as the recommended cloud path

- **Hosted Postgres** with no DBA burden; managed backups, point-in-time recovery, connection pooling.
- **Row-Level Security (RLS)** is first-class — necessary for multi-CSM tenant isolation when a team shares a single database.
- **Auth** ships built-in (email/password, magic link, SSO via SAML/OIDC). The plan does not need to design its own auth.
- **Realtime** Postgres-changes subscriptions enable live-updating dashboards in `dashboard-query` without polling.
- **Storage** is S3-compatible object storage with RLS-aware access — the right home for bulky `raw_blob` payloads (and, optionally, for cached document/transcript bodies if the team decides to mirror them).
- **Edge Functions** (Deno) can run ingestion server-side if the team eventually wants the data-sync managed agent to run in the cloud instead of on a CSM laptop. Not required for the migration; called out as a future option.
- **Free tier** covers solo and small-team usage; cost predictability for larger teams.

#### Migration tooling (`lib/cli.py` additions)

```
python -m data_layer.lib.cli migrate-to-supabase \
    --supabase-url https://<project>.supabase.co \
    --supabase-service-role <key>      \   # only used for initial schema + RLS push
    --tenant-id <org_slug>             \   # tenants are identified per row via tenant_id column
    --dry-run                                # default; --commit required for real migration
```

Steps the command performs:

1. **Schema push** — runs all `migrations/*.sql` against the Supabase project. Schema is Postgres-flavor compatible (the DuckDB phase already used Postgres-compatible SQL, so no rewrite).
2. **Tenant column injection** — adds `tenant_id UUID NOT NULL` to every fact table via a generated migration. Existing rows are backfilled with the supplied `--tenant-id`. New rows are stamped on insert by RLS-aware insert wrappers.
3. **RLS policy generation** — for every fact table, writes two policies: `SELECT USING (tenant_id = auth.jwt() ->> 'tenant_id')` and `INSERT/UPDATE/DELETE WITH CHECK (tenant_id = auth.jwt() ->> 'tenant_id')`. `write_back_stage` adds an additional `approver` check. `gdpr_deletions` becomes append-only via a deny-all-updates policy. Generated policies are written to `migrations/cloud/rls_<table>.sql` so they're code-reviewable.
4. **Storage bucket setup** — creates a `raw-blobs` bucket with RLS-equivalent access policy (signed-URL access only, scoped to the requester's tenant_id). Existing `raw_blob` payloads are uploaded; the column is replaced with a `raw_blob_storage_path` URI. Retention sweeps now operate on Storage objects, not table columns.
5. **Realtime publication** — enables Postgres logical replication for `signals_snapshot`, `accounts`, `lifecycle_events` (if Phase 3 active), `write_back_stage`. `dashboard-query` and the managed-agent digest skills can subscribe.
6. **Data copy** — streams rows from local DuckDB → Supabase Postgres in batches with checksum verification. Idempotent: re-running with `--resume` continues from the last verified checkpoint.
7. **Smoke test** — runs a configurable subset of the verification steps (steps 1–13 plus a new Auth + RLS verification) against the Supabase backend. Fails the migration if any step fails; rolls back the data copy with a `--rollback` flag.
8. **Local cutover** — flips `data-layer/CLAUDE.md` `backend:` from `duckdb` to `supabase`, points it at the new URL. Original DuckDB file is preserved as `~/.local/share/cs-data-layer/cs.duckdb.bak` for fallback.

#### Auth model

A new `cold-start-interview --cloud` flow asks:

- Single CSM with multi-device access vs. multi-CSM team.
- Auth method: email/password, magic link, or SSO (SAML/OIDC) — the latter targeting orgs that already federate with Okta/Azure AD.
- Tenant strategy: one tenant per organization (default) vs. one tenant per CSM (rarely needed; isolates portfolios within an org).

User identities map to Supabase Auth users. The JWT `tenant_id` claim is attached via a Supabase auth hook on signup. The local `cache_helper.sh` and `cli.py` accept a `--access-token` (or env `SUPABASE_ACCESS_TOKEN`) — short-lived, refreshed via the Supabase auth client. No service-role key is ever stored on a CSM laptop after the initial migration.

#### RLS policies — per-table summary

| Table | SELECT | INSERT/UPDATE | DELETE | Notes |
|---|---|---|---|---|
| `accounts`, `contacts`, `opportunities`, `renewals`, `signals_*`, `calls`, `outcomes`, `value_registry_entries`, `lifecycle_events`, `deals`, `discovery_findings`, `mutual_success_plans`, `msp_commitments`, `customer_signoffs`, `handoff_events`, `lineage_traces` | `tenant_id = JWT.tenant_id` | same + `auth.uid()` stamped to `created_by` | `tenant_id = JWT.tenant_id` AND role has `data_admin` claim | Standard tenant isolation. |
| `source_account_map` | `tenant_id = JWT.tenant_id` | same | same | Reconciliation is per-tenant. |
| `write_back_stage` | `tenant_id = JWT.tenant_id` | proposer = `auth.uid()`; approver-only updates require role `write_back_approver`; `diff_checksum` immutable after first write | append-only (no delete) | Approver and proposer cannot be the same user (enforced via policy with `auth.uid() != proposed_by`). |
| `gdpr_deletions` | `tenant_id = JWT.tenant_id` AND role has `compliance_reviewer` | insert only by role `compliance_officer` | denied | Append-only audit; immutability enforced by policy, not application code. |
| `data_quality_violations` | `tenant_id = JWT.tenant_id` | insert by service role only (managed-agent ingestion) | role `data_admin` only | Surfaces in dashboards; CSMs see their own tenant's violations. |
| Storage `raw-blobs` bucket | signed-URL access scoped to `tenant_id` path prefix | same | role `data_admin` only | Retention sweep is a scheduled Edge Function (or local cron, configurable). |

Policy templates land in `migrations/cloud/rls_template.sql` so a self-hosted Postgres deployment can adapt them by stripping the Supabase-Auth-specific JWT references and substituting a session-variable equivalent.

#### Realtime — live dashboards

`dashboard-query --watch <canned-query>` opens a Realtime subscription on the underlying tables and re-renders when relevant rows change. Targeted use cases:

- Portfolio triage board: live updates when `signals_snapshot.health_band` changes (no polling).
- Pending write-backs queue: approver sees new staged writes the moment they're proposed.
- Renewal countdown: live re-prioritization as `renewals.status` flips.
- Phase 3 lifecycle: `lifecycle_events` inserts trigger immediate `lineage-trace` attachment in the digest.

Subscription cost is bounded by tenant_id RLS filtering (Supabase Realtime respects RLS), so a CSM in tenant A never sees tenant B's row changes even by accident.

#### Storage — `raw_blob` and bulky payloads

The hybrid storage model carries through with one change: instead of nulling `raw_blob` after 30 days inside the Postgres row, the Supabase backend writes `raw_blob` to the `raw-blobs` Storage bucket at path `<tenant_id>/<table>/<row_pk>/<fetched_at>.json`. The column becomes `raw_blob_storage_path` (URI). Retention sweep deletes Storage objects past the horizon. This keeps the Postgres rows narrow (faster queries) and uses S3-equivalent economics for the bulky audit payloads. Document and transcript bodies remain reference-only (the same decision as the local design); the team can opt-in to mirroring select document bodies into Storage if a use case emerges, but defaults stay reference-only to limit PII concentration.

#### Cost + data residency caveats

- **PII leaves the laptop.** This is the central tradeoff. The cold-start `--cloud` flow surfaces it explicitly: the CSM acknowledges that data will sit in Supabase's infrastructure (region-selectable; the user picks the region during project setup). Encryption in transit + at rest is on by default; the user can enable an additional vault-managed encryption layer (Supabase Vault) for the most sensitive columns (`raw_blob_storage_path` payloads, `customer_signoffs.text_committed`, etc.).
- **Cost predictability.** The cold-start estimates monthly cost based on portfolio size (rows, raw_blob storage, Realtime concurrent connections). The estimate is conservative; documented as "best-effort, validate after first month."
- **Vendor lock-in.** Mitigated by keeping the schema Postgres-generic. A team can `pg_dump` the Supabase project at any time and restore to self-hosted Postgres with a one-line CLI flip — only the Supabase-specific features (Auth, RLS using `auth.jwt()`, Realtime, Storage) need substitution, and templates for each are in `migrations/cloud/`.
- **Compliance.** Supabase is SOC 2 Type 2 and offers HIPAA-eligible plans. Organizations with stricter compliance requirements (FedRAMP, GxP, data-residency-locked regions Supabase doesn't serve) should stay on self-hosted Postgres; the cold-start asks and routes accordingly.

#### Files added for the Supabase path

- `data-layer/lib/postgres_backend.py` — generic Postgres backend (used by self-hosted Postgres too)
- `data-layer/lib/supabase_backend.py` — extends postgres_backend with Auth, RLS-aware queries, Realtime, Storage
- `data-layer/lib/cli.py` — adds `migrate-to-supabase`, `verify-rls`, `rotate-token`, `cloud-cost-estimate`
- `data-layer/migrations/cloud/rls_template.sql` — generic RLS policies (Supabase-flavored, substitutable for self-hosted)
- `data-layer/migrations/cloud/rls_<table>.sql` — generated per-table policies, code-reviewable
- `data-layer/migrations/cloud/storage_bucket_setup.sql` — bucket creation + access policies
- `data-layer/migrations/cloud/realtime_publication.sql` — logical replication config
- `data-layer/skills/cloud-cold-start/SKILL.md` — `--cloud` flow for the migration cold-start
- `data-layer/skills/cloud-cost-estimate/SKILL.md` — pre-migration cost preview
- `data-layer/evals/test_rls_isolation.py` — RLS smoke: tenant A cannot read tenant B's rows even with a valid JWT
- `data-layer/evals/test_storage_signed_urls.py` — Storage RLS smoke: signed URLs scoped to tenant_id path prefix
- `ARCHITECTURE.md` — appendix documenting the cloud transition design and the "stay on self-hosted Postgres if Supabase doesn't fit your compliance posture" decision tree

#### Verification additions (folded into Verification section)

- Steps 26–30 will exercise: migration dry-run, RLS isolation, Realtime subscription delivery, Storage signed-URL access, rollback to DuckDB.

### Migrations — hybrid auto/manual

Cold-start asks: *"Auto-migrate the database schema on update? (default: yes for solo, manual for team)"*. Setting stored in `data-layer/CLAUDE.md` → `auto_migrate: true|false`. On `backend.connect()`:
- `auto_migrate=true`: run pending migrations transparently.
- `auto_migrate=false`: detect pending migrations, surface a warning, refuse queries until `/data-layer:run-migrations` runs.
- Every migration file ships with a `backfill_checker` block that asserts ≥80% backfill coverage before COMMIT.

### Solo deployment defaults

- DB path: `~/.local/share/cs-data-layer/cs.sqlite` (XDG-compliant); permissions `0600`
- Confidence: 1d High / 7d Moderate / >7d Low (per source, tunable)
- Sync cadence: daily 06:00; per-source soft-fail
- Write-back: pre-populated safe allowlist; AUQ prose fallback disabled for approvals
- Auto-migrate: yes (solo default)

---

## Files to create or modify

**Critical new seeds:**
- `data-layer/reference/schema.sql`, `lib/backend.py`, `lib/sqlite_backend.py`, `lib/identity.py`, `lib/ingestion.py`, `lib/adapter_executor.py`, `lib/normalize.py`, `lib/secret_scrub.py`, `lib/sanitize.py`, `lib/cli.py`, `lib/cache_helper.sh`
- `data-layer/adapters/hubspot.adapter.yaml`, `gainsight.adapter.yaml`, `gong.adapter.yaml`, `adapter.schema.yaml`
- `data-layer/skills/{cold-start-interview,customize,data-sync,data-hygiene-check,dashboard-query,write-back-stage,outcome-registry-import,gdpr-delete,gdpr-export,run-migrations}/SKILL.md`
- `data-layer/CLAUDE.md`, `README.md`, `ARCHITECTURE.md`, `.mcp.json`, `.gitignore`
- `data-layer/migrations/001_init.sql`
- `data-layer/samples/seed-{crm,csp,calls}.json`
- `data-layer/evals/test_{schema_contract,sync_golden_path,adapter_conformance,injection_safe}.py`
- `managed-agent-cookbooks/data-sync/{agent.yaml, README.md, subagents/}`

**Light touches:**
- `README.md` — add `data-layer/` row; bump plugin badge 6 → 7
- `shared/cross-skill-registry.md` — 14 new commands (incl. msp-deal-desk-review, lineage-trace, revenue-postmortem, msp-pregate)
- `shared/cs-domain-model.md` — §8 "Read-Through Cache" reference
- `shared/revops-domain-model.md` — §11 "Lifecycle Spine: deal → MSP → outcome → renewal" with the canonical lineage query
- `deployment-cookbooks/solo-csm-cookbook.md` — §0.9 install + cold-start inheritance note
- `csm/skills/cold-start-interview/SKILL.md` — one new question that auto-generates data-layer config
- `csm/managed-agents/health-watcher/`, `churn-signal-digest/` — pre-wire `cache_helper.sh` (still soft-fail to MCP); on Tier-3 churn signal, automatically attach a `lineage-trace` summary
- `csm/skills/{account-research,call-prep,risk-flag,renewal-readiness,qbr-builder}/SKILL.md` — add 6-line cache helper snippet
- `rev-ops/skills/sales-cs-handoff-quality-scoring/SKILL.md` — write verdict to `handoff_events`
- `rev-ops/skills/deal-to-outcome-tracing/SKILL.md` — read commitments from `msp_commitments`; write checkpoint rubric levels to `outcomes`
- `rev-ops/skills/outcome-to-value-tracking/SKILL.md` — update `msp_commitments.current_status` and `lifecycle_events.linked_outcomes`
- `rev-ops/skills/deal-desk-workflow-management/SKILL.md` — invoke `/data-layer:msp-deal-desk-review` as the CS-portion in-workflow review surface; then call `msp-pregate` as the final gate before handoff approval
- `rev-ops/skills/early-churn-downgrade-signal-detection/SKILL.md` — Tier-3 alerts attach `lineage-trace` output
- `managed-agent-cookbooks/value-map-system/README.md` — "Downstream subscribers" note
- `CHANGELOG.md`, `.gitignore`

**Explicitly NOT changing:**
- Value Map schema or write-authority rules
- The 70+ skills that don't benefit from the cache

---

## Verification

End-to-end smoke test. Steps 1–8 are Phase 1; steps 9–19 are Phase 2; steps 20–25 + 31–33 are Phase 3 (only run when Phase 3 activates); steps 26–30 are the Supabase cloud transition (only when the team migrates).

1. **Cold-start via CSM inheritance** — `/csm:cold-start-interview`, answer "yes" to data-layer activation; verify `data-layer/CLAUDE.md` was auto-generated with inherited thresholds and the safe-default write-back allowlist.
2. **Seed sample data** — `python -m data_layer.lib.cli load-fixture --file samples/seed-crm.json --source salesforce` (then HubSpot for the same accounts under different IDs).
3. **Identity reconciliation** — verify `SELECT * FROM source_account_map WHERE canonical_account_id='ACME-001'` returns two rows (one Salesforce, one HubSpot) after both syncs.
4. **Time-series correctness** — sync the same Gainsight health snapshot twice; verify `signals_snapshot` has one row (upserted) and `signals_history` has two (append-only).
5. **Hygiene check** — artificially age a row + drop a custom CRM field from a synced source; `/data-layer:data-hygiene-check --full` flags both (`[stale — N days]` and `adapter schema drift: hubspot.company.name no longer present`).
6. **Dashboard query** — `/data-layer:dashboard-query --canned health-watcher-alerts` returns prioritized account list; reviewer note shows `[CS Platform ✓ live]` + data-as-of + confidence.
7. **Write-back hardening** — propose a CSM-owner change; verify diff preview, AUQ prose-fallback is NOT proceeding-by-default; approve via explicit response; verify `diff_checksum` + approver logged.
8. **Rate-limit** — script 11 staged writes in one minute; backend rejects the 11th with `too_many_pending`.
9. **Secret scrubbing** — inject a fixture with a fake AWS key in a custom field; verify it's redacted in `raw_blob` and logged in `audit/scrub.log`.
10. **Injection safety** — `pytest data-layer/evals/test_injection_safe.py` poisons `internal_notes` with `<script>` and a SQL fragment; verify normalized column is sanitized while `raw_blob` is preserved.
11. **GDPR delete** — `/data-layer:gdpr-delete --account ACME-001`; verify columns wiped, immutable `gdpr_deletions` row written, CRM write-back staged.
12. **Migration backfill** — drop a synthetic `002_add_test_column.sql` with a deliberately failing backfill assertion; verify migration rolls back and surfaces the failure.
13. **Cache helper opt-in** — invoke `/csm:account-research ACME-001` twice; verify second call returns sub-100ms (cache hit); kill the SQLite file; verify third call falls back to MCP within the 2s timeout.
14. **Source-mapping inspect** — `python -m data_layer.lib.cli inspect --source hubspot`; verify the diff identifies a deliberately-renamed field as drift and lists three live-only fields as candidates.
15. **Dry-run pull** — `python -m data_layer.lib.cli dry-run --source salesforce --sample 5`; verify the report shows reconciliation collisions, confidence-band stamps, and sanitization events without writing to the real cache. Confirm a JSON copy lands in `audit/dry-runs/`.
16. **Auto-dry-run on adapter change** — edit `adapters/hubspot.adapter.yaml`, then run `/data-layer:data-sync --source hubspot`; verify it refuses to commit until a dry-run has been viewed and approved.
17. **Transforms + quality rules** — set `arr_usd` source field to cents and add `divide_by: 100` transform; verify normalized output is correct dollars. Inject a record with negative ARR; verify `range` quality rule writes an error to `data_quality_violations` and (with `block_commit_on_quality_fail: true`) the sync aborts.
18. **Storage budget** — `/data-layer:data-hygiene-check --storage` reports per-table size and total; assert under 500 MB after a full 200-account seed. Confirm `signals_history` is bounded by retention setting.
19. **Retention prune** — manually backdate signal observations past the retention horizon; run `python -m data_layer.lib.cli prune --dry-run` then `--commit`; verify rows are gone and `raw_blob` is nulled on rows beyond `raw_blob_days`, while normalized columns remain.
20. **Lineage spine — happy path** — seed ACME-001 with a `deals` row, a `discovery_findings` row, a signed `mutual_success_plans` with 3 `msp_commitments`, a `customer_signoffs`, a `handoff_events` with `pass_threshold_met=true`, then a `lifecycle_events` row with `event_type='churned'` and `responsible_msp_id` set. Run `/data-layer:lineage-trace --account ACME-001 --event churn --format full`. Verify the breadcrumb chain is complete: discovery → MSP → 3 commitments (with current_status) → signoff (with evidence URI) → handoff score → close → churn event. Verify a `lineage_traces` audit row was written.
21. **Lineage spine — gap detection** — same as above but delete the `discovery_findings` row. Re-run `lineage-trace`. Verify the output calls out "Discovery findings missing — the chain cannot prove what was promised vs. what was discovered" rather than silently producing a partial trace.
22. **MSP pre-handoff gate** — create a deal with no signed MSP. Run `/data-layer:msp-pregate --deal SF-9991`. Verify it returns `block` with the list of missing artifacts. Add a signed MSP with commitments + a signoff + a passing handoff_events row. Re-run; verify `pass`.
23. **Cohort post-mortem** — seed 12 SMB churn events across the lifecycle spine, 9 sharing a missed OCV entry. Run `/data-layer:revenue-postmortem --segment SMB --period 2026Q1`. Verify the output surfaces the shared OCV-miss pattern and the common discovery pain point.
24. **Rev-ops skill integration** — invoke `/rev-ops:sales-cs-handoff-quality-scoring` for a deal; verify the verdict is persisted to `handoff_events` (not just printed). Re-run the same skill 5 minutes later; verify it does not re-score but reads from the cached row (with freshness reviewer note). Then invoke `/rev-ops:deal-to-outcome-tracing` at the 30-day checkpoint; verify it writes a new `outcomes` row with `rubric_level` and reads MSP commitments from `msp_commitments`, not the source MSP file.
25. **Deal immutability** — close-won an opportunity (writes both `opportunities` and `deals`). Edit the opportunity's `acv_usd` field upstream; re-sync. Verify `opportunities.acv_usd` updates but `deals.acv_usd` does NOT; verify the `immutability_after_close` quality rule fires an error if the deal row is touched.
26. **Supabase migration dry-run** — provision a free-tier Supabase project, run `python -m data_layer.lib.cli migrate-to-supabase --dry-run`. Verify schema push, tenant_id backfill plan, RLS policy generation, Storage bucket plan, and Realtime publication plan are all reported with no errors. No data should land yet.
27. **Supabase migration commit + RLS isolation** — run with `--commit`. Provision two test tenants (A and B). Insert fixture rows into both. Authenticate as tenant A's user; verify queries return only tenant A's rows. Authenticate as tenant B's user; verify the same. Attempt cross-tenant SELECT/UPDATE/DELETE via direct SQL; verify all are denied by RLS.
28. **Realtime live subscription** — start `/data-layer:dashboard-query --watch portfolio-health` as tenant A. Insert a new `signals_snapshot` row as tenant A via a separate process. Verify the dashboard re-renders within 2 seconds. Insert a row as tenant B; verify tenant A's dashboard does NOT re-render (Realtime respects RLS).
29. **Storage signed-URL access** — write a `raw_blob` to Storage during ingestion. Fetch the corresponding row as tenant A; verify the returned signed URL resolves to the payload. Attempt to fetch the same signed URL as tenant B (or unauthenticated); verify access denied.
30. **Rollback path** — re-point `data-layer/CLAUDE.md` `backend:` from `supabase` back to `duckdb` using the preserved `.duckdb.bak`. Verify all skills continue to function against the local backend. Confirm the Supabase project remains intact (rollback is local-config only; no destructive ops on the cloud project).
31. **MSP auto-materialization on close-won** — seed an opportunity with `is_won=false`, sync. Verify no `mutual_success_plans` row exists. Flip the opportunity to `is_won=true` with `closed_won_at` set; re-sync. Verify that *one* MSP row is created with `materialization_method='auto'`, `materialized_from_deal_at` matching the sync time, `acv_at_close_usd` matching the deal, `committed_segment` / `committed_icp_tier` / `committed_product_skus` snapshotted, and `status='dd_review_pending'`. Verify a draft `msp_commitments` set was seeded from the deal line items joined against `value_registry_entries`. Re-sync once more — verify materialization is idempotent (no duplicate row, no commitment duplication; drift between snapshot and current opp surfaces as a quality warning).
32. **CS deal-desk review checkpoint** — run `/data-layer:msp-deal-desk-review --deal SF-9991 --inspect` against the materialized MSP from step 31. Verify the output shows the snapshotted deal fields, the draft commitments with OCV alignment status (mapped/unmapped), the revenue-attribution snapshot if present, and the alignment gaps. Run `--request-changes "Add OCV-007 commitment for ICP-tier-2 onboarding velocity"`; verify `deal_desk_review_status='changes_requested'` and the note is persisted. Then run `--approve`; verify `deal_desk_review_status='approved'`, `deal_desk_reviewed_by` / `deal_desk_reviewed_at` populated, and the `msp-pregate` skill (when invoked) now returns `pass` for this deal (assuming the other prereqs are met). Attempt `--approve` on an MSP with any commitment missing `ocv_entry_id`; verify it refuses with a clear error message identifying the orphan commitments.
33. **Revenue attribution lookup chain** — at MSP materialization, verify the `revenue_attribution_lookup` adapter fires and writes one row to `revenue_attribution` with `awareness_source`, `campaign_name`, `lead_source`, `icp_tier`, `mql_at` / `sql_at` / `mql_to_sql_lag_days`, and `lookup_completeness`. Test the three lookup-completeness paths: (a) full — all configured sources return data; (b) partial — Marketo returns lead_source but 6sense is offline, verify `lookup_completeness='partial'` and `lookup_sources` reflects what succeeded; (c) unavailable — disable all lookup sources, verify `lookup_completeness='unavailable'`, `awareness_source` is null, and the `lineage-trace` skill surfaces the gap explicitly rather than producing a silent truncation. Verify `revenue_attribution` rows are NOT re-fetched on subsequent syncs (point-in-time snapshot, by design).

---

## Risks & tradeoffs (post-review)

1. **SQLite ceiling.** Cold-start warns if the configured portfolio exceeds 50 accounts and recommends Postgres for v2; tests assert query latency on a 200-account seed stays under 500ms for canned queries.
2. **Adapter drift.** YAML adapter specs validated against live MCP responses by `data-hygiene-check`; drift surfaces at ingest, not at query time. Community PRs are YAML diffs (lower bar than Python).
3. **Write-back blast radius.** Rate-limited, diff-confirmed, AUQ-fallback disabled, audit-logged. Off-by-default-but-pre-populated allowlist trades a small security loss for adoption.
4. **Cache adoption.** Pre-wiring `health-watcher` + `churn-signal-digest` covers the highest-frequency cost. If only the pre-wired skills adopt, the cache still pays for itself; if more adopt, value compounds.
5. **OCV sync drift.** Markdown-checksum gate + reverse-sync skill make the rhythm visible. `data-hygiene-check` lists dirty OCV entries so they don't quietly diverge.
6. **PII on a laptop.** `chmod 600`, secret-scrub on ingest, secure-delete on uninstall, encrypted-backup default. For team Postgres deployments, RLS template lives in `ARCHITECTURE.md`.
7. **Silent adapter drift / bad reconciliation.** Inspect → map → dry-run → commit loop fires on cold-start, on every adapter YAML change, and on every custom-field allowlist change. The failure modes surface in a sandboxed in-memory pipeline before any persistent write — much cheaper to catch than after the cache is polluted.
8. **Storage growth.** Hybrid mirror (structured rows mirrored; transcripts, document bodies, attachments referenced) + retention policy on `signals_history` and `raw_blob` keeps a 200-account portfolio under 200 MB. `data-hygiene-check --storage` surfaces the breakdown. Hard ceilings: 500 MB warn, 1 GB recommend Postgres v2 migration.
9. **Transform/quality-rule sprawl.** Rules live in adapter YAML alongside the field map (one place, one diff). Built-in op registry covers the common cases; custom ops register in `lib/transforms.py` and `lib/quality.py` with tests required in `evals/`. Severity `error` vs `warn` keeps the abort-on-violation behavior explicit and tunable.
10. **Lifecycle-spine source-of-truth — MSP as normalization path.** The MSP is the **deal-derived normalized post-sale record** — auto-materialized from the Salesforce/HubSpot opportunity at close-won, then reviewed in-workflow by the CS portion of the deal desk via `/data-layer:msp-deal-desk-review`. The materialized row is authoritative for queries and lineage; the Drive/Notion MSP document, when present, acts as a human-readable mirror (`mirror_source_uri`, `mirror_markdown_checksum`) that can carry narrative extensions but does not own the structured fields — eliminating the dual-source race that historically created drift between "what the doc says" and "what the deal says." The MSP is also the **first checkpoint for outcome/value alignment** (every commitment must reference a `value_registry_entries` row before deal-desk approval) and the **initial write-point for revenue observability**: at materialization, the `revenue_attribution` lookup chain queries marketing/CRM systems to snapshot awareness path, campaign, lead source, ICP tier — completing the awareness→current-state picture without requiring upstream marketing tools to be continuously synced (deliberate point-in-time snapshot). Source-of-truth ownership for non-materialized artifacts is unchanged: signoffs stay in DocuSign/email; `deals` remains an immutable Salesforce snapshot at close; the live opportunity stays in the CRM. `lineage-trace` always cites the originating system URIs so any breadcrumb can be opened in source.
11. **Rev-ops adoption coupling.** Three existing rev-ops skills (`sales-cs-handoff-quality-scoring`, `deal-to-outcome-tracing`, `outcome-to-value-tracking`) become **producers** of spine data, not just consumers. The first release ships their `cache_helper.sh` write-back snippets pre-wired, same as the CSM consumer pattern. If a rev-ops skill is not updated, the cache still receives data via the adapter pipeline (read from CRM directly); the verdict-write-back path is the optimization, not a hard dependency.
12. **Lineage gap honesty.** Missing artifacts (no discovery_findings, no signed MSP, no signoff) are explicitly called out in the `lineage-trace` output rather than producing a silent partial chain. The `lifecycle_events.responsible_msp_id not_null` quality rule (when event_type is churned/downgraded/lost_renewal) makes the gap visible at ingest time. Hygiene digests surface accounts whose lineage chain has structural breaks before a post-mortem question is asked.
13. **Cloud migration data residency + PII concentration.** Migrating to Supabase moves PII off the CSM laptop into Supabase's infrastructure. The `cold-start-interview --cloud` flow surfaces this explicitly and requires user acknowledgement before migration. Region is user-selected at Supabase project setup. Mitigations: RLS enforced at the database level (not application), Storage uses signed-URL access scoped to tenant_id, Supabase Vault available for an additional encryption layer on the most sensitive columns/payloads. Compliance routing: organizations with FedRAMP, GxP, or specific data-residency requirements that Supabase doesn't serve stay on self-hosted Postgres — the cold-start asks and routes accordingly.
14. **Cloud cost predictability.** Supabase free tier covers solo and small-team usage; cost scales with rows, raw_blob Storage volume, and Realtime concurrent connections. `cloud-cost-estimate` skill produces a pre-migration projection. Documented as best-effort; teams validate after first month. Self-hosted Postgres remains the cost-bounded alternative for teams that want to keep cloud bills flat.
15. **Vendor lock-in (Supabase).** Mitigated by keeping the schema Postgres-generic and the ABC Postgres-agnostic. `pg_dump` from Supabase + restore to self-hosted Postgres is a documented one-line CLI flip; only Supabase-specific features (Auth, RLS using `auth.jwt()`, Realtime, Storage) need substitution. Templates for each substitution live in `migrations/cloud/` so the off-ramp is exercised, not theoretical.
