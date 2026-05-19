# Auto-summon Intent Classification — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session (Wave 2-B; runs in parallel with Wave 2-C and Wave 2-D)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AUTOSUMMON-INTENT-1.md`
- v0.3.1 P1 baseline auto-summon: `aiplus-public/crates/aiplus-cli/src/agent/coordinator.rs` (keyword match function)
- v0.6.0 G2 semantic dispatch gate (LLM-call infra to reuse) — locate via `grep -rn "semantic\|dispatch.gate\|G2" crates/aiplus-cli/src/`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.autosummon-intent/`
- Branch:    `feat/autosummon-intent`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.autosummon-intent -b feat/autosummon-intent`
- All work inside this worktree.

Shared-file ownership for this round (CRITICAL — Wave 2 runs in parallel; collisions will hurt):
- This session OWNS: `crates/aiplus-cli/src/agent/coordinator.rs` (auto-summon section ONLY — NOT scoring rubric), `.aiplus/agents/security-reviewer.toml` + `tech-writer.toml` + `ai-integration-specialist.toml` (or equivalent expert files), new tests in `crates/aiplus-cli/tests/`, calibration fixture auto-summon entries
- This session must NOT modify: `crates/aiplus-cli/src/agent/route.rs` (Wave 2-D owns), `crates/aiplus-cli/src/agent/doctor.rs` (Wave 2-D owns), `crates/aiplus-cli/src/agent/commands.rs` (Wave 2-C owns), workspace `Cargo.toml` (Wave 2-C owns), CONTRACT.md (FROZEN), scoring rubric in coordinator.rs, calibration fixture existing 16 baseline entries, adapter code

## Scope — replace keyword match with LLM intent

Single semantic change: where v0.3.1 P1 ships `[autosummon] keywords = [...]` and string-match, this goal ships `[autosummon] intent_hint = "..."` and LLM-judged intent match.

## Implementation outline

### Step 1: Locate G2 semantic dispatch gate infrastructure

`grep -rn "semantic\|dispatch.gate" crates/aiplus-cli/src/` finds it. Goal: find the existing module that makes lightweight LLM calls for intent parsing. Reuse it. If reuse path is not clean, fall back to a minimal local helper (one HTTP call to Anthropic Messages API with Haiku model).

### Step 2: Role TOML schema

Replace this:
```toml
[autosummon]
keywords = ["payment", "auth", "credentials", ...]
match_mode = "any"
priority = 100
```

With this:
```toml
[autosummon]
intent_hint = "支付/认证/敏感数据/credentials/凭据相关的工作"
priority = 100
```

(`match_mode` is no longer needed — LLM either says yes or no.)

Apply to 3 experts: `security-reviewer`, `tech-writer`, `ai-integration-specialist`. CEO picks the actual intent_hint strings — examples above are starting points; final wording is CEO design choice based on each persona file.

### Step 3: Coordinator integration

In `coordinator.rs`:
- Build registry of `(role_name, intent_hint)` pairs at startup from role TOMLs (same as keyword registry today, just different field)
- For each task being scored, after tier + risk-forced staffing, walk registry:
  - For each (role, intent_hint): call `intent_match(task, intent_hint) -> bool`
  - If true, add role to staffing
- `intent_match` calls LLM with structured prompt (CEO designs template; suggestion below)
- Apply cluster cap (same as v0.3.1 P1: tier baseline + 2 max)

### Step 4: Intent classifier prompt

Suggested template (CEO finalizes):

```
You are classifying whether a software task matches an intent description.

Task: "{task}"
Intent: "{intent_hint}"

Does this task match this intent? Reply with a single word: YES or NO.
```

Model: Anthropic Claude Haiku (cheapest + fastest tier). Cost: ~$0.0001 per classification.

If the LLM call fails (network / quota / parse error): default to NO (fail-closed — don't summon spurious experts).

### Step 5: Cache

In-process LRU cache, key = sha256(task + "\n" + intent_hint), value = bool. Size cap: 1000 entries (overflow evicts oldest). Cleared on process exit (no persistence in v1).

If a task is classified twice in the same process (e.g., score-only then real route), second call is cache hit and instant.

### Step 6: Calibration fixture update

In `crates/aiplus-cli/tests/fixtures/coordinator_calibration.toml`:
- Original 16 entries (v0.3.0 polish baseline): byte-identical, do NOT touch
- The 10 entries added in v0.3.1 P1 covering auto-summon: REWRITE for intent mode. Each entry asserts (task → which experts auto-summoned). The TOML schema may need a slight adjustment if it currently encodes "expected matched keywords"; switch to "expected matched intents" or just "expected auto-summoned roles".

Final fixture entry count: still 26.

### Step 7: Tests

- New `tests/autosummon_intent_smoke.rs` (or extend existing `score_only_smoke.rs` — CEO's organizational choice):
  - 3 D5-style tasks each hit a specific expert's intent
  - 1 negative task (`describe git status`) auto-summons nothing
  - 1 cache-hit test (call intent_match twice with same args, second is cache hit)

## Phase structure

### Phase 1 — Write `docs/proposals/autosummon-intent-impl-notes.md` BEFORE coding

Required sections:
1. G2 reuse decision: which file/function hosts G2's LLM call? Reuse cleanly or local helper? Document with paths.
2. Prompt template final version (with examples of expected YES/NO).
3. Cache design: key formula, size cap, eviction policy.
4. Role TOML schema migration: exact intent_hint for each of 3 experts.
5. Calibration fixture rewrite plan: which entries change, which assertions move from keyword-based to intent-based.
6. CHANGELOG draft text for v0.6.5.

### Phase 2 — Implement

Order: prompt + cache helper → registry build → intent_match call → integrate into staffing → TOML migration → fixture update → tests.

### Phase 3 — Evidence

- All tests green (existing 360 + new ~3-5 = ~363-365 aiplus-cli)
- Live smoke: 3 D5 tasks confirm right experts auto-summoned
- `route.rs` unchanged from baseline (Wave 2 isolation invariant)

## STOP rule

- **STOP only on `_REGRESSION_`** (non-auto-summon test fails) or `_SCOPE-BREACH_` (touched a file in the "must not modify" list, especially `route.rs`)
- **Do NOT STOP** on `_IMPL-BUG_`
- Retry-once gate mandatory before classification

**Special**: if you find G2 reuse impossible and need to add new dependencies (new crate, new module), STOP and ping Advisor — that's a scope decision.

## Conformance / verification

- All ACs in goal doc met
- `cargo test --package aiplus-cli` PASS
- Live `aiplus agent route --score-only "实现支付接口"` produces security-reviewer in staffing
- `route.rs` byte-identical to baseline
- Calibration fixture's first 16 entries byte-identical

## Scope fence

- **IN**: coordinator.rs auto-summon section, 3 expert TOMLs, calibration auto-summon entries, new tests
- **OUT**: route.rs (Wave 2-D), doctor.rs (Wave 2-D), commands.rs (Wave 2-C), workspace Cargo.toml (Wave 2-C), scoring logic, calibration baseline 16 entries, CONTRACT.md, adapter code, Cargo.toml version bump, CHANGELOG actual
- **GRAY**: small comment improvements

## Deliverables (6)

1. `docs/proposals/autosummon-intent-impl-notes.md`
2. `coordinator.rs` auto-summon section (intent classifier + cache + registry)
3. 3 expert TOML migrations
4. Calibration fixture auto-summon entries rewritten
5. New tests covering intent + cache
6. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+3d**.

## Handoff endpoint

Report `IMPL_VERDICT=...` + which sub-features deviated + cache hit metrics. Advisor coordinates with Wave 2-C and Wave 2-D Phase C (likely combined into single v0.6.5 ship if all 3 Wave 2 sessions land roughly the same time).

## Owner gates

Standard. BWS-backed: wrap any LLM-call test with `aiplus secret-broker run --aliases anthropic -- <cmd>`.

## Dependencies

- Wave 1 (G-AT-AUDITOR-REMOVE-1) merged to main
- v0.6.0 G2 semantic dispatch gate code
- v0.3.1 P1 auto-summon code as baseline

---

— Advisor, 2026-05-19. Wave 2-B. Day-0 narrow STOP + retry-once.
