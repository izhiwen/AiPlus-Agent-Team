# Token Cost Crate — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session (Wave 2-C; parallel with B and D)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-TOKEN-COST-1.md`
- Existing dispatch-log writer (data source): `aiplus-public/crates/aiplus-cli/src/agent/route.rs`
- Existing `usage_tokens` field shipped in `coordinator_decision` events (v0.3.0 P0) and per-role dispatch events
- LiteLLM pricing source: `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.token-cost/`
- Branch:    `feat/token-cost`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.token-cost -b feat/token-cost`

Shared-file ownership (CRITICAL — Wave 2 parallel):
- This session OWNS: `crates/aiplus-token-cost/` (new crate, all files), workspace `Cargo.toml` (only the `members = [...]` addition), `crates/aiplus-cli/src/agent/commands.rs` (only the new `token-cost` subcommand registration — DO NOT MODIFY EXISTING SUBCOMMANDS in commands.rs), new tests in the new crate
- This session must NOT modify: `crates/aiplus-cli/src/agent/route.rs` (Wave 2-D), `coordinator.rs` (Wave 2-B), `doctor.rs` (Wave 2-D), CONTRACT.md (FROZEN), adapter code, calibration fixture
- **commands.rs is shared with Wave 2-D** for the `--quiet` flag on doctor. To avoid collision: this session only ADDS a new `TokenCost` enum variant + subcommand handler at the bottom of commands.rs; do NOT touch any existing subcommand definitions

## Scope — new sibling crate

Add a new Rust crate `aiplus-token-cost` to the workspace. Library + thin CLI integration via `aiplus-cli`.

## Crate structure (suggested; CEO finalizes)

```
crates/aiplus-token-cost/
├── Cargo.toml
├── src/
│   ├── lib.rs              # public API
│   ├── pricing.rs          # provider × model → per-token USD; fetch / cache / override / embedded
│   ├── rollup.rs           # dispatch-log walker + time-window aggregator
│   ├── snapshot.rs         # hourly snapshot writer
│   ├── embedded.rs         # embedded fallback pricing constants
│   └── error.rs            # error types
├── tests/
│   └── token_cost_smoke.rs
└── README.md               # crate-level docs
```

## Pricing data model

```rust
// In pricing.rs
pub struct PricePerToken {
    pub input_usd: f64,    // USD per input token (e.g., 0.000003)
    pub output_usd: f64,   // USD per output token
}

pub struct PricingTable {
    // key: (provider_name, model_name) where provider is "anthropic"/"openai"/etc.
    // and model is "claude-3-5-sonnet-20241022" / "gpt-4o" / etc.
    entries: HashMap<(String, String), PricePerToken>,
}

impl PricingTable {
    pub fn load() -> Self {
        // 1. Try local override (.aiplus/pricing.toml in project root)
        // 2. Try local cache (~/.cache/aiplus-token-cost/pricing.json, max age 24h)
        // 3. Try fetch LiteLLM JSON (https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json), write to cache on success
        // 4. Fall back to embedded constants
        ...
    }
    
    pub fn lookup(&self, provider: &str, model: &str) -> Option<&PricePerToken> { ... }
}
```

## Embedded fallback (must ship)

Minimum coverage for embedded constants — pull representative prices from LiteLLM JSON at build-time. At least:
- Anthropic: claude-opus-4-7, claude-sonnet-4-6, claude-haiku-4-5-20251001 (or whichever current models per knowledge cutoff)
- OpenAI: gpt-5 family, gpt-4o family
- (Add others as found in real dispatch-log examples)

Hardcode as `const` in `embedded.rs`. Document update procedure in crate README: "Run `aiplus token-cost --refresh-embedded` to write embedded constants from current LiteLLM JSON" — but this command is itself out of scope for v1, just document.

## Rollup logic

```rust
// In rollup.rs
pub struct RollupResult {
    pub window_label: String,    // "1h" / "8h" / "24h"
    pub total_tokens: u64,
    pub total_usd: f64,
    pub top_tasks: Vec<TaskCost>,  // sorted desc by USD
    pub by_role: HashMap<String, RoleCost>,
}

pub fn rollup_from_dispatch_log(
    log_path: &Path,
    pricing: &PricingTable,
    now: DateTime<Utc>,
    windows: &[Duration],
) -> Result<Vec<RollupResult>> {
    // walk dispatch-log.jsonl
    // for each entry with usage_tokens != null:
    //   - extract provider + model
    //   - look up pricing
    //   - compute USD = input * input_usd + output * output_usd
    //   - bucket by task (use decisionId or task_excerpt as task key)
    //   - bucket by role
    //   - sum into window if entry timestamp is within window
    // null usage_tokens → count 0/0 (per D6)
    // unknown (provider, model) pair → count tokens but USD=0 + log a WARN somewhere
    ...
}
```

## CLI subcommand

In `aiplus-cli/src/agent/commands.rs`, add new enum variant + handler:

```rust
// Pseudocode
pub enum AgentSub {
    // ... existing variants ...
    TokenCost {
        by_role: bool,           // --by-role
        window: Option<String>,  // --window=<1h|8h|24h> ; default: all 3
        top_n: u8,               // --top-n=<N> ; default 5
    },
}
```

Dispatch from `aiplus agent token-cost` OR top-level `aiplus token-cost` (CEO picks; either works).

## Snapshot writer

```rust
// In snapshot.rs
pub fn maybe_write_hourly_snapshot(
    log_path: &Path,
    snapshot_path: &Path,        // .aiplus/agents/token-cost-snapshots.jsonl
    now: DateTime<Utc>,
) -> Result<bool> {  // true if written, false if last snapshot < 1h old
    // 1. Read last snapshot from path (tail it)
    // 2. If last snapshot.timestamp + 1h > now → skip, return false
    // 3. Otherwise compute current rollup + append as new line
}
```

Called from the subcommand handler. No background thread for v1 (subcommand invocation is the trigger).

## Phase structure

### Phase 1 — Write `docs/proposals/token-cost-impl-notes.md`

Required sections:
1. Final crate structure
2. Pricing data flow (which source wins in which order; cache TTL; failure paths)
3. Rollup algorithm pseudocode + edge cases (missing fields, unknown models, partial entries)
4. Subcommand UX (exact output format + flags)
5. Snapshot file format + rotation considerations (note future rotation, but don't implement)
6. Embedded constants chosen (which models, which prices, sourced from which LiteLLM JSON snapshot)
7. CHANGELOG ## 0.6.5 draft text — the "new token-cost crate" paragraph
8. Test plan

### Phase 2 — Implement

Order:
1. Crate skeleton + workspace member + Cargo.toml
2. Embedded constants + pricing.rs (with fallback chain)
3. Rollup walker + tests using fixture dispatch-log
4. Snapshot writer + tests
5. CLI subcommand registration + smoke test
6. Compile + run on real `.aiplus/agents/dispatch-log.jsonl`

### Phase 3 — Evidence

- All tests green
- `cargo build --workspace` clean
- Live: `aiplus token-cost` produces output in a test project
- Live: `--by-role` works
- Live: blocking network (offline test) still works via embedded
- Live: with `.aiplus/pricing.toml` local override → override takes precedence
- Phase 3 trailer in impl-notes

## STOP rule

- **STOP only on `_REGRESSION_`** (existing 360 aiplus-cli tests fail after workspace addition) or `_SCOPE-BREACH_` (touched a not-owned file)
- **Do NOT STOP on `_IMPL-BUG_`**
- Retry-once gate mandatory

**Special**: if LiteLLM JSON URL is unreachable during your development → use embedded fallback. Do NOT block on network availability.

## Conformance / verification

- All ACs in goal doc met
- `cargo test --workspace` PASS
- `cargo clippy -p aiplus-token-cost -- -D warnings` clean (new crate; no pre-existing debt)
- Live demos for 3 sub-features (Litellm fetch, embedded fallback, local override)

## Scope fence

- **IN**: new crate, workspace Cargo.toml `members` list, commands.rs new subcommand registration (bottom-append only), new tests
- **OUT**: route.rs, coordinator.rs, doctor.rs, CONTRACT.md, adapter code, calibration, existing subcommands in commands.rs, aiplus-cli Cargo.toml version, CHANGELOG actual
- **GRAY**: small README touchups in the new crate are fine

## Deliverables (5)

1. `docs/proposals/token-cost-impl-notes.md`
2. Complete new crate `crates/aiplus-token-cost/` (lib + 5 modules + tests + README)
3. Workspace `Cargo.toml` updated
4. `commands.rs` new subcommand variant + handler appended (NOT modifying existing parts)
5. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+5d**.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + which sub-features deviated + sample CLI output as evidence. Advisor coordinates Phase C with Wave 2-B and Wave 2-D into single v0.6.5 ship.

## Owner gates

Standard.

If using BWS for any LLM-call test during dev: wrap. (This goal mostly doesn't call LLMs; just walks log files + computes USD.)

## Dependencies

- Wave 1 (G-AT-AUDITOR-REMOVE-1) merged to main
- LiteLLM JSON URL reachable for first-fetch; otherwise embedded fallback works
- Existing dispatch-log.jsonl format

---

— Advisor, 2026-05-19, Wave 2-C. Day-0 narrow STOP + retry-once.
