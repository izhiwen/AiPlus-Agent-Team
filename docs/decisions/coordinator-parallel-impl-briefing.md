# Coordinator Parallelism — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-18 by Advisor (post G-AT-PROD-2 close)
**Target executor**: CEO codex session (full impl — Advisor verifies + ratifies)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-COORDINATOR-PARALLEL-1.md`
- Perf-1 baseline: `aiplus-public/crates/aiplus-cli/src/agent/route.rs:637-707` (`route_batch`)
- WorktreePool: `aiplus-public/crates/aiplus-cli/src/agent/worktree_pool.rs`
- Adaptive coordinator path (regression site): `aiplus-public/crates/aiplus-cli/src/agent/route.rs:152-244` (`run_adaptive_route`)
- CONTRACT v1.1 FROZEN: `aiplus-agent-team/adapters/CONTRACT.md`
- Auto-memory: `advisor_briefing_stop_rules.md` (narrow STOP + retry-once gate baked in below)

---

## TL;DR

Wire the adaptive coordinator dispatch path to the existing Perf-1 parallel batch primitive. HEAVY-tier dispatch today serial; should be ≥ 2× faster after this goal. ~3-5 days. No new infrastructure.

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.coord-parallel-1/`
- Branch:    `feat/coordinator-parallel-1`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.coord-parallel-1 -b feat/coordinator-parallel-1`
- All work inside this worktree. Do NOT touch `~/Projects/AiPlus/aiplus-public/` main worktree.
- Push to `feat/coordinator-parallel-1` only. Advisor handles main merge after Phase 3 ratification.

Shared-file ownership:
- This session OWNS: `crates/aiplus-cli/src/agent/route.rs`, `crates/aiplus-cli/src/agent/worktree_pool.rs`, new tests in `crates/aiplus-cli/tests/`
- This session must NOT modify: `adapters/CONTRACT.md` (FROZEN), `crates/aiplus-cli/src/agent/coordinator.rs` scoring rubric (calibration-locked), `crates/aiplus-cli/tests/fixtures/coordinator_calibration.toml` existing 26 entries, any adapter code

## Background — what we found

In `route.rs`:

```rust
// Line 152-244: run_adaptive_route (v0.3.0+ path, called for `aiplus agent route "<task>"`)
for (idx, role) in plan.staffing_roles.iter().enumerate() {
    route_known_role(project_root, role, ..., DispatchKind::Primary or Sidecar, ...)?;
    //               ^^^^^^^^^^^^^^^^^^^^^^^                                       ^
    //               Serial — each call blocks the next                            ? unwraps before continuing
}
```

vs.

```rust
// Line 637-707: route_batch (Perf-1 path, called via --workflow flag)
for sidecar in sidecars {
    handles.push(thread::spawn(move || { route_known_role(...) }));  // ← parallel
}
for handle in handles { handle.join()?; }
```

`coordinator_role_task` (line 429) just wraps the task with `"role {position}/{total}"` framing — there is **NO code-level dependency chain enforced between roles**. PM/Architect/Engineer-A/B/Reviewer/QA workflow is convention-only; from execution standpoint, all 6 roles are independent peers each receiving the same root task.

**Therefore**: HEAVY's 6 roles can all run in parallel without breaking correctness. CEO integrates results after all complete.

## Phase 1 — Write `docs/proposals/coordinator-parallel-impl-notes.md` BEFORE code

Required sections:

1. **Dispatch model decision** — two options:
   - **(a) Reuse `route_batch` unchanged** by treating staffing_roles[0] as primary and rest as sidecars. Pro: zero refactor risk. Con: forces a Primary/Sidecar role distinction that's artificial for adaptive coordinator (all roles peer).
   - **(b) Build new `coordinator_batch` peer-based primitive** alongside route_batch. Pro: clean model. Con: parallel infra duplication.
   
   CEO picks based on actual route_batch surface area. Document trade-off in notes.

2. **Partial-failure policy** — when 1 of 6 roles errors mid-flight, what happens to the other 5?
   - Recommendation: each role gets its own thread; one thread's Err is collected, not propagated immediately; other threads continue; final result aggregates `Vec<Result<RoleOutput, Error>>`. CEO writes this up.
   - Edge case: what if WorktreePool entry held by a panicking thread? Audit Drop semantics.

3. **WorktreePool race assessment** — Perf-1 tested at 1 primary + 2 sidecars = 3-way. Adaptive HEAVY = 6-way. Verify lock contention, deadlock risk, race condition in pool acquisition logic. If issue found: fix (likely add fine-grained per-role lock or use `parking_lot::RwLock`).

4. **Test plan** — `tests/coordinator_parallel_smoke.rs` covers:
   - 6-way fan-out completes (all 6 results collected)
   - 1-role-fail partial: 5 succeed, 1 fail, result reflects this
   - WorktreePool contention: 6 threads racing for pool entries (use deterministic fake-runtime)
   - Wall-clock baseline: HEAVY task takes ≤ 1.5× single-role time (vs current 6× serial baseline)

5. **CHANGELOG `## 0.6.3` entry** (CEO drafts; Advisor edits in Phase C). One-line per feature; non-programmer friendly.

## Phase 2 — Implement

Order:
1. Choose (a) or (b) per Phase 1 decision.
2. Wire `run_adaptive_route` to chosen primitive.
3. Add `tests/coordinator_parallel_smoke.rs` with the 4 scenarios above.
4. Verify all 348 existing aiplus-cli tests + 543 workspace tests still PASS.

## Phase 3 — Evidence

- `cargo test --package aiplus-cli` — green
- `cargo test --workspace` — green
- Wall-clock baseline: time a HEAVY dispatch before and after (use mock/fake runtime to remove API variance)
- Optional: D5 task `aiplus agent route --score-only "实现支付接口"` to confirm planning unchanged (parallel is execution-only)

Trailer in impl-notes.md captures all 3 evidence items.

## STOP rule (day-0 narrow, per `advisor_briefing_stop_rules` auto-memory)

- **STOP only on `_REGRESSION_`** — existing v0.3.0/v0.3.0-polish/v0.3.1 test fails after parallel wiring → STOP + ping Advisor.
- **STOP only on `_SCOPE-BREACH_`** — touched forbidden file (CONTRACT.md / adapter code / scoring rubric / fixture existing entries) → revert.
- **Do NOT STOP on `_IMPL-BUG_`** — your new parallel code has a bug during dev. Fix and continue.

### Retry-once gate (mandatory before classifying any FAIL)

Before classifying any first-attempt FAIL as `_REGRESSION_`:
1. Re-run SAME (test × env) once, identical command.
2. If retry PASSes → record as `PASS-after-flaky-retry`, continue.
3. If retry FAILs → proceed to classification.

(Especially relevant here — parallel tests are flaky-prone. Retry-once is necessary not optional.)

## Conformance / verification

- All ACs in goal doc met.
- All 348 aiplus-cli tests + 543 workspace tests PASS.
- New parallel smoke test PASS.
- No new clippy warnings.
- HEAVY task wall-clock improvement ≥ 2× over serial baseline (measurable in benchmark or test timing).

## Scope fence

- **IN**: `route.rs` adaptive path, `worktree_pool.rs` race fixes, new tests, CHANGELOG note in impl-notes (Advisor commits to actual CHANGELOG).
- **OUT**:
  - `adapters/CONTRACT.md` (FROZEN)
  - Adapter code (frozen)
  - `coordinator.rs` scoring logic (calibration-locked)
  - Calibration fixture existing 26 entries
  - `Cargo.toml` version bump (Advisor handles)
  - `CHANGELOG.md` direct edit (Advisor handles, CEO drafts in impl-notes)
  - Any daemon / IPC / persistent process work (explicitly rejected)
- **GRAY**: small comment improvements in route.rs / worktree_pool.rs are OK.

## Deliverables (5)

1. `docs/proposals/coordinator-parallel-impl-notes.md` — Phase 1 design + Phase 3 evidence
2. `crates/aiplus-cli/src/agent/route.rs` — `run_adaptive_route` calls parallel primitive
3. `crates/aiplus-cli/src/agent/worktree_pool.rs` — race fixes if needed (else unchanged)
4. `crates/aiplus-cli/tests/coordinator_parallel_smoke.rs` — new tests
5. `CHANGELOG.md ## 0.6.3` draft text inside impl-notes (Advisor commits actual CHANGELOG)

## Time ceiling

**Hard ceiling: T+5d** (5 days). Faster expected — this is largely re-wiring existing infra.

## Handoff endpoint

After Phase 3 evidence committed to `feat/coordinator-parallel-1` and pushed:

- Report back with one of three verdicts:
  - `IMPL_VERDICT=PASS` — all ACs met; performance baseline met; all tests green
  - `IMPL_VERDICT=PASS_WITH_DEVIATIONS` — ACs met but design deviated (e.g., chose (a) when (b) was expected, or added unforeseen lock); Advisor decides accept
  - `IMPL_VERDICT=BLOCKED` — `_REGRESSION_` or `_SCOPE-BREACH_` after retry
- Phase 3 summary in impl-notes with wall-clock numbers

Advisor verifies + ff-merges + bumps aiplus-cli to v0.6.3 + writes CHANGELOG + writes retrospective.

## Owner gates (always active)

- NO push to `origin/main`. Push only `feat/coordinator-parallel-1`.
- NO `adapters/CONTRACT.md` edits.
- NO scoring rubric changes.
- NO calibration fixture existing 26 entries changes.
- NO global config edits.
- NO force ops, NO `--amend`, NO hook bypass.
- BWS-backed secret broker: wrap with `aiplus secret-broker run --aliases anthropic,openai -- <cmd>`. NOT `secret-broker need` (keyring-only).

## Dependencies

- v0.3.1 P1 shipped ✓ (`aiplus-public@a5fa4e5`)
- Perf-1 batch primitive exists ✓ (`route.rs:637-707`)
- WorktreePool exists ✓ (`worktree_pool.rs`)
- CONTRACT v1.1 FROZEN ✓ (`aiplus-agent-team@f784eaa`)

---

— Advisor, 2026-05-18, against CONTRACT v1.1 FROZEN. Day-0 narrow STOP + retry-once gate per `advisor_briefing_stop_rules` auto-memory.
