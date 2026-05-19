# Goal G-AT-AUTOFLOW-MULTITURN-1 — Dispatch flow + multi-turn patterns in SKILL.md

**Status**: PROPOSED — Owner-authorized 2026-05-19 as part of 3-goal program ("3 个新 goal 全做"). Wave 1 parallel with G-AT-AUTOFLOW-COVERAGE-1.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AUTOFLOW-MULTITURN-1`

---

## TL;DR

v0.6.9 discovery layer routes natural single-turn queries to the right MCP tool. But the full agent-team dispatch flow (user describes task → agent previews via `agent_route_score_only` → user confirms → agent dispatches via `agent_route` → integrates via `agent_integrate`) hasn't been explicitly demonstrated in SKILL.md. Multi-turn conversations where agent should remember previous tool calls also aren't covered.

This goal adds dispatch-flow examples and multi-turn patterns to SKILL.md + project-root preamble. After ship, "fix this bug in user auth" should produce a 3-4 turn conversation: scoring → confirm → dispatch → integrate, with agent guiding the user.

Target: ≥ 2 of 3 simulated multi-turn flows produce correct dispatch sequence in Wave 2 validation.

## Success criterion (single sentence)

> After this goal ships, "Help me refactor the user auth module" produces an agent response that: (a) calls `agent_route_score_only` first; (b) surfaces the would-staffing plan to user; (c) asks for confirmation; (d) on user confirm, calls `agent_route` with the right role; (e) later when work is done, suggests `agent_integrate`.

## Goals (D1-D3)

- **D1**: SKILL.md × 3 runtimes add new "Dispatch Flow" section showing the 3-4 turn pattern explicitly.
- **D2**: SKILL.md add new "Multi-turn Patterns" section with examples of common multi-turn conversations (followup questions, mid-flight changes, dispatch confirmations).
- **D3**: Project-root preamble managed-block appends a "Dispatch Flow" sub-section after the intent list (added by parallel Session A).

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/autoflow-multiturn-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/autoflow-multiturn-1-impl-notes.md` | PENDING — CEO Session B |
| 4 | SKILL.md × 3 new "Dispatch Flow" section | PENDING |
| 5 | SKILL.md × 3 new "Multi-turn Patterns" section | PENDING |
| 6 | Preamble managed-block dispatch flow append | PENDING |
| 7 | Tests for content shape | PENDING |

## Out of scope

- 11 other MCP tools / non-MCP feature coverage → G-AT-AUTOFLOW-COVERAGE-1 (parallel Session A)
- Multi-runtime LLM validation → G-AT-AUTOFLOW-VALIDATE-1 (Wave 2)
- All standard forbidden files (CONTRACT, adapter, scoring, etc.)

## Acceptance criteria

- ✓ D1-D3 met
- ✓ `cargo test --workspace` PASS
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ SKILL.md content tests verify "Dispatch Flow" + "Multi-turn Patterns" sections exist

## Dependencies

- v0.6.9 main ✓
- Parallel-safe with Coverage-1 via explicit ownership

---

— Advisor, 2026-05-19. Wave 1 parallel goal B.
