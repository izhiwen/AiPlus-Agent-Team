# Goal G-AT-AGENT-AUTOFLOW-MCP-1 — Register 3 new MCP tools for agent discoverability

**Status**: PROPOSED — Owner-authorized 2026-05-19 (chose Option A from agent-autoflow analysis: MCP-only, defer SKILL.md updates + install prompts).
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AGENT-AUTOFLOW-MCP-1`

---

## TL;DR

Three features shipped in v0.6.5 + v0.6.6 (`aiplus agent token-cost`, `aiplus agent audit verify-log`, `aiplus agent route --score-only`) are CLI-only — the user's agent (Claude Code / Codex / OpenCode) doesn't know they exist because they aren't exposed as MCP tools. **This goal registers them as MCP tools** so the agent can discover + invoke them via existing `aiplus mcp-register` plumbing, without any explicit user prompt.

Option A only. SKILL.md docs (Option B) and `aiplus install` opt-in prompts (Option C) deferred until dogfood signal informs whether they're worth the additional work.

## Success criterion (single sentence)

> After this goal, an agent in Claude Code / Codex / OpenCode that has `aiplus mcp-register` configured can list `agent_token_cost`, `agent_audit_verify_log`, and `agent_route_score_only` in its tool inventory, and successfully call each one and get structured JSON back.

## Goals (D1-D4)

- **D1**: `mcp_server.rs` registers `agent_token_cost` tool with schema for `window` (optional `"1h"|"8h"|"24h"`), `by_role` (optional bool), `top_n` (optional int, default 5). Returns structured JSON mirroring `aiplus agent token-cost` output.
- **D2**: `mcp_server.rs` registers `agent_audit_verify_log` tool with no required parameters. Returns `{"verdict": "PASS" | "FAIL", "first_bad_line": <int|null>, "reason": <string|null>}` mirroring `aiplus agent audit verify-log` output.
- **D3**: `mcp_server.rs` registers `agent_route_score_only` tool with required `task` parameter. Returns structured JSON with `complexity`, `risk`, `tier`, `code_change`, `design_impact`, `consultant`, `staffing_roles`, `forced_by_risk`, `auto_summoned`. Mirrors `aiplus agent route --score-only "<task>"` output.
- **D4**: Tests cover all 3 new tools: (a) tool listed in tool inventory, (b) tool invokable end-to-end, (c) malformed args produce clear MCP error.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/agent-autoflow-mcp-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/agent-autoflow-mcp-impl-notes.md` (Phase 1 design + Phase 3 evidence) | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/mcp_server.rs` extension (3 new tools + 3 new dispatch functions) | PENDING — CEO |
| 5 | New tests covering tool list inclusion + happy-path invoke + error path | PENDING — CEO |
| 6 | CHANGELOG `## Unreleased` entry (later promoted to ## 0.6.7 by Advisor at Phase C) | PENDING — CEO drafts in impl-notes |

## Phases

```
Phase 1 (CEO codex, ~0.5 day):
  - Read mcp_server.rs to confirm tool-registration pattern (matches the 11
    existing tools: agent_route / agent_status / etc.)
  - Design 3 new tool JSON schemas (input + output)
  - Design 3 new call_agent_<x> dispatch functions
  - Decide whether to subprocess `aiplus` self (clean but slow) or call
    crate functions directly (faster, requires more careful wiring)

Phase 2 (CEO codex, ~0.5-1 day):
  - Implement 3 tool definitions in mcp_server.rs
  - Implement 3 dispatch functions
  - Add tests

Phase 3 (CEO codex, ~0.5 day):
  - All 569 existing tests still PASS + new tests
  - cargo clippy clean
  - Live MCP verification: connect via MCP client, list tools, invoke each
  - Evidence trailer in impl-notes
```

**Total target**: ≤ 2 days. Likely ~1 day per pattern.

## Out of scope

- **SKILL.md updates** (Option B) — deferred until dogfood signal shows agent isn't using these tools without being told
- **`aiplus install` opt-in prompts** (Option C) — deferred for same reason
- **`mcp_server.rs` refactoring** — only ADD 3 tools, don't restructure existing 11
- **CONTRACT.md edits** — FROZEN
- **Adapter code** — frozen
- **`crates/aiplus-token-cost/`** — subtree mirror, hands-off
- **`coordinator.rs` scoring rubric** — calibration-locked
- **Calibration fixture existing entries** — locked

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Dispatch function deadlock under concurrent MCP calls | Each call subprocess-spawns `aiplus`; processes are isolated; no shared mutable state |
| `agent_token_cost` slow when LiteLLM JSON cache miss | Acceptable — first call of the day is slower, subsequent fast; document |
| Schema mismatch between MCP tool output and CLI output | Phase 1 design captures exact serialization; tests verify parity |
| MCP tool args parsing differs from CLI arg parsing | Use same arg parser; subprocess approach inherits CLI parsing automatically |
| New tools break existing MCP client (older Claude Code with cached tool list) | MCP clients re-fetch tool list per session; backward-compatible by add-only design |

## Acceptance criteria

- ✓ D1-D4 met
- ✓ `cargo test --workspace` PASS (569+ tests)
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` PASS
- ✓ Live: `aiplus mcp-serve` followed by MCP client `tools/list` shows all 3 new tools
- ✓ Live: MCP `tools/call agent_token_cost` returns valid JSON
- ✓ Live: MCP `tools/call agent_route_score_only` for a sample task returns valid JSON
- ✓ Live: MCP `tools/call agent_audit_verify_log` returns valid JSON
- ✓ Existing 11 MCP tools still work
- ✓ CHANGELOG draft text in impl-notes (Advisor commits actual CHANGELOG at Phase C)

## What happens when goal completes

- `aiplus-cli` bumps to v0.6.7
- After tag + release, any user running `aiplus mcp-register` picks up the 3 new tools
- Their agent (Claude/Codex/OpenCode) sees the tools in inventory and can invoke them per its own prompt-following heuristics
- Owner observes during continued dogfood whether the agent actually uses them or just ignores them; that signal drives whether B (SKILL.md) and C (install prompts) are worth doing

## Dependencies

- v0.6.6 shipped to main ✓ (`aiplus-public@da5dd95`)
- v0.6.5 features (`agent token-cost`, `agent audit verify-log`, `agent route --score-only`) all available as CLI subcommands ✓
- Existing MCP server infrastructure (`mcp_server.rs` 594 lines) ✓
- Existing `aiplus mcp-register` subcommand for runtime adapter wiring ✓

## Next action

Advisor dispatches CEO codex with the companion briefing.

---

— Advisor, 2026-05-19, Option A of agent-autoflow analysis (smallest scope: just MCP registration).
