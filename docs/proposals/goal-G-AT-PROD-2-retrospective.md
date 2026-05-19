# Goal G-AT-PROD-2 — Retrospective (COMPLETE)

**Status**: **DONE** — v0.3.1 P1 polish shipped 2026-05-18.
**Original goal**: `docs/proposals/goal-G-AT-PROD-2.md`
**Closing date**: 2026-05-18 (same day as opening)
**Duration**: ~1 work day (vs. ≤2 week budget). Under budget by ~93%.

---

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | `docs/proposals/goal-G-AT-PROD-2.md` (goal doc) | `338d1d1` | aiplus-agent-team |
| 2 | `docs/decisions/v0.3.1-p1-impl-briefing.md` (CEO briefing, day-0 narrow STOP + retry-once) | `338d1d1` | aiplus-agent-team |
| 3 | `docs/proposals/v0.3.1-p1-impl-notes.md` (Phase 1 design + Phase 3 evidence) | `5aade60` | aiplus-public |
| 4 | Coordinator extensions (auto-summon + risk-forced) | `5aade60` | aiplus-public |
| 5 | Cache extensions (TTL enforcement; opt-in via `enforce_ttl=false` default) | `5aade60` | aiplus-public |
| 6 | Route + doctor extensions (new log fields + TTL doctor INFO/WARN) | `5aade60` | aiplus-public |
| 7 | Role TOML schema extension `[autosummon]` for 3 experts | `5aade60` | aiplus-public |
| 8 | Calibration fixture +10 entries (existing 16 locked) | `5aade60` | aiplus-public |
| 9 | New integration tests (auto-summon + risk-forced + TTL) | `5aade60` | aiplus-public |
| 10 | `install.sh` fallback parity bump | `5aade60` | aiplus-public |
| 11 | Version bump + CHANGELOG ## 0.6.2 entry | Phase C | aiplus-public |
| 12 | This retrospective | Phase C | aiplus-agent-team |

**Total**: ~800 lines impl + ~600 lines spec/retrospective = ~1400 lines.

## D-goal status

- D1 (Expert auto-summoning, ≥3 v0.1 experts): ✓ — `security-reviewer` / `tech-writer` / `ai-integration-specialist` shipped with trigger sets
- D2 (Risk-based forced summoning): ✓ — thresholds 0.7 (reviewer) / 0.85 (qa)
- D3 (TTL honoring, opt-in): ✓ — `[cache] enforce_ttl=false` default; new `ttl_expired` field
- D4 (Calibration test extension): ✓ — 26 entries total, 16 existing locked
- D5 (D5 acceptance scenario): ✓ — `write secure payment API docs` → 8-role staffing including auto-summoned security-reviewer + tech-writer

## All 10 ACs PASS

- AC1: `实现支付接口` → security-reviewer auto-summoned ✓
- AC2: `describe git status` → no expert auto-summoned ✓
- AC3: Multi-keyword `write secure payment API docs` → 2 experts + dedup ✓
- AC4: `--score-only` shows auto-summoned experts ✓
- AC5: risk=0.85 LIGHT_CODE → reviewer forced ✓
- AC6: risk=0.5 LIGHT_CODE → reviewer NOT forced ✓
- AC7: `forced_by_risk` field populated in log ✓
- AC8: stale cache + `enforce_ttl=true` → cold-start + `ttl_expired=true` ✓
- AC9: fresh cache → warm hit + `ttl_expired=false` ✓
- AC10: `enforce_ttl=false` → TTL ignored (regression safety) ✓

## What went well

1. **Day-0 narrow STOP + retry-once gate worked.** Per `advisor_briefing_stop_rules` auto-memory (codified after G-AT-CONTRACT-V1.1's iteration-2 lesson), this briefing baked in the correct STOP rule from the start. CEO never hit a false-blocker; `IMPL_VERDICT=PASS_WITH_DEVIATIONS` came back first-try.
2. **Blast-radius-ordered implementation.** TTL → risk-forced → auto-summon ordering meant each feature shipped cleanly with its own test before the next started. No cross-feature debugging needed.
3. **Opt-in TTL enforcement preserved Owner's current cache.** `enforce_ttl=false` default means v0.3.1 is a pure-additive release for existing users; nobody loses their warm cache.
4. **CEO surfaced (and fixed) a prior Advisor bug**: install.sh fallback was lagging Cargo version since v0.6.1 ship (Advisor oversight in v0.3.0 polish patch); CEO bumped it to keep parity test passing. This is exactly the kind of "good citizen scope creep" we want.
5. **Calibration fixture as regression baseline worked.** Existing 16 entries locked byte-identical; CEO appended 10 new. Confirms the v0.3.0 polish patch design held up under real extension.
6. **Test coverage grew naturally.** 342 → 348 aiplus-cli tests, 543 workspace tests; no test removed.

## What went wrong

1. **CEO chose to extend existing test files instead of creating new ones.** Briefing Deliverable #9 said "new integration tests: `auto_summon_smoke.rs`, `risk_forced_smoke.rs`, `ttl_invalidation_smoke.rs`". CEO extended `disk_warm_cache.rs` / `score_only_smoke.rs` / `v03_adaptive_coordinator.rs` instead. **Cost**: zero — functionally equivalent, ACs PASS, organizational choice. **Lesson**: future briefings should distinguish "behavior contract" (binding) from "file organization" (suggested, not binding) more clearly.
2. **install.sh fallback parity was a hidden carry-over bug**. Should have been caught during v0.3.0 polish ship. **Cost**: CEO had to fix it. **Lesson** (already in this retrospective + auto-memory candidate): "Version bump checklist must include install.sh fallback if parity test enforces."

## Architectural lessons codified

1. **Briefings should distinguish behavior contract from file organization.** AC list defines the binding outcome; file-organization suggestions are non-binding. Future briefings explicitly mark which is which.
2. **Day-0 narrow STOP + retry-once gate is the new baseline for ALL future Advisor briefings.** Applied here for the first time post `advisor_briefing_stop_rules` memory. Worked perfectly. Going forward, every briefing template starts with this section.
3. **Calibration fixture as regression-baseline contract works.** v0.3.0 polish locked 16 entries; v0.3.1 P1 extended to 26 without touching the original 16. Pattern is reusable for v0.3.2 / v0.4 / etc.

## Open follow-ups (not part of this goal)

- Auto-summon trigger sets for remaining 3 v0.1 experts (PM-Coach / Product-Reviewer / Data-Steward) — defer until dogfood signal arrives.
- 5 v0.2 stub experts need full impl before they can be auto-summoned — separate productionization track.
- Tune risk thresholds (0.7 / 0.85) based on real Owner dogfood — calibration fixture pins current behavior; thresholds in code constants, easy to flip when evidence accumulates.
- Auto-summon match-mode "all" (AND match) is implemented but no expert uses it yet; document use case when first expert needs it.
- TTL enforce_ttl flip-to-true rollout — Owner runs with default false for 1-2 weeks, then flips after observing cache behavior.

## Next goal

Per the 4-goal program Owner ratified ("好。全都做"):

1. ✓ **G-AT-CONTRACT-V1.1** — DONE (1 day).
2. ✓ **G-AT-PROD-2** (this goal) — DONE (1 day).
3. **G-AT-DAEMON-1** (v0.4 cross-runtime parallelism) — Owner checkpoint pending. Original plan: 1-week dogfood of v0.3.1 first to see whether spawn-time is really the bottleneck. With Owner's pace ("skip dogfood, do all 4"), Advisor may propose to start drafting G-AT-DAEMON-1 spec NOW + run dogfood in parallel during impl.
4. **G-AT-SEC-1** (hardware signing + cross-provider auditor) — runs parallel with G-AT-DAEMON-1 per the original 4-goal plan, in different repo per CONTRACT App D.

Advisor next-action: surface "G-AT-DAEMON-1 vs dogfood checkpoint" decision to Owner.

---

— Advisor, 2026-05-18 (goal opened + closed same day, 2 of 4 in the program); against CONTRACT v1.1 FROZEN at `6cd4e9b`
