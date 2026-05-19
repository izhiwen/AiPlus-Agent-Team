# Goal G-AT-TOKEN-COST-1 — New crate aiplus-token-cost

**Status**: PROPOSED — Owner-authorized 2026-05-19. Wave 2-C (parallel with B and D).
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-TOKEN-COST-1`

---

## TL;DR

Ship a new sibling crate `aiplus-token-cost` that reads token usage from existing dispatch-log and turns it into USD cost rollups. Source of pricing data: LiteLLM's community-maintained `model_prices_and_context_window.json`. Display 3 time windows (1h / 8h / 24h) + top-5 tasks by cost. Hourly snapshot to a local JSONL file.

## Success criterion (single sentence)

> Owner runs `aiplus token-cost` and sees: past 1h / 8h / 24h token totals + USD equivalents + top-5 most expensive recent tasks; if LiteLLM JSON is unreachable, falls back to embedded pricing constants without erroring; if Owner has `.aiplus/pricing.toml` it overrides both sources.

## Goals (D1-D6)

- **D1**: New crate `crates/aiplus-token-cost/` added to workspace (library + integrated into `aiplus-cli` for the subcommand).
- **D2**: Pricing source: LiteLLM JSON primary (fetched once per day, cached at `~/.cache/aiplus-token-cost/pricing.json`) + embedded constants fallback + `.aiplus/pricing.toml` local override.
- **D3**: Cost rollups: 1h / 8h / 24h windows + top-5 most expensive tasks. Both per-task primary view and per-role detail view.
- **D4**: New `aiplus token-cost` subcommand registered in commands.rs. Flags: `--by-role`, `--window=<1h|8h|24h>` (default: show all 3), `--top-n=<N>` (default 5).
- **D5**: Hourly snapshot: append a roll-up entry to `.aiplus/agents/token-cost-snapshots.jsonl` once per hour (when subcommand is run; skip if last snapshot < 1h old).
- **D6**: Graceful handling of missing usage_tokens in dispatch-log rows (some adapters/runs return null) — count tokens=0 USD=0 for that entry, not error.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/token-cost-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/token-cost-impl-notes.md` | PENDING — CEO |
| 4 | `crates/aiplus-token-cost/` (Cargo.toml + src/lib.rs + src/pricing.rs + src/rollup.rs etc.) | PENDING — CEO |
| 5 | Workspace `Cargo.toml` adds the new crate to members | PENDING — CEO |
| 6 | `aiplus-cli` integration: `commands.rs` registers `token-cost` subcommand | PENDING — CEO |
| 7 | New integration tests in `crates/aiplus-token-cost/tests/` | PENDING — CEO |
| 8 | Embedded pricing constants (fallback) — covers Anthropic Claude family + OpenAI GPT family at minimum | PENDING — CEO |

## Phases

```
Phase 1 (CEO codex, ~0.5-1 day):
  - Design crate structure
  - Design pricing data model (provider × model → input/output per-token USD)
  - Design LiteLLM JSON fetch + cache + parse
  - Design dispatch-log walker + time-window aggregator
  - Design top-N task selector
  - Define embedded constants (sufficient for fallback)

Phase 2 (CEO codex, ~2-3 days):
  - crate skeleton + workspace integration
  - pricing module (fetch + cache + override + embedded fallback)
  - rollup module (walk dispatch-log + compute windows)
  - CLI subcommand integration
  - Snapshot writer

Phase 3 (CEO codex, ~0.5-1 day):
  - Tests for all 6 D-goals
  - Live smoke: cargo build + run `aiplus token-cost` in a project with dispatch-log
  - Phase 3 evidence trailer
```

**Total target**: ≤ 5 days.

## Out of scope

- **Real-time alerts** when cost exceeds budget — opt-in feature for v2 if needed
- **Multi-currency** — USD only in v1
- **Historical cost dashboards** beyond JSONL snapshots — Owner can grep snapshots manually for v1
- **Cost-aware scheduling** — coordinator doesn't read pricing to pick cheap providers
- **Adapter trace files as data source** — v1 uses dispatch-log only; trace files are out of scope
- **Auditor-related fields** (auditor_verdict cost) — Wave 1 removes that feature; if it's present in legacy dispatch-log rows, count tokens=0

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| LiteLLM JSON URL changes / repo moves | Hardcoded URL in v1; refactor to config if it breaks in v2 |
| LiteLLM JSON schema changes | Embedded fallback is always present; fetch failure / parse failure → use fallback |
| Network call adds startup latency | Cache lives for 24h; only first call of day pays network cost; can be background-refreshed |
| `usage_tokens` field absent in some dispatch-log rows | Count as 0/0 (documented behavior; CEO's Phase 1 design captures this) |
| Owner runs in offline environment | Embedded constants + local override both work without network |
| Snapshot file grows unbounded | Document rotation policy (e.g., keep last 30 days); v2 can add auto-rotation |
| `aiplus-cli` workspace member count grows | Acceptable; new crate is purpose-isolated |

## Acceptance criteria

- ✓ All 6 D-goals met
- ✓ `cargo test --workspace` PASS (existing tests unchanged + new tests added)
- ✓ Live: `aiplus token-cost` runs in a project with dispatch-log, produces 3-window output + top-5 list
- ✓ Live: `aiplus token-cost --by-role` produces per-role breakdown
- ✓ Live: temporarily blocking network → still works (uses embedded fallback)
- ✓ Live: `.aiplus/pricing.toml` with custom prices overrides both LiteLLM and embedded
- ✓ `aiplus doctor` doesn't break with new crate in workspace

## Dependencies

- Wave 1 (G-AT-AUDITOR-REMOVE-1) merged to main
- LiteLLM JSON at `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json` (community-maintained, MIT license)
- Existing dispatch-log.jsonl format and `usage_tokens` field semantics (preserved through v0.6.4)

## Next action

After Wave 1 merges, Advisor dispatches CEO for this goal in parallel with B and D.

---

— Advisor, 2026-05-19, Wave 2-C of post-4-goal sprint.
