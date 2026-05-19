# Polish-1 Bundle — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session (Wave 2-D; parallel with B and C)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-POLISH-1.md`
- AdapterResult plumbing baseline: `crates/aiplus-cli/src/agent/route.rs:route_known_role` (post-Wave-1)
- Briefing source material for Skill: `aiplus-agent-team/docs/decisions/*.md` (8 prior briefings)
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.polish-1/` (for aiplus-public changes)
- Branch:    `feat/polish-1` (in aiplus-public)
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.polish-1 -b feat/polish-1`

**Cross-repo note**: D6 (Skill file) is in `~/Projects/AiPlus/aiplus-agent-team/skills/`. For that file:
- Don't create a parallel worktree of aiplus-agent-team (Advisor handles aiplus-agent-team commits at Phase C ratification).
- Instead: write the Skill content directly into `~/Projects/AiPlus/aiplus-agent-team/skills/aiplus-ceo-briefing.md` in the main aiplus-agent-team worktree. Advisor reviews + commits.

Shared-file ownership (CRITICAL — Wave 2 parallel):
- This session OWNS: `crates/aiplus-cli/src/agent/route.rs`, `crates/aiplus-cli/src/agent/doctor.rs`, install.sh hook installation files (probably `scripts/install-hooks.sh` + `.git/hooks/pre-commit` template), clippy-target files in `crates/aiplus-core/` + non-Wave-2-B `crates/aiplus-cli/` files (NOT coordinator.rs auto-summon section), the new Skill file in aiplus-agent-team
- This session must NOT modify: `crates/aiplus-cli/src/agent/coordinator.rs` (Wave 2-B owns), `crates/aiplus-cli/src/agent/commands.rs` (shared with Wave 2-C — only ADD/MODIFY the existing `Doctor` subcommand to support `--quiet`; do NOT touch new `TokenCost` subcommand area), CONTRACT.md (FROZEN), adapter code, calibration fixture
- **commands.rs collision avoidance**: Wave 2-C is appending a NEW subcommand variant. You're adding a NEW flag to the EXISTING Doctor subcommand. Different parts of the file. Git will auto-merge cleanly if both stay in their lane.

## Scope — 6 polish items

### Sub-feature 1 (D1): AdapterResult return-value plumbing

**Current state**: `route_known_role` returns `Result<()>`. Callers see success/failure but cannot access primary's `final_text` / `tool_calls` / `usage_tokens`.

**Target state**: `route_known_role` returns `Result<AdapterResult>`. Caller chain (`run_adaptive_route`, `coordinator_batch`, `route_batch`, etc.) updated to thread the result back.

**Care points**:
- `AdapterResult` struct may already exist somewhere (check `aiplus-agent-team` / `crates/aiplus-core/`); if so reuse, don't redefine
- Adapter trace files already contain AdapterResult JSON; the question is whether to expose the same struct in-process or define a thin Rust mirror
- Backward compatible at the dispatch-log layer (don't change emitted format)
- Test cascade — many tests assert on `Result<()>`-style returns; updating signatures will hit many test files

**Why first**: largest blast radius. Once D1 lands, D2-D6 build on top.

### Sub-feature 2 (D2): dispatch-log row schemaVersion

**Target**: every JSONL row in `.aiplus/agents/dispatch-log.jsonl` (including legacy event types: per-role dispatch, coordinator_decision, tamper-evident chained rows) carries `"schemaVersion": "0.4.0"`.

(Why 0.4.0? Coordinator decision events have evolved across v0.3.0 (initial) → v0.3.1 (forced_by_risk + auto_summoned + ttl_expired) → SEC-1 (prev_hash + entry_hash + genesis). Bumping the row-level schema to 0.4.0 signals "this is post-Polish-1 row format". CEO finalizes the exact version string in Phase 1.)

**Care points**:
- Field is purely additive; old parsers ignore unknown fields
- Apply consistently across ALL row writers in route.rs (multiple events)
- Don't retroactively rewrite old log rows; only new writes carry the field

### Sub-feature 3 (D3): `aiplus doctor --quiet`

Add `--quiet` / `-q` flag to existing `doctor` subcommand. When set:
- Suppress all INFO lines (cache age, signing config, auditor configured [now removed], chain status, etc.)
- Show only WARN+ and FAIL
- Final `DOCTOR_STATUS=...` line still shown

Implementation: thread `quiet: bool` into doctor.rs's print functions. Cleanest: a small `output_level: enum` with Verbose / Default / Quiet variants.

### Sub-feature 4 (D4): install.sh / Cargo parity pre-commit hook

**Hook logic**:
```bash
#!/bin/bash
# .git/hooks/pre-commit
# Refuses commit if Cargo.toml version changes but install.sh fallback doesn't match.
cargo_version=$(grep -m1 '^version = ' crates/aiplus-cli/Cargo.toml | sed 's/.*"\(.*\)"/\1/')
install_sh_version=$(grep -m1 'VERSION="${VERSION:-v' install.sh | sed 's/.*v\([^"]*\)".*/\1/')
if [ "$cargo_version" != "$install_sh_version" ]; then
  echo "ERROR: aiplus-cli Cargo.toml version ($cargo_version) does not match install.sh fallback (v$install_sh_version)"
  echo "Either bump install.sh fallback OR revert Cargo.toml version change."
  exit 1
fi
```

**Installation**: `scripts/install-hooks.sh` that copies hook into `.git/hooks/`. Document in README.

### Sub-feature 5 (D5): clippy lint debt cleanup

**Run** `cargo clippy --workspace --all-targets -- -D warnings` from a clean state to enumerate exact warnings. Fix file-by-file:
- `crates/aiplus-core/` — likely most of the debt
- Older `crates/aiplus-cli/src/*` files (pre-v0.3 code)

**Rules**:
- Fix the lint (don't silence with `#[allow]` unless the lint is wrong about correctness)
- If a fix changes semantics, STOP and ping Advisor (lint cleanup should be pure cosmetic)
- Test count must not change from clippy fixes alone

### Sub-feature 6 (D6): Briefing template Skill

**Location**: `~/Projects/AiPlus/aiplus-agent-team/skills/aiplus-ceo-briefing.md`

**Content**: Extract the recurring structure from prior 8 CEO briefings:
- Worktree isolation block (template with `<work-name>` placeholder)
- Shared-file ownership template
- Phase 1/2/3 structure
- STOP rule + retry-once gate boilerplate
- Scope fence template (IN / OUT / GRAY sections)
- Deliverables list pattern
- Time ceiling format
- Handoff endpoint template (IMPL_VERDICT variants)
- Owner gates standard block
- Dependencies template

**Format**: Markdown with placeholder annotations like `<GOAL-NAME>`, `<work-name>`, `<owned-files>`, `<forbidden-files>`. Future Advisor copies this file and fills in placeholders.

Plus a top-level "How to use" paragraph: "Read this Skill at the start of any new CEO briefing draft. Copy template, fill placeholders, customize per goal."

## Phase structure

### Phase 1 — Write `docs/proposals/polish-1-impl-notes.md` BEFORE coding

Required sections (one per sub-feature):
1. AdapterResult plumbing: which call sites change, struct location, test cascade plan
2. schemaVersion value choice + write sites
3. --quiet output rules + flag wiring
4. pre-commit hook script + installation pattern
5. Clippy inventory (run clippy, capture warnings, classify into groups)
6. Skill content outline (which sections, what placeholders)
7. CHANGELOG ## 0.6.5 draft text (covers all 6 sub-features for v0.6.5 ship)
8. Test plan

### Phase 2 — Implement (order matters)

1. **D1 first** — AdapterResult plumbing (function signature change ripples; do this before other route.rs edits)
2. **D2** — schemaVersion field (small additive)
3. **D3** — doctor --quiet (small)
4. **D4** — pre-commit hook script (separate file, no Rust dependency)
5. **D5** — clippy cleanup (file-by-file; intersperse with smaller items if helpful)
6. **D6** — Skill file write

After each sub-feature: `cargo test --workspace` + `cargo clippy --workspace --all-targets -- -D warnings` (after D5 lands; before D5 you'll have known warnings).

### Phase 3 — Evidence

- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` PASS (clean)
- Live: pre-commit hook test — manually bump Cargo.toml version + try commit → hook refuses
- Live: `aiplus doctor --quiet` shows no INFO lines
- Live: tail dispatch-log → `schemaVersion` field present
- Skill file exists at `aiplus-agent-team/skills/aiplus-ceo-briefing.md`
- Trailer in impl-notes

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** (test failure post-change) or `_SCOPE-BREACH_` (touched a not-owned file, especially coordinator.rs auto-summon or commands.rs TokenCost area)
- **Do NOT STOP on `_IMPL-BUG_`** — fix and continue
- **Special**: clippy fix that changes runtime semantics → STOP + ping Advisor (must be lint-only, never behavior change)

### Retry-once gate (mandatory)

Standard.

## Conformance / verification

- All 6 D-goals met
- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Skill file structurally sensible (Advisor reviews at Phase C)
- pre-commit hook does what it claims (live tested)

## Scope fence

- **IN**: route.rs, doctor.rs, install.sh hook script, clippy target files in aiplus-core + old aiplus-cli paths, new Skill file in aiplus-agent-team (cross-repo single-file write)
- **OUT**: coordinator.rs, commands.rs (except adding `--quiet` to existing Doctor subcommand), new crate aiplus-token-cost, calibration fixture, CONTRACT.md, adapter code, Cargo.toml version bump, CHANGELOG actual
- **GRAY**: small unrelated lint fixes adjacent to your changes are OK

## Deliverables (8)

1. `docs/proposals/polish-1-impl-notes.md` — Phase 1 design + Phase 3 evidence (in aiplus-public)
2. `route.rs` updates (D1 + D2)
3. `doctor.rs` updates (D3)
4. Pre-commit hook script + install script (D4)
5. Clippy fixes (D5, multiple files)
6. `aiplus-agent-team/skills/aiplus-ceo-briefing.md` (D6, cross-repo)
7. New tests for any new functionality (especially D1 plumbing tests)
8. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+5d**.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + per-sub-feature status + test/clippy counts. Advisor coordinates Phase C with Wave 2-B and Wave 2-C as single v0.6.5 ship.

## Owner gates

Standard. BWS: wrap any LLM-call test (probably not needed here — polish work doesn't dispatch).

## Dependencies

- Wave 1 (G-AT-AUDITOR-REMOVE-1) merged to main
- Post-Wave-1 route.rs as baseline
- 8 prior briefings in `aiplus-agent-team/docs/decisions/` as source for D6 Skill

---

— Advisor, 2026-05-19, Wave 2-D. Day-0 narrow STOP + retry-once.
