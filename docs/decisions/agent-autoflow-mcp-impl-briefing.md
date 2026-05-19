# Agent-Autoflow MCP Registration — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AGENT-AUTOFLOW-MCP-1.md`
- Reference: existing MCP tools in `aiplus-public/crates/aiplus-cli/src/mcp_server.rs` (594 lines, 11 existing tools modeled on `agent_route` / `agent_status` etc.)
- Existing CLI surface to wrap:
  - `aiplus agent token-cost [--window <1h|8h|24h>] [--by-role] [--top-n <N>]`
  - `aiplus agent audit verify-log`
  - `aiplus agent route --score-only "<task>"`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.agent-autoflow-mcp/`
- Branch:    `feat/agent-autoflow-mcp`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.agent-autoflow-mcp -b feat/agent-autoflow-mcp`

Shared-file ownership:
- This session OWNS: `crates/aiplus-cli/src/mcp_server.rs`, new tests in `crates/aiplus-cli/tests/`, optionally small touch to `crates/aiplus-cli/src/main.rs` if a thin wrapper is needed
- This session must NOT modify: CONTRACT.md (FROZEN), `crates/aiplus-token-cost/` (subtree mirror), `coordinator.rs` scoring rubric (calibration-locked), `tests/fixtures/coordinator_calibration.toml` existing entries, adapter code, `Cargo.toml` version field, `CHANGELOG.md` actual, `install.sh`
- If `mcp_server.rs` needs structural refactor for new tools → STOP and ping Advisor (briefing assumes additive)

## Scope — add 3 MCP tools

Pattern: each new tool gets (a) a JSON tool definition in the `tools/list` response, (b) a dispatch function in `tools/call` matching, (c) tests. Model after the 11 existing tools (`agent_route` / `agent_status` / etc.) — minimal deviation from established style.

### Tool 1: `agent_token_cost`

```json
{
  "name": "agent_token_cost",
  "description": "Show token consumption and USD cost rollups for AiPlus dispatch logs in 1-hour / 8-hour / 24-hour windows, with optional per-role breakdown and top-N most expensive tasks. Useful for the agent to check current burn before authorizing more dispatches.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "window": {
        "type": "string",
        "enum": ["1h", "8h", "24h"],
        "description": "Single window to restrict to. Omit to get all three."
      },
      "by_role": {
        "type": "boolean",
        "description": "Include per-role breakdown. Default false."
      },
      "top_n": {
        "type": "integer",
        "minimum": 1,
        "maximum": 50,
        "description": "Top-N most expensive tasks per window. Default 5."
      }
    },
    "required": []
  }
}
```

Dispatch: shell out to `aiplus agent token-cost [args]`. Parse the structured output. Return as JSON.

### Tool 2: `agent_audit_verify_log`

```json
{
  "name": "agent_audit_verify_log",
  "description": "Verify the integrity of .aiplus/agents/dispatch-log.jsonl hash chain. Reports PASS or FAIL with line number of first bad row. Useful for the agent to confirm no log tampering before relying on dispatch history.",
  "inputSchema": {
    "type": "object",
    "properties": {},
    "required": []
  }
}
```

Dispatch: shell out to `aiplus agent audit verify-log`. Parse `VERIFY_LOG=PASS` or `VERIFY_LOG=FAIL line=N reason=...` output. Return:

```json
{"verdict": "PASS"}
// or
{"verdict": "FAIL", "first_bad_line": 42, "reason": "hash mismatch"}
```

### Tool 3: `agent_route_score_only`

```json
{
  "name": "agent_route_score_only",
  "description": "Pre-flight a task: run the adaptive coordinator's scorer and tier classifier on the task text, but do NOT actually dispatch any role. Returns the complexity/risk score, tier, would-staff list, and any auto-summoned experts. Useful for the agent to estimate cost/team-size before committing.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "task": {
        "type": "string",
        "description": "Task description in natural language."
      }
    },
    "required": ["task"]
  }
}
```

Dispatch: shell out to `aiplus agent route --score-only "<task>"`. Parse the structured output line `Adaptive coordinator: complexity=N risk=F tier=T ...`. Return as JSON.

## Implementation approach

### Subprocess vs in-process

**Recommended: subprocess** (`std::process::Command::new("aiplus").arg("agent").arg("token-cost")...`).

Why:
- Mirrors the existing 11 tools' approach (they all subprocess `aiplus agent <verb>`)
- Decouples MCP server from CLI internals — refactoring CLI doesn't break MCP
- Same arg parsing as CLI; no schema duplication
- Easier failure isolation

In-process (call `aiplus_token_cost::run_token_cost(...)` directly) would be faster but couples MCP to crate internals. Pattern says subprocess.

### Phase 1 — Write `docs/proposals/agent-autoflow-mcp-impl-notes.md` BEFORE coding

Required sections:
1. Confirmation of subprocess approach (or, if you decide otherwise, document why)
2. Exact JSON tool definitions for 3 tools (final form; iterate on briefing's starting template)
3. Output parsing strategy for each (regex / `serde_json` / line-by-line)
4. Test plan
5. CHANGELOG draft text

### Phase 2 — Implement

Order:
1. `agent_audit_verify_log` first (simplest — no args, simple output)
2. `agent_route_score_only` second (one required arg)
3. `agent_token_cost` last (most args)

After each: `cargo build` clean.

### Phase 3 — Evidence

- `cargo test --workspace` PASS (569 + new tests = ~575)
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Live MCP smoke test:
  - Start `aiplus mcp-serve` in background
  - Use stdin/stdout JSON-RPC to send `tools/list` — verify 3 new tools listed
  - Send `tools/call` for each of the 3 new tools with valid args — verify response
  - Send `tools/call` with invalid args — verify clean error
- Document each smoke test result in evidence trailer

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** (existing test fails) or `_SCOPE-BREACH_` (touched a not-owned file)
- **Do NOT STOP on `_IMPL-BUG_`** — fix and continue
- **Retry-once gate** for any first-attempt FAIL before classification

## Conformance / verification

- All 4 D-goals met
- All tests green
- All 3 new MCP tools discoverable in tool list
- All 3 new MCP tools invokable end-to-end
- Existing 11 MCP tools unaffected
- No new clippy warnings

## Scope fence

- **IN**: `mcp_server.rs` (3 new tool definitions + 3 new dispatch functions), new tests, optional small touch to `main.rs` for thin wrapper if needed
- **OUT**: any other file in aiplus-public (especially: Cargo.toml version, CHANGELOG.md actual, install.sh, calibration fixture, coordinator scoring, adapter code), CONTRACT.md, `crates/aiplus-token-cost/`
- **GRAY**: small comment improvements in `mcp_server.rs` adjacent to your edits

## Deliverables (5)

1. `docs/proposals/agent-autoflow-mcp-impl-notes.md` — Phase 1 design + Phase 3 evidence
2. `crates/aiplus-cli/src/mcp_server.rs` — 3 new tools + 3 dispatchers
3. New tests covering tool list + happy-path invocation + error path
4. CHANGELOG draft text in impl-notes
5. (Optional) brief MCP smoke-test script if the existing test infrastructure makes this clean

## Time ceiling

**Hard ceiling: T+2d**. Likely ~1 day actual based on pattern.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + which sub-features deviated + live MCP smoke evidence. Advisor handles Phase C (version bump, CHANGELOG, install.sh parity, retrospective).

## Owner gates

Standard. BWS-backed: wrap with `aiplus secret-broker run --aliases ...` if any live tests need API keys (probably not needed — tests are MCP plumbing not LLM calls).

## Dependencies

- v0.6.6 main ✓ (`aiplus-public@da5dd95`)
- 3 CLI subcommands already work ✓
- `mcp_server.rs` infrastructure ✓

---

— Advisor, 2026-05-19, Option A of agent-autoflow. Day-0 narrow STOP + retry-once.
