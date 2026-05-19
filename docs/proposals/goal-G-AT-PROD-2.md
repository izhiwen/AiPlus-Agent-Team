# Goal G-AT-PROD-2 — v0.3.1 P1 Polish (adaptive coordinator)

**Status**: PROPOSED — Owner sign-off pending.
**Drafted by**: Advisor (post G-AT-CONTRACT-V1.1 close, 2026-05-18)
**Goal ID**: `G-AT-PROD-2` (Goal, Agent-Team, Production-readiness, milestone 2)

---

## TL;DR

v0.3.0 P0 shipped the adaptive coordinator skeleton (scorer + tier classifier + fixed-per-tier staffing). G-AT-PROD-2 ships the **3 P1 features** that make the skeleton intelligent: **expert auto-summoning** (task content → domain experts joined to staffing), **risk-based forced summoning** (high-risk overrides tier-baseline staffing), and **TTL honoring** (warm bench cache actually expires per role config). End state: `aiplus agent route "<task>"` produces staffing that adapts to task content, not just task tier. ~2 weeks of work.

## Success criterion (single sentence)

> Owner says "实现支付接口" → coordinator scores 5/0.85 → HEAVY tier baseline staffs [pm, architect, engineer-a, engineer-b, reviewer, qa] → expert auto-summon adds **security-expert** (keyword "支付/接口") → risk-based forced adds nothing new (reviewer already in HEAVY) → final staffing 7 roles → CEO dispatches → 30 min later Owner re-dispatches same task → disk-warm cache loads cleanly (TTL=1800s not exceeded) → second dispatch starts instantly. **Same task at T+2h → TTL exceeded, role cold-starts, dispatch-log records `ttl_expired=true`.**

## Goals (D1-D5)

- **D1**: Expert auto-summoning — new `[autosummon]` schema in role TOML; coordinator parses task → matches expert trigger sets → adds experts to staffing list. ≥3 of 6 v0.1-shipped experts have working trigger sets (per `v0.3-adaptive-coordinator.md` AC).
- **D2**: Risk-based forced summoning — risk ≥ 0.7 force-adds reviewer regardless of tier; risk ≥ 0.85 also force-adds qa.
- **D3**: TTL honoring — `warm_bench_ttl_seconds` from role TOML enforced at dispatch time; expired cache → cold-start path; `ttl_expired` field in `coordinator_decision` dispatch-log event.
- **D4**: Calibration test extension — existing 16-entry fixture from v0.3.0 polish patch UNCHANGED + new entries covering expert-trigger keywords + risk-forced thresholds + TTL invalidation. Final fixture ~25-30 entries.
- **D5**: D5 success-criterion scenario reproducible end-to-end without Owner intervention.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/v0.3.1-p1-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/v0.3.1-p1-impl-notes.md` (Phase 1 design notes + Phase 3 evidence trailer) | PENDING — CEO codex |
| 4 | `crates/aiplus-cli/src/agent/coordinator.rs` extension (auto-summon + risk-forced) | PENDING — CEO codex |
| 5 | `crates/aiplus-cli/src/agent/cache.rs` extension (TTL enforcement) | PENDING — CEO codex |
| 6 | `crates/aiplus-cli/src/agent/route.rs` extension (new `ttl_expired` log field; auto-summon plumbing) | PENDING — CEO codex |
| 7 | `crates/aiplus-cli/src/agent/doctor.rs` extension (TTL-age WARN) | PENDING — CEO codex |
| 8 | Role TOML schema extension: optional `[autosummon] keywords = [...]` | PENDING — CEO codex |
| 9 | Calibration fixture extension (~10 new entries) | PENDING — CEO codex |
| 10 | New integration tests: `auto_summon_smoke.rs`, `risk_forced_smoke.rs`, `ttl_invalidation_smoke.rs` | PENDING — CEO codex |

## Phases + sequencing

```
Phase 1 (CEO codex, 1-2 days):
  - Write impl-notes.md design plan first (per established pattern)
  - Sections: auto-summon schema, expert trigger sets (≥3 experts), risk thresholds,
    TTL invalidation algorithm, calibration extension matrix, test plan

Phase 2 (CEO codex, ~1 week):
  - Impl order: D3 TTL (smallest, isolated to cache.rs) → D2 risk-forced
    (coordinator.rs staffing extension) → D1 auto-summon (largest; new
    schema + parser + 3+ expert trigger sets).
  - Each feature ships with unit + integration tests before moving to next.

Phase 3 (CEO codex, 1-2 days):
  - Calibration fixture extension (~10 new entries) — covers all 3 features
  - D5 acceptance scenario reproduction
  - Evidence trailer in impl-notes.md
  - cargo test --package aiplus-cli all green
```

**Total target**: ≤ 2 weeks wall-clock.

## Out of scope

- **CONTRACT.md edits** — v1.1 FROZEN; this goal does not touch adapter contract.
- **Adapter code** — frozen at G-AT-PROD-1 close.
- **`coordinator.rs` scoring rubric** — calibration test fixture from v0.3.0 polish locks current scoring behavior; this goal extends staffing logic, not scoring.
- **v0.3 P2 features** — manual `--tier HEAVY` override, score audit trail, disagreement signal. Those are G-AT-PROD-2.1 if pursued, or absorbed into G-AT-DAEMON-1.
- **New experts / role personas** — auto-summon uses existing 6 v0.1-shipped experts + 5 v0.2 stubs; no new role files this goal.
- **DESIGN.md §15.1 hardware signing / §22 cross-provider auditor** — G-AT-SEC-1 scope.

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Expert keyword sets are blind guesses (no dogfood data) — over- or under-staff in real use | MEDIUM | LOW (observable, easy to tune via TOML) | Conservative initial keyword sets (start small); calibration fixture pins current behavior; tune via TOML edits (not code) once dogfood data arrives |
| Risk-forced thresholds (0.7, 0.85) are blind — could over-staff LIGHT tasks | MEDIUM | LOW (token cost, easily tuned) | Same: calibration fixture pins; thresholds in const, easy to flip in v0.3.2 if needed |
| TTL enforcement breaks existing dogfood cache that Owner wanted to keep | LOW | LOW (cold-start is functional, just slower) | TTL enforcement is opt-in for the first release (`[cache] enforce_ttl = false` default = current behavior; `true` enables enforcement). Owner can opt-in after Phase 3 PASS |
| Role TOML schema extension breaks existing 19 role configs that don't have `[autosummon]` | LOW | LOW | New section is optional; absent → no auto-summon (current behavior); schema_version field detects absence cleanly |
| 3 features interact unexpectedly — auto-summon + risk-forced + tier-baseline → over-staffed cluster | MEDIUM | LOW (cap at HEAVY-tier max of 8 roles) | Staffing dedup is enforced; calibration fixture covers interaction cases; final-cluster cap = HEAVY-tier staffing + 2 extras max |
| Dogfood signal arrives mid-goal showing P1 features wrong direction | LOW | MEDIUM (rework or descope) | Owner explicitly skipped dogfood; this risk is accepted. Checkpoint at Phase 3 end before declaring v0.3.1 shipped |

## Acceptance criteria (for goal completion)

- ✓ D1-D5 all met
- ✓ All 10 acceptance scenarios in briefing AC1-AC10 PASS
- ✓ `cargo test --package aiplus-cli` all green (existing 342 + new tests)
- ✓ Calibration fixture extended; ALL existing 16 entries still PASS unchanged
- ✓ D5 success-criterion scenario (`实现支付接口` → 7-role staffing + TTL warm hit + TTL cold-start at T+2h) reproducible
- ✓ No regression in v0.3.0 P0 behavior (disk_warm_cache tests + v03_adaptive_coordinator test still PASS)
- ✓ CHANGELOG entry for v0.6.2

## What happens when goal completes

- `aiplus-cli` bumps to v0.6.2 (semver patch — additive features within v0.3 line)
- v0.3.1 ships
- **Owner Checkpoint**: 1 week of dogfood `v0.3.1` to see if real bottleneck is spawn time. If yes → G-AT-DAEMON-1 priority. If no → re-prioritize G-AT-SEC-1 / G-AT-DAEMON-1.

## Dependencies

- CONTRACT v1.1 FROZEN ✓ (`6cd4e9b`)
- v0.3.0 P0 + polish patch shipped ✓ (`aiplus-public@6710565`)
- Disk cache opt-in (v0.2) infrastructure ✓
- Calibration test fixture from v0.3.0 polish (DON'T modify existing 16 entries; extend)

## Next action

Owner reads this goal + companion briefing → ack or push-back → Advisor dispatches CEO codex with iteration-2-quality briefing (narrow STOP + retry-once gate baked in from start, per `advisor_briefing_stop_rules` auto-memory).

---

— Advisor, 2026-05-18, against CONTRACT v1.1 FROZEN at `6cd4e9b`, aiplus-cli v0.6.1 at `6710565`
