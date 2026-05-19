# Goal G-AT-AUTOFLOW-COVERAGE-1 — Extend discovery layer to remaining 11 MCP tools + non-MCP features

**Status**: PROPOSED — Owner-authorized 2026-05-19 as part of 3-goal program ("3 个新 goal 全做"). Wave 1 parallel with G-AT-AUTOFLOW-MULTITURN-1.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AUTOFLOW-COVERAGE-1`

---

## TL;DR

v0.6.9 discovery layer covers 3 of 14 MCP tools (`agent_token_cost`, `agent_audit_verify_log`, `agent_route_score_only`) and 0 non-MCP aiplus features. Per Advisor's "50-70% implemented" honest assessment, this goal extends discovery to:
- The **other 11 MCP tools** (`agent_route`, `agent_status`, `agent_set_team`, `agent_list`, `agent_doctor`, `agent_invite`, `agent_dismiss`, `agent_disable`, `agent_enable`, `agent_integrate`, `agent_talk`)
- **Non-MCP aiplus features** that lack MCP wrappers: memory (`aiplus memory ...`), compact-reminder (`aiplus compact ...`), velocity (`aiplus velocity ...`), identity signing (`aiplus identity setup-signing`), doctor flags (`aiplus doctor --quiet` / `--check-keyring`), team switching (`aiplus agent set-team`)

For non-MCP features, the agent calls them via shell — but with explicit SKILL.md guidance routing the agent to the right CLI subcommand instead of bypass-via-internal-knowledge.

Target: ≥ 8 of 11 + 5 of 6 non-MCP feature categories empirically discoverable in Phase C re-test.

## Success criterion (single sentence)

> After this goal ships, a user asking natural-language questions across the full aiplus feature surface (not just cost/planning/audit) gets the agent routing to the right aiplus tool (MCP or CLI) — e.g., "remind me about our naming convention" triggers `aiplus memory context`, "what's my role velocity been" triggers `aiplus velocity report`, "switch team to AEL" triggers `agent_set_team`, etc.

## Goals (D1-D5)

- **D1**: 11 existing MCP tools get "PREFERRED programmatic surface" prefix + intent-line + CLI-alternative reference in their `description` field (`crates/aiplus-cli/src/mcp_server.rs`).
- **D2**: SKILL.md × 3 runtimes "Use These Tools First" section expanded to include all 14 MCP tools + the 6 non-MCP feature categories with intent → tool/command mapping.
- **D3**: Project-root preamble (`CLAUDE.md` / `AGENTS.md` / `.opencode/instructions/aiplus.md`) intent list extended to cover the full surface.
- **D4**: Phase C re-test: ≥ 8 of 11 other MCP tools + ≥ 5 of 6 non-MCP feature categories empirically discoverable when prompted naturally. Acceptance gate per re-test.
- **D5**: Existing 3-tool discovery (v0.6.9) remains intact — additive only, no regressions.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/autoflow-coverage-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/autoflow-coverage-1-impl-notes.md` (Phase 1 + 3) | PENDING — CEO Session A |
| 4 | `mcp_server.rs` description enhancement for 11 existing tools | PENDING |
| 5 | SKILL.md × 3 "Use These Tools First" section extension | PENDING |
| 6 | Project-root preamble intent list extension | PENDING |
| 7 | Tests for content shape + idempotency | PENDING |

## Out of scope

- Multi-turn dialogue patterns / dispatch flow examples → that's G-AT-AUTOFLOW-MULTITURN-1 (Session B parallel)
- Validation harness / multi-runtime full coverage test → G-AT-AUTOFLOW-VALIDATE-1 (Wave 2 serial)
- CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh fallback

## Acceptance criteria

- ✓ D1-D5 met
- ✓ `cargo test --workspace` PASS
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase C re-test (Advisor will run): ≥ 8/11 other MCP + ≥ 5/6 non-MCP feature discoverable
- ✓ Existing 3-tool discovery (v0.6.9) still works (regression test)

## Dependencies

- v0.6.9 main ✓ (post-discovery v1+v2 ship)
- 14 MCP tools registered ✓
- Parallel-safe with G-AT-AUTOFLOW-MULTITURN-1 via explicit ownership matrix in briefing

---

— Advisor, 2026-05-19. Wave 1 parallel goal A.
