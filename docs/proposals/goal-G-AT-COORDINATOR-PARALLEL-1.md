# Goal G-AT-COORDINATOR-PARALLEL-1 — Wire adaptive coordinator to Perf-1 parallelism

**Status**: PROPOSED — Owner sign-off pending. **Replaces G-AT-DAEMON-1** in the 4-goal program after Advisor ROI analysis showed daemon was premature optimization solving a non-bottleneck.
**Drafted by**: Advisor (post G-AT-PROD-2 close, 2026-05-18)
**Goal ID**: `G-AT-COORDINATOR-PARALLEL-1`

---

## TL;DR

v0.6.0 Perf-1 already shipped parallel batch dispatch (`route_batch` in `crates/aiplus-cli/src/agent/route.rs:637-707`) using `thread::spawn` + shared `WorktreePool`. But the v0.3.0 P0 + v0.3.1 P1 adaptive coordinator path (`run_adaptive_route` in `route.rs:152-244`) was wired with a **serial `for` loop** that calls `route_known_role` one role at a time, ignoring the existing parallel infrastructure.

**HEAVY tasks today run 6 roles serial — they should run mostly parallel.** Fixing this requires no new infrastructure; just wire the adaptive coordinator path to the existing batch primitive.

**Scope**: ~3-5 days. ~80% of what daemon promised (cross-runtime parallelism via subprocess fan-out) at 5% of daemon's cost (no IPC, no persistent process, no CONTRACT unfreeze).

## Success criterion (single sentence)

> Owner runs `aiplus agent route "实现支付接口"` → coordinator scores HEAVY → 6 roles dispatch via existing `route_batch` parallel primitive → wall-clock dispatch time is bounded by `max(role_times)` not `sum(role_times)` → real-world HEAVY task wall-time drops from ~6× serial-baseline to ~1-2× serial-baseline.

## Goals (D1-D4)

- **D1**: `run_adaptive_route` calls `route_batch` (or equivalent N-way parallel primitive) instead of serial `for` loop, for all tiers that staff ≥ 2 roles (LIGHT_CODE, MEDIUM, HEAVY).
- **D2**: WorktreePool race-condition safe under 6-way concurrent fan-out (Perf-1 tested at 1+2; v0.3 HEAVY = 6-way). Either confirm safe via test or add necessary synchronization.
- **D3**: Performance baseline test: HEAVY task wall-clock before/after. Acceptance: ≥ 2× speedup on HEAVY tier dispatch overhead (excluding actual LLM work which dominates anyway).
- **D4**: All 348 existing aiplus-cli tests + 543 workspace tests still PASS. New tests cover N-way parallel correctness, partial-failure semantics (1 role fails, others continue?), and worktree race regression.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/coordinator-parallel-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/coordinator-parallel-impl-notes.md` (CEO Phase 1 design + Phase 3 evidence) | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/agent/route.rs` — `run_adaptive_route` calls parallel primitive | PENDING — CEO |
| 5 | `crates/aiplus-cli/src/agent/worktree_pool.rs` — race-safety review + fixes if needed | PENDING — CEO |
| 6 | New integration test: `tests/coordinator_parallel_smoke.rs` (N-way fan-out correctness) | PENDING — CEO |
| 7 | Performance baseline measurement in impl-notes evidence trailer | PENDING — CEO |
| 8 | CHANGELOG `## 0.6.3` entry (Advisor writes Phase C) | PENDING — Advisor |

## Phases + sequencing

```
Phase 1 (CEO codex, 0.5-1 day):
  - Read Perf-1 batch primitive (route.rs:637-707) + WorktreePool source.
  - Decide: reuse route_batch unchanged, or refactor into N-way coordinator_batch?
    Trade-off: route_batch has Primary/Sidecar role distinction; for adaptive
    coordinator the 6 roles may all be peers. CEO decides.
  - Write impl-notes design: dispatch model, partial-failure semantics
    (1 role panics → others continue?), worktree-pool race assessment.

Phase 2 (CEO codex, 1-2 days):
  - Wire run_adaptive_route → parallel primitive.
  - Add tests: 6-way parallel correctness, partial-failure, worktree race.
  - Performance baseline: time a HEAVY task before/after (mock runtime ok).

Phase 3 (CEO codex, 0.5-1 day):
  - All 348+543 existing tests green.
  - D5-style task wall-time evidence.
  - Trailer in impl-notes.
```

**Total target**: ≤ 5 days wall-clock.

## Out of scope

- **CONTRACT.md edits** — v1.1 FROZEN; parallelism is execution-layer, no contract change.
- **Adapter code** — frozen at G-AT-PROD-1 close.
- **Daemon, IPC, persistent process** — explicitly rejected per ROI analysis. Daemon may be revisited in v0.5+ if dogfood signal demands.
- **Cross-runtime distribution strategy** — assigning role X to runtime Y is `agent set-runtime` business; parallelism just means "whichever runtime each role is bound to runs concurrently with others".
- **API rate limit / quota handling** — partial-failure semantics covers this (rate-limited role fails → others continue), but no rate-aware scheduler.
- **v0.6.3 release artifact publishing** — separate Owner action.

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| WorktreePool has race condition at 6-way fan-out that Perf-1 1+2 testing missed | MEDIUM | MEDIUM (corrupt worktree state) | Phase 1 design includes pool race assessment; add `parking_lot::Mutex` or `Arc<RwLock>` if needed; test with `loom` if available |
| Partial failure semantics unclear — 1 role panics, others mid-flight | HIGH | LOW (worst case: orphaned runtime processes, no data loss) | Phase 1 decides explicit policy (e.g., "let failed role's thread panic; gather Result from others; report partial success"); test this case |
| 6-way concurrent adapter spawn hits API rate limit / quota cliff | MEDIUM | LOW (rate-limited role fails, others succeed → partial result) | Acceptable failure mode; document for Owner; v0.6.4 could add semaphore if needed |
| Wall-clock improvement smaller than expected (adapter spawn small fraction of total) | MEDIUM | LOW | If true, validates the Advisor analysis that daemon was overkill; ship anyway since 3-5 day spend; absorbed into goal completion |
| Refactoring route_batch breaks Perf-1's existing `--workflow author-critic-fixer` callers | LOW | MEDIUM | Don't refactor; create a NEW parallel primitive if route_batch's primary/sidecar shape doesn't fit. Adapter coordinator dispatch is a new call site, not a replacement |

## Acceptance criteria

- ✓ D1-D4 met
- ✓ HEAVY task wall-clock for dispatch overhead ≥ 2× faster than current serial
- ✓ Partial failure (1 role errors) doesn't deadlock or hang other roles
- ✓ All 348+543 existing tests PASS
- ✓ New `coordinator_parallel_smoke.rs` tests PASS
- ✓ No new clippy warnings introduced
- ✓ CHANGELOG `## 0.6.3` entry (Advisor)

## What happens when goal completes

- `aiplus-cli` bumps to v0.6.3 (semver patch — additive parallel execution)
- 4-goal program advances 3/4 (CONTRACT-V1.1 ✓, PROD-2 ✓, COORDINATOR-PARALLEL-1 ✓; G-AT-SEC-1 remains)
- Owner has the option to start G-AT-SEC-1 OR pause for actual v0.3.1+parallel dogfood before deciding more goals.

## Dependencies

- CONTRACT v1.1 FROZEN ✓ (`6cd4e9b`)
- v0.3.1 P1 shipped ✓ (`aiplus-public@a5fa4e5`)
- Perf-1 batch primitive exists ✓ (`route.rs:637-707`)
- WorktreePool exists ✓ (`worktree_pool.rs`)

## Why this goal replaces G-AT-DAEMON-1

Advisor analysis (informal, recorded in retrospective post-completion): daemon's 4-6 week dev cost vs ~20 min/week Owner-time savings = 8-12 month ROI break-even, even before counting daemon's bug surface, IPC complexity, CONTRACT unfreeze risk, and background-process onboarding cliff. Cross-runtime parallelism is the genuine value daemon promised, and it's already 90%-built (Perf-1 batch primitive). Wiring it in is 3-5 days, not 4-6 weeks. Skipping daemon preserves CONTRACT v1.1 FROZEN.

If dogfood after this goal shows spawn-time IS still a real bottleneck (e.g., Owner does 50+ dispatches/day and per-dispatch spawn cost matters), daemon can be revisited as a v0.5 milestone. Default: don't build until evidence demands it.

## Next action

Owner reads goal + companion briefing → ack or push-back → Advisor dispatches CEO codex.

---

— Advisor, 2026-05-18, against CONTRACT v1.1 FROZEN at `6cd4e9b`, aiplus-cli v0.6.2 at `a5fa4e5`. Replaces G-AT-DAEMON-1 in the 4-goal program.
