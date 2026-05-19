# Goal G-AT-AUTOSUMMON-INTENT-1 — Replace keyword auto-summon with LLM intent classification

**Status**: PROPOSED — Owner-authorized 2026-05-19. Wave 2-B (parallel with C and D).
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AUTOSUMMON-INTENT-1`

---

## TL;DR

v0.6.2 (G-AT-PROD-2) shipped expert auto-summoning via brittle keyword matching: each role's TOML had `[autosummon] keywords = ["payment", "auth", ...]` and coordinator did string-contains. Per Owner direction, replace this with **LLM-based intent classification** — each role declares an `intent_hint` natural-language description, and the coordinator asks a small LLM "does this task match this intent?" Reuse v0.6.0 G2 (semantic dispatch gate) infrastructure for the LLM call. Cache results by (task hash, intent_hint hash) to bound cost.

## Success criterion (single sentence)

> Owner says `aiplus agent route "实现支付接口"` → coordinator classifies HEAVY → for each role with `[autosummon] intent_hint = "..."`, coordinator asks LLM "does '实现支付接口' match '...'?" → security-reviewer's `intent_hint = "支付/认证/敏感数据/credentials/凭据相关的工作"` returns YES → security-reviewer added to staffing; tech-writer's `intent_hint = "文档/README/教程/API 文档相关的工作"` returns NO → not added; cache hit on repeat dispatch within session.

## Goals (D1-D5)

- **D1**: Replace keyword match with LLM intent classification — coordinator.rs `expert_keyword_match` function (or equivalent) ripped out and replaced with `expert_intent_match`.
- **D2**: Role TOML schema migration — `[autosummon] keywords = [...]` → `[autosummon] intent_hint = "..."`. Migrate 3 existing experts (security-reviewer / tech-writer / ai-integration-specialist).
- **D3**: Result caching — `(task_hash, intent_hint_hash) → bool` cached for the duration of a process; same task asked twice doesn't re-call LLM.
- **D4**: Reuse v0.6.0 G2 LLM-call infrastructure (whatever module hosts semantic dispatch gate's LLM call).
- **D5**: Calibration fixture updated — keyword-based test entries replaced with intent-based (CEO designs replacement entries; existing 26 entries stay byte-identical except the auto-summon section).

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/autosummon-intent-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/autosummon-intent-impl-notes.md` | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/agent/coordinator.rs` — replace keyword match with intent classification | PENDING — CEO |
| 5 | `.aiplus/agents/*.toml` (security-reviewer / tech-writer / ai-integration-specialist) — schema migration | PENDING — CEO |
| 6 | New + extended tests covering intent classification + cache behavior | PENDING — CEO |
| 7 | Calibration fixture auto-summon entries updated | PENDING — CEO |

## Phases

```
Phase 1 (CEO codex, ~0.5 day):
  - Locate G2 semantic dispatch gate LLM call infra (grep for it)
  - Design intent classification prompt template
  - Design cache key + size limit + invalidation
  - Design role TOML schema migration (intent_hint field shape)

Phase 2 (CEO codex, ~1.5 days):
  - Strip keyword match path
  - Wire intent classifier into coordinator
  - Migrate 3 expert TOMLs
  - Update calibration fixture auto-summon section

Phase 3 (CEO codex, ~0.5 day):
  - All tests green + new tests for intent match + cache hit
  - Live smoke: 3 D5-style tasks confirm right experts auto-summoned
```

**Total target**: ≤ 3 days.

## Out of scope

- **route.rs** — B's work entirely in coordinator.rs (this is the ownership designation per CONTRACT App D)
- All other coordinator scoring rubric (calibration-locked)
- Adapter code, CONTRACT.md, hardware signing, tamper-evident log
- Adding NEW experts (just migrating existing 3)
- Persistent cache across processes (in-process only for v1)

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| G2 semantic gate LLM-call infra harder to reuse than expected | If reuse fails, write a minimal local LLM call helper (one Claude Haiku API call) — adds ~50 LoC |
| Intent classification LLM cost adds up | In-process cache; Haiku ($0.25/M input tokens) means ~$0.0001 per classification; ~$0.01 for 100 distinct tasks |
| Cache key collision (different tasks hash same way) | Use task FULL text hash (sha256) — collision negligible |
| Briefing's "B doesn't touch route.rs" assumption breaks | If broken, STOP and ping Advisor — the parallel-with-D safety depends on this assumption |
| 3 expert intent_hints don't actually trip in real dogfood | Calibration fixture documents intent_hints' expected behavior; Owner can tune intent_hint via TOML edit (not code change) post-dogfood |

## Acceptance criteria

- ✓ All 5 D-goals met
- ✓ `cargo test --package aiplus-cli` PASS
- ✓ Live: `aiplus agent route --score-only "实现支付接口"` → staffing includes security-reviewer (auto-summoned by intent, NOT keyword)
- ✓ Live: same call within same process is faster (cache hit, no LLM re-call); test asserts via instrumentation if available
- ✓ `route.rs` byte-for-byte unchanged from post-Wave-1 baseline
- ✓ Calibration fixture's 16 original entries still byte-identical; only the auto-summon-specific 10 entries from v0.3.1 P1 are updated to intent-mode

## Dependencies

- Wave 1 (G-AT-AUDITOR-REMOVE-1) merged to main (clean baseline for parallel Wave 2)
- v0.6.0 G2 semantic dispatch gate ✓ (reuse target)
- v0.3.1 P1 auto-summon code as baseline to replace

## Next action

After Wave 1 merges, Advisor dispatches CEO for this goal in parallel with C and D.

---

— Advisor, 2026-05-19, Wave 2-B of post-4-goal sprint.
