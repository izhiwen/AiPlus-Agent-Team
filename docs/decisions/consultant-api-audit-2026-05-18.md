# Consultant API Audit — 2026-05-18

**Auditor**: Advisor
**Purpose**: Determine whether `aiplus-auto-team-consultant` Coordinator scoring API is stable enough for v0.3 to depend on (per DESIGN.md §9.1 "reuse" principle).
**Scope**: half-day read-only audit of `~/Projects/AiPlus/aiplus-auto-team-consultant/` v0.4.6.

## Verdict

**API NOT STABLE for direct reuse**. v0.3 must internally re-implement scoring per DESIGN.md §9.1 spec.

## Findings

1. **Scale mismatch** — DESIGN.md §9.1 says `complexity 1-5 + risk 0.0-1.0`. consultant module's `core/docs/consultant-team-decision-system.md` line 418 says `complexity_score=0-3`. `core/docs/protocol.md` line 9 mentions `L0-L5 routing levels`. **Three different scales** in the dependency surface; none matches DESIGN.md §9.1.

2. **No code-level API** — consultant module is template + persona driven (TOML configs + markdown personas + slash commands). There is no Rust function `score_task(input) -> (complexity, risk)` to call. "Reuse" would mean parsing consultant docs and implementing the rubric.

3. **Schema versions present but inconsistent** — `aiplus-module.json schemaVersion=0.1.0`; `consultant-team.default.toml schema_version="0.1"`; README example shows `schema_version="2.1"`. No single authoritative versioning across the dependency surface.

4. **Module version** = v0.4.6. Pre-1.0 → no compatibility guarantees yet.

## Recommendation for v0.3

**Option A (chosen)**: For v0.3.0, **implement scoring inside `aiplus-cli` per DESIGN.md §9.1 spec** (1-5 complexity + 0.0-1.0 risk). Treat consultant module as a downstream consumer that should eventually align to DESIGN.md §9.1, not as upstream source of truth.

- Pragmatic: unblocks v0.3 launch
- Slight DESIGN.md §4 (DRY) violation: documented as deviation justified by current consultant API instability
- Future cleanup: when consultant module reaches v1.0 with stable scoring API, v0.3 can refactor to delegate

**Option B (rejected)**: Block v0.3 until consultant module ships stable scoring API. Rejected because: (a) consultant is pre-1.0 with no committed timeline, (b) implementing scoring is small (~200 LOC) and per DESIGN.md §9.1 is fully specified.

## Implication for v0.3 briefing

The implementation briefing for CEO codex MUST instruct: "Implement scoring per DESIGN.md §9.1 inside `crates/aiplus-cli/src/agent/coordinator.rs`. Do NOT import from `aiplus-auto-team-consultant`. Document this deviation from DESIGN.md §4 DRY principle in IMPLEMENTATION notes."

## Follow-up

When consultant module reaches v1.0 (timeline TBD by consultant team), Advisor reviews whether v0.3 should be refactored to delegate. Until then: aiplus-cli is single source of truth for Coordinator scoring.
