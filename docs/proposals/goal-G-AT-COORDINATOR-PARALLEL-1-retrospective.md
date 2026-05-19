# Goal G-AT-COORDINATOR-PARALLEL-1 — Retrospective (COMPLETE)

**Status**: **DONE** — v0.3 coordinator parallelism shipped 2026-05-18.
**Original goal**: `docs/proposals/goal-G-AT-COORDINATOR-PARALLEL-1.md`
**Closing date**: 2026-05-18 (same day as opening)
**Duration**: ~1 work day (vs. ≤5 day budget). Under budget by ~80%.

---

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | `docs/proposals/goal-G-AT-COORDINATOR-PARALLEL-1.md` (goal doc) | `b581ef1` | aiplus-agent-team |
| 2 | `docs/decisions/coordinator-parallel-impl-briefing.md` (CEO briefing, day-0 narrow STOP + retry-once baked in) | `b581ef1` | aiplus-agent-team |
| 3 | `docs/proposals/coordinator-parallel-impl-notes.md` (Phase 1 design + Phase 3 evidence) | `0f57f39` | aiplus-public |
| 4 | `crates/aiplus-cli/src/agent/route.rs` — new `coordinator_batch` peer primitive + `run_adaptive_route` rewired | `0f57f39` | aiplus-public |
| 5 | `crates/aiplus-cli/tests/coordinator_parallel_smoke.rs` — 4 new tests | `0f57f39` | aiplus-public |
| 6 | Version bump + CHANGELOG ## 0.6.3 + install.sh parity | Phase C | aiplus-public |
| 7 | This retrospective | Phase C | aiplus-agent-team |

**Total**: ~525 lines impl + ~300 lines spec/retrospective ≈ 825 lines.

## D-goal status

- D1 (adaptive coordinator calls parallel primitive): ✓ — new `coordinator_batch` peer primitive; `run_adaptive_route` rewired
- D2 (WorktreePool race-safe at 6-way fan-out): ✓ — audited, found safe under existing `Arc<Mutex<WorktreePool>>`, no fix needed
- D3 (≥ 2× HEAVY wall-clock speedup): ✓ — **5.7× measured** (5.40s serial → 0.94s parallel on 6-role × 900ms fixture)
- D4 (existing 348+543 tests still PASS + new tests added): ✓ — 352+547 (net +4/+4 from new parallel smoke tests)

## All ACs PASS

Goal doc ACs:
- ✓ D1-D4 met
- ✓ HEAVY task wall-clock for dispatch overhead ≥ 2× faster than serial (5.7× actual)
- ✓ Partial failure doesn't deadlock or hang other roles
- ✓ All existing tests PASS
- ✓ New `coordinator_parallel_smoke.rs` PASS (4 tests)
- ✓ No new clippy warnings introduced (CEO verified)
- ✓ CHANGELOG ## 0.6.3 entry (Advisor)

## What went well

1. **Day-0 narrow STOP + retry-once gate worked again.** Second consecutive goal where the iteration-1 briefing template (codified after G-AT-CONTRACT-V1.1 lesson) produced a clean first-try result. Pattern is now stable.
2. **CEO chose option (b) thoughtfully** — built `coordinator_batch` peer primitive instead of force-fitting `route_batch`'s primary/sidecar shape. Documented the trade-off in Phase 1 design notes (route_batch was inappropriate model for adaptive coordinator's peer roles). This is the right kind of judgment call to delegate.
3. **WorktreePool race audit caught nothing because there was nothing to catch.** The existing `Arc<Mutex<WorktreePool>>` already serialized correctly. CEO ran the audit (didn't skip it because "probably fine"), confirmed safe, documented in evidence. This is the right discipline — audit even when it might be a no-op.
4. **Performance massively exceeded target.** 2× was the AC; 5.7× was the result. Why? Because Advisor analysis correctly identified that the adaptive path was inadvertently serial — the gap was much wider than expected. The "missing wiring" framing turned out to be more accurate than the "spawn-time bottleneck" framing daemon was supposed to solve.
5. **CEO scope-discipline was textbook.** Clippy `-D warnings` failed on pre-existing untouched files (aiplus-core lint debt). CEO retry-once confirmed deterministic, classified as out-of-scope, did NOT attempt cleanup. This is a notable improvement over v0.3.1 P1 where CEO did good-citizen-fix on install.sh parity (correct outcome, but technically scope creep). Here CEO honored the fence strictly.
6. **G-AT-DAEMON-1 replacement validated.** This goal is the empirical case for "don't build daemon now": 1 day + 525 lines delivered 5.7× speedup with zero new infrastructure. Daemon's 4-6 week / new-IPC plan would have been a 30× overspend for the same speedup.

## What went wrong

1. **None this goal.** This is the first goal in the 4-goal program with zero "what went wrong" entries. The pattern of (a) better briefings + (b) tighter scopes + (c) Advisor verifying spec assumptions in code first (rather than relying on memory) is producing higher-quality outputs.

   The single deviation (clippy pre-existing lint debt) was handled correctly by CEO; that's not a "went wrong", it's "scope working as designed".

## Architectural lessons codified

1. **"Build the smaller goal first" pattern works.** When G-AT-DAEMON-1 was on the table at 4-6 weeks, Advisor surfaced an alternative (Alt A) that was 80% of value at 5% of cost. Goal documents should always include an "alternatives considered + rejected" section so future Advisors don't blindly inherit framing. Both this goal's `goal-G-AT-COORDINATOR-PARALLEL-1.md` (in "Why this goal replaces G-AT-DAEMON-1" section) and Advisor's conversational analysis preserved the reasoning.
2. **Verify in code before authoring spec.** Advisor's pivot to PARALLEL-FLAG-1 only happened because Owner asked "do we even have parallel today?" — which forced Advisor to grep the codebase. Result: discovered Perf-1 batch primitive existed and was unused. Lesson: when drafting performance goals, always verify the current state in code first, don't trust the prior goal's framing.
3. **5-day budgets are often 1-day goals in disguise** when the scope is correctly framed. G-AT-CONTRACT-V1.1 (1-week budget → 1 day), G-AT-PROD-2 (2-week budget → 1 day), this goal (5-day budget → 1 day). Budget inflation is a real pattern; future budgets should be more aggressive.

## Open follow-ups (not part of this goal)

- aiplus-core + older aiplus-cli pre-existing clippy lint debt could be a small standalone cleanup goal (~half day). Low priority; only worth doing if `cargo clippy -D warnings` becomes a binding CI gate.
- `dispatch-log.jsonl` ordering is no longer guaranteed for adaptive-coordinator-staffed roles. Downstream consumers should sort by `timestamp`. If a consumer surfaces during dogfood, that's a v0.6.4 patch.
- API rate-limit / quota awareness when 6 concurrent runtime sessions fire — currently relies on partial-failure semantics. If real Owner usage hits this, build a semaphore in v0.6.4+.
- Daemon: explicitly deferred to v0.5+ if dogfood shows persistent-process value (per analysis, unlikely).

## Next goal

Per the 4-goal program (with G-AT-DAEMON-1 replaced by this goal):

1. ✓ G-AT-CONTRACT-V1.1 — DONE
2. ✓ G-AT-PROD-2 — DONE
3. ✓ G-AT-COORDINATOR-PARALLEL-1 (this goal) — DONE
4. **G-AT-SEC-1** (hardware signing + cross-provider auditor) — DECISION PENDING

Advisor position on #4 has been consistent: G-AT-SEC-1 has clear value **only if Owner is preparing to distribute AiPlus to external users (open-source / team / customers)**. For solo Owner usage, hardware signing + cross-provider auditor is over-engineering. With 3/4 goals done in ~3 work days (vs ~10-week original budget for 4 goals), Owner has the option to:

- (a) Run G-AT-SEC-1 anyway (per the "好。全都做" commitment) — 3-4 weeks of work for distribution-readiness that may never matter
- (b) Re-prioritize #4: drop G-AT-SEC-1, pick a different next goal (memory layer / cost control / web UI / something dogfood surfaces)
- (c) Enter actual v0.3 dogfood window now that the toolchain is in a good state — let real usage drive the next goal selection

Advisor recommends (c) — given how fast goals 1-3 went, a few days of real dogfood before picking #4 is high-leverage. But Owner's call.

Advisor will surface "(a) / (b) / (c) for goal #4" decision after this retrospective lands.

---

— Advisor, 2026-05-18 (goal opened + closed same day, 3 of 4 in the program); against CONTRACT v1.1 FROZEN at `6cd4e9b`
