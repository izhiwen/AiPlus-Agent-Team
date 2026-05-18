# Goal G-AT-PROD-1 — Agent-Team Production-Ready in ~4 weeks

**Status**: PROPOSED — Owner sign-off pending.
**Drafted by**: Advisor (post-CONTRACT v1 FROZEN, 2026-05-18)
**Goal ID**: `G-AT-PROD-1` (Goal, Agent-Team, Production-readiness, milestone 1)

---

## TL;DR

CONTRACT v1 is FROZEN. The adapter layer (3 runtimes) is production-ready. **G-AT-PROD-1 closes v0.2's remaining items + ships v0.3.0 P0 + dogfoods the result end-to-end**. End state: agent-team has 3 conformance-validated adapters, persistent warm bench across CLI invocations, and a CEO that autonomously scores + staffs tasks per DESIGN.md §9. ~4 weeks of work.

## Success criterion (single sentence)

> Owner says "实现支付接口" → CEO scores 4-5 complexity / 0.7-0.9 risk → fires consultant team → staffs full HEAVY team across 3 runtime worktrees → CEO integrates + reports back without Owner intervention beyond the initial dispatch. **And the second time Owner re-dispatches the same role within 30 minutes, the agent resumes instantly from disk-warm cache.**

That sentence captures D5 (v0.3 acceptance) + disk-warm cache + multi-adapter validation in one observable workflow.

## Goals (D1-D6)

- **D1**: v0.2 closure — disk-warm cache (DESIGN.md §6.3 Option B) shipped + Codex round 3 conformance PASS (zero `_CONTRACT-BUG_`).
- **D2**: Consultant API audit complete (`docs/decisions/consultant-api-audit-2026-05-18.md`) → v0.3 implements scoring internally per DESIGN.md §9.1 (audit recommendation Option A).
- **D3**: v0.3.0 P0 shipped — CEO scorer + tier classifier + staffing dispatcher + two-step flow.
- **D4**: D5 acceptance scenario passes end-to-end without Owner intervention.
- **D5**: All 3 adapters production-validated (Claude Code ✓ round 1, OpenCode ✓ round 2, Codex pending round 3).
- **D6**: No regression — existing v0.1/v0.2 flows still work; CONTRACT v1 remains FROZEN throughout.

## Sub-deliverables (this goal contains)

| # | Artifact | Status |
|---|---|---|
| 1 | `docs/proposals/v0.2-disk-cache.md` (spec) | DRAFT (this commit) |
| 2 | `docs/decisions/codex-adapter-impl-briefing.md` (round 3) | DONE (commit `a16853b`) |
| 3 | `docs/decisions/consultant-api-audit-2026-05-18.md` (audit memo) | DONE (this commit) |
| 4 | `docs/decisions/v0.3-coordinator-impl-briefing.md` (v0.3.0 P0 briefing) | DONE (this commit) |
| 5 | v0.2 disk-cache code (~1.5 weeks) | PENDING — CEO impl |
| 6 | Codex adapter (~1-2 days) | PENDING — CEO impl |
| 7 | v0.3.0 P0 code (~1 week) | PENDING — CEO impl |
| 8 | D5 acceptance scenario evidence | PENDING — produced during impl |

## Phases + sequencing

```
Week 1: Phase A + B (parallel)
  Phase A — disk-cache impl (CEO codex, per spec #1)
  Phase B — Codex round 3 adapter (CEO codex, per briefing #2)

Week 2: Phase C
  Phase C — v0.3.0 P0 impl (CEO codex, per briefing #4 + audit #3)

Week 3: Phase D
  Phase D — D5 acceptance dogfood + integration test

Week 4 (buffer): Phase E
  Phase E — fix discovered issues + final polish + ship docs
```

Parallel Phase A + B is OK because they touch different code paths (cache primitives vs adapter implementation). Phase C depends on consultant audit ✓ but not on A/B; can technically start in parallel but recommend serial for clean dogfood evidence.

## Out of scope (this goal explicitly does NOT include)

- v0.3 P1 (expert auto-summoning, risk-based forced summoning, TTL honoring) → v0.3.1
- v0.3 P2 (audit trail, manual override, disagreement signal) → v0.3.2
- v0.4 cross-runtime parallelism
- DESIGN.md §15.1 hardware-backed signing
- DESIGN.md §22 cross-provider auditor (Layer 7)
- Web UI for team status
- Multi-project team sharing
- Consultant module v1.0 alignment (v0.3 implements scoring internally; consultant alignment is a future cleanup)

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Codex round 3 surfaces `_CONTRACT-BUG_` | LOW | HIGH (unfreezes v1, delays goal 2-3 weeks) | Codex is cleanest of 3 runtimes; explicit STOP protocol catches early |
| Disk-cache redaction misses something → Owner finds secret in `.cbor` | LOW | HIGH (privacy breach) | AC4 acceptance + redaction reuse from existing `aiplus-agent-memory` 12 patterns + opt-in only |
| v0.3 scoring rubric mis-calibrated → over-staffs LIGHT tasks | MEDIUM | LOW (wasted tokens, observable, easy to tune) | Calibration test (20 historical tasks) + Owner spot-check |
| Consultant module ships v1.0 mid-goal, forces v0.3 refactor | LOW | MEDIUM | Implementation notes document deviation; refactor is future cleanup not goal-blocker |
| Owner's dogfood reveals D5 scenario doesn't match real workflow | MEDIUM | MEDIUM | Phase D + E include real-task validation, not only the canonical example |

## Acceptance criteria (for goal completion)

- ✓ D1-D6 all met
- ✓ 3 adapter conformance PASS (Claude Code, OpenCode, Codex — all spec #01-07)
- ✓ Disk-warm cache visible in `aiplus agent status --verbose` (D2 from disk-cache spec AC1)
- ✓ D5 acceptance scenario reproducible by Owner without Advisor / CEO mediation
- ✓ CHANGELOG entry for the agent-team v0.2 + v0.3.0 ship
- ✓ All work fits in ≤ 5 weeks (4 weeks target + 1 week buffer)

## What happens when goal completes

- `agent-team` module bumps to v0.3.0 (semver: minor bump = additive features per DESIGN.md §17 roadmap)
- CONTRACT remains v1 FROZEN (no contract changes in this goal)
- Next goal candidates: **G-AT-PROD-2** (v0.3.1 P1 features) OR **G-AT-DAEMON-1** (v0.4 cross-runtime parallelism) OR **G-AT-SEC-1** (DESIGN §15.1 hardware signing + §22 cross-provider auditor)
- Owner picks next goal based on real dogfood signal

## Dependencies

- CONTRACT v1 FROZEN ✓ (commit `2affaa9`)
- Owner approval to enable disk cache (DESIGN.md §16 opt-in gate; Owner runs `aiplus agent cache --enable-disk` once before AC verification)
- Owner availability for ~1-2 dogfood sessions in Week 3-4

## Next action

Owner reviews this goal doc + sub-deliverable specs/briefings → ack or push-back → Advisor coordinates CEO codex execution of phases A→E. Acceptance evidence committed to repo. Advisor closes goal with retrospective.

---

— Advisor, 2026-05-18, against DESIGN.md v0.1.2 + CONTRACT v1 FROZEN at `2affaa9`
