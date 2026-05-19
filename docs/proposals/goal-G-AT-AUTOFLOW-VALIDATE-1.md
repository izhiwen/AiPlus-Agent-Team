# Goal G-AT-AUTOFLOW-VALIDATE-1 — Multi-runtime full-coverage discovery validation

**Status**: PROPOSED — Owner-authorized 2026-05-19 as part of 3-goal program ("3 个新 goal 全做"). Wave 2 serial after Wave 1 (Coverage + Multi-turn) ships.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AUTOFLOW-VALIDATE-1`

---

## TL;DR

After Wave 1 ships (Coverage + Multi-turn), the discovery layer should cover the full aiplus feature surface. This goal **systematically validates** the coverage with a programmatic multi-runtime test harness + 1 week of Owner dogfood.

End state: empirical coverage matrix — for each of 14 MCP tools + 6 non-MCP feature categories × 3 runtimes (codex / claude-code / opencode), known whether the agent auto-discovers from natural prompts.

## Success criterion (single sentence)

> After this goal closes, a coverage matrix exists showing which aiplus features are reliably agent-auto-discoverable per runtime, and a dogfood note from Owner identifying which natural-language → aiplus paths felt seamless vs friction-y in 1 week of real usage.

## Goals (D1-D4)

- **D1**: Programmatic multi-runtime test harness in `crates/aiplus-cli/tests/autoflow_coverage_matrix.rs` or similar. Each test = one prompt × one runtime, asserts which MCP tool / CLI was called (or bypass observed).
- **D2**: 20-30 test cases covering: each MCP tool (one prompt per) + each non-MCP feature category (one prompt per) × at least 2 runtimes (codex + opencode reliably testable non-interactively).
- **D3**: Coverage matrix output: a markdown table in `docs/proposals/autoflow-validate-1-coverage-matrix.md` showing PASS / FAIL / RUNTIME-LIMITATION / NOT-TESTED per cell.
- **D4**: Owner dogfood week — Owner uses aiplus naturally for ~1 week, records friction points (natural-language paths that felt awkward, features they wanted but couldn't access via agent, etc.) in a notes file Advisor collates.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/autoflow-validate-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/autoflow-validate-1-impl-notes.md` | PENDING — CEO Session C |
| 4 | `tests/autoflow_coverage_matrix.rs` (or similar) test harness | PENDING |
| 5 | Coverage matrix doc | PENDING |
| 6 | Owner dogfood notes collation | PENDING — Advisor Phase C |

## Out of scope

- New feature work (this is pure validation; no production-code changes beyond test harness)
- Standard forbidden files

## Acceptance criteria

- ✓ D1-D4 met
- ✓ Coverage matrix has ≥ 20 cells filled
- ✓ Owner has used aiplus across ≥ 5 distinct sessions over 1 week
- ✓ Friction notes have at least 3 entries (positive or negative — both useful)

## Dependencies

- Wave 1 ratified (Coverage-1 + Multi-turn-1 both shipped)
- Owner availability for 1 week dogfood

## Timing

- CEO test harness work: ~1-2 days after Wave 1 ratifies
- Owner dogfood: 1 calendar week (no active CEO/Advisor work; Owner just uses aiplus)
- Advisor Phase C collation: ~half day post-dogfood

**Total wall-clock**: ~1.5 weeks (mostly Owner dogfood time)

---

— Advisor, 2026-05-19. Wave 2 sequential goal.
