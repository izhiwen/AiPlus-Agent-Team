# Goal G-AT-AGENT-AUTOFLOW-MCP-1 — Retrospective (COMPLETE)

**Status**: **DONE** — 3 new MCP tools registered, shipped as part of v0.6.7 on 2026-05-19.
**Original goal**: `docs/proposals/goal-G-AT-AGENT-AUTOFLOW-MCP-1.md`
**Closing date**: 2026-05-19 (opened same day)
**Duration**: ~half work day (vs. ≤2 day budget). Under budget by ~75%.

---

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | Goal doc + CEO briefing | `e6bad4c` | aiplus-agent-team |
| 2 | `mcp_server.rs` 3 new tools + `agent_autoflow_mcp.rs` live integration test + impl-notes | `b5887f7` | aiplus-public |
| 3 | Phase C: 0.6.6 → 0.6.7 + CHANGELOG + install.sh parity + retrospective | post-`b5887f7` (this commit) | aiplus-public + aiplus-agent-team |

**Total**: 912 lines impl across 3 files + 313 lines spec/retrospective.

## D-goal status

- D1 (`agent_token_cost`): ✓ — JSON schema with optional window/by_role/top_n; subprocess dispatch; structured JSON return
- D2 (`agent_audit_verify_log`): ✓ — no-args schema; PASS/FAIL with line-number on FAIL
- D3 (`agent_route_score_only`): ✓ — required `task` arg; returns complexity/risk/tier/staffing_roles/forced_by_risk/auto_summoned
- D4 (tests): ✓ — 3 happy-path calls + invalid-args error case; live MCP integration test starts `aiplus mcp-serve`, sends real JSON-RPC

## What went well

1. **First pure PASS verdict in the multi-day sprint**, not PASS_WITH_DEVIATIONS. CEO returned no scope deviations to disclose — all changes within OWN list, all forbidden files untouched, all sub-features as specified. The briefing template is now stable enough that CEO produces clean PASSes by default.
2. **Subprocess pattern reuse paid off.** CEO followed the existing 11-tool pattern (each subprocess-calls `aiplus <verb>`) without inventing a new mechanism. ~536 LoC added to mcp_server.rs but conceptually it's just "3 more instances of the same pattern". Low cognitive load, low review burden.
3. **Live MCP integration test in this goal sets a precedent.** Future MCP-related work has a working test harness to extend (`tests/agent_autoflow_mcp.rs` starts the server, JSON-RPC roundtrips, asserts). Worth more than the 126 LoC it took to write.
4. **Goal scope discipline.** Option A only (MCP registration); resisted creeping into Option B (SKILL.md) or Option C (install prompts). Sets up clean follow-up decisions post-dogfood.

## What went wrong

1. **Nothing this goal.** Second goal in a row with no "what went wrong" entries (after G-AT-COORDINATOR-PARALLEL-1). The accumulated discipline — briefing template + day-0 narrow STOP + retry-once gate + worktree isolation + clean ownership matrix — is producing consistently clean output.

## Architectural lessons codified

1. **Subprocess MCP tool pattern is reusable for any CLI subcommand.** Future v0.6.x features that ship as `aiplus agent <verb>` can become MCP-discoverable via the same pattern: 1 JSON schema + 1 subprocess dispatcher + 1 test. ~30-50 LoC per new tool.
2. **MCP server live integration test pattern.** `tests/agent_autoflow_mcp.rs` is the template for future MCP work — start the server as subprocess, JSON-RPC via stdin/stdout, assert on response. Avoids mock complexity.
3. **Estimate-vs-actual continues to favor actual side.** Briefing said T+2d hard ceiling, "likely ~1 day". Actual: ~half day. Future single-feature MCP goals can budget ~half day per tool.

## Open follow-ups (not part of this goal)

- **Option B (SKILL.md guidance)** — defer until dogfood shows agent isn't using the new tools. If agent ignores them per its own heuristics, write explicit "when to use" guidance.
- **Option C (`aiplus install` opt-in prompts)** — same trigger.
- **Token-cost MCP tool returns text-content JSON, not native object.** Current implementation matches the 11-tool pattern (stable JSON inside MCP text content). If the agent has trouble parsing text-content, future revision could switch to native MCP object content. Defer until evidence demands.
- **Per-tool rate limiting** — if agent calls `agent_token_cost` on every dispatch, that adds LiteLLM fetch overhead. Probably not an issue (24h cache) but watch during dogfood.

## Next goal

No new goal pre-committed. The Owner's stated direction was Option A only; Options B + C are dogfood-driven.

**This goal closes the post-program follow-up sprint.** Status of the broader work:

- 4-goal program (2026-05-18): ✓ DONE
- Wave 1+2 sprint (2026-05-18 → 19): ✓ DONE
- G-AT-TOKEN-COST-STANDALONE-1 (2026-05-19): ✓ DONE
- Platform narrowing (2026-05-19): ✓ DONE  
- G-AT-AGENT-AUTOFLOW-MCP-1 (this goal, 2026-05-19): ✓ DONE

aiplus-cli is at v0.6.7. **Total 10 goals shipped + 1 platform decision + 1 sibling repo created in ~2 work days.** Original estimate would have been many weeks.

Advisor's standing recommendation: enter real dogfood. Now even more emphatic — the toolchain has accumulated significant surface area without any single-day of real Owner usage. Next goal should be dogfood-driven, not Advisor-anticipated.

---

— Advisor, 2026-05-19, G-AT-AGENT-AUTOFLOW-MCP-1 closed (Option A only).
