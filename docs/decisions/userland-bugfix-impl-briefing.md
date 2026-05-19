# Userland Bugfix — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-USERLAND-BUGFIX-1.md`
- Userland sandbox preserved at `/tmp/aiplus-userland-test/` (Advisor can leave it for your reproduction)
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.userland-bugfix/`
- Branch:    `feat/userland-bugfix`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.userland-bugfix -b feat/userland-bugfix`

Shared-file ownership:
- This session OWNS: `crates/aiplus-cli/src/main.rs` (mcp_register parser), `crates/aiplus-cli/src/agent/doctor.rs` (--quiet flag wiring), `crates/aiplus-cli/src/agent/commands.rs` (Doctor subcommand variant adding --quiet field), `crates/aiplus-cli/src/agent/coordinator.rs` (Bug 3 — auto-summon intent classifier, **only the bug fix, NOT scoring rubric**), new tests
- This session must NOT modify: CONTRACT.md (FROZEN), `crates/aiplus-token-cost/` (subtree mirror), `tests/fixtures/coordinator_calibration.toml` existing entries, adapter code, Cargo.toml version field, CHANGELOG actual, install.sh
- **Coordinator scoring rubric stays locked** — Bug 3 fix touches ONLY the auto-summon LLM intent path, NOT the complexity/risk scoring functions
- If Bug 3 root cause is structural (API design needs redesign), STOP and ping Advisor

## Three bugs (in order of investigation difficulty: easiest first)

### Bug 1: mcp-register alias

Current state: clap arg parser in `command_mcp_register` (around `crates/aiplus-cli/src/main.rs:18147`) restricts `--runtime` to `codex|claude|opencode`. Needed: also accept `claude-code` as alias for `claude`.

**Fix**: in the `--runtime` value parsing, normalize `claude-code` → `claude` before matching. Update `--help` to show both forms.

**Test**: 
- `aiplus mcp-register --runtime claude-code --dry-run` succeeds with same output as `--runtime claude`
- `aiplus mcp-register --runtime claude --dry-run` still works (backward compat)
- `aiplus mcp-register --runtime invalid` still errors

### Bug 2: doctor --quiet flag

Current state: `aiplus doctor --help` lists `--fix`, `--check-keyring` — no `--quiet`. The flag was claimed shipped in v0.6.5 G-AT-POLISH-1 D3 but the clap definition for the `Doctor` subcommand was never updated.

**Fix**:
1. Locate the `Doctor` enum variant in clap structure (`crates/aiplus-cli/src/agent/commands.rs` based on diff in v0.6.5 polish-1 commit `0ed8b30`). Add `quiet: bool` field with `#[arg(long, short)]`.
2. Thread the bool through to `agent::doctor::run()` (`crates/aiplus-cli/src/agent/doctor.rs`).
3. In `doctor.rs`, define a small `OutputLevel { Verbose, Default, Quiet }` enum (or just `quiet: bool`). When `quiet`, suppress lines that start with `INFO ` and just print final `DOCTOR_STATUS=...` plus any WARN/FAIL.

**Test**: 
- `aiplus doctor --quiet` runs without error
- Output line count substantially less than `aiplus doctor` (without flag)
- No line starts with "INFO" prefix
- Final summary line still present (`DOCTOR_STATUS=PASS` or similar)
- `aiplus doctor --help` shows `--quiet` flag

**Verify CEO's prior claim**: pre-fix, search for `--quiet` references in source — if it exists somewhere but is dead code, mention in impl-notes; could indicate the bug is "wiring missing", not "feature missing". Update tests accordingly.

### Bug 3: LLM intent auto-summon doesn't fire (investigation-heavy)

Current state: `coordinator.rs` autosummon section reads `[autosummon] intent_hint` from role TOML and (per v0.6.5 G-AT-AUTOSUMMON-INTENT-1 spec) calls a lightweight LLM to classify whether task matches intent. In real userland test with `OPENAI_API_KEY` in env and `intent_hint` clearly matching task, `auto_summoned` field returns `[]`.

**Investigation steps (Phase 1)**:
1. Locate the auto-summon code path in `coordinator.rs`. Find the function that decides whether a role auto-summons (likely `auto_summon` or `match_intent` or similar).
2. Find where the LLM call is supposed to be made. Is it actually wired? Or returns false unconditionally?
3. If wired: which model? Which provider (Anthropic Haiku per spec)? Which API key env var does it read?
4. Is there any logging/error swallow that would hide a failure?
5. Reproduce: add temporary `eprintln!` or use `tracing` if available — run `aiplus agent route --score-only "实现支付接口"` and see what code paths actually execute.

**Phase 2 fix (depends on root cause; options ordered by likelihood)**:
- **(a) LLM call wired but silently fails**: surface failure. Add `coordinator_decision` event field like `"intent_classifier_status": "skipped|failed|ok"` with error message in `warnings` array. User sees the failure.
- **(b) Provider not configured / key missing**: read either `ANTHROPIC_API_KEY` or `OPENAI_API_KEY`; pick whichever is available; fall back with clear warning if neither.
- **(c) LLM call not wired (returns false unconditionally)**: wire it. Use existing G2 semantic gate's LLM-call helper. Cache results.
- **(d) Structural problem** (intent classifier API doesn't actually exist; was never built; was stubbed and forgotten): STOP and ping Advisor. Don't try to build it from scratch in this goal.

**Test**:
- Integration test that mocks LLM call to return YES, asserts `auto_summoned` contains the role
- Integration test that simulates missing API key, asserts visible warning in output
- If LLM is real (cannot mock cleanly), use a feature-gated test that skips when no API key

## Phase structure

### Phase 1 — Write `docs/proposals/userland-bugfix-impl-notes.md` BEFORE coding

Required sections:
1. Reproduction commands for all 3 bugs (cd to fresh sandbox, install, run, observe)
2. Root cause findings for Bug 3 (file:line references; what code path runs; what's wrong)
3. Fix plan per bug
4. Test plan per bug
5. CHANGELOG draft text

### Phase 2 — Fix in order: Bug 1 → Bug 2 → Bug 3

Bug 3 is investigation-heavy; doing easy ones first ensures partial progress even if Bug 3 hits a STOP.

After each: `cargo build --bin aiplus` clean.

### Phase 3 — Re-run userland test

Manual verification (mirror what Advisor did to find the bugs):
1. `rm -rf /tmp/userland-retest && mkdir /tmp/userland-retest && cd /tmp/userland-retest`
2. Copy fresh `aiplus` binary
3. `aiplus install all --yes`
4. Run each bug's reproduction command — assert FIX
5. `cargo test --workspace` PASS
6. `cargo clippy --workspace --all-targets -- -D warnings` clean

Document outputs in evidence trailer.

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** (existing tests fail post-fix) or `_SCOPE-BREACH_` (touched forbidden file)
- **STOP on Bug 3 STRUCTURAL** — if you find intent classifier was never actually wired and would need significant new code, STOP and ping Advisor. Don't write a whole new classifier.
- **Do NOT STOP on `_IMPL-BUG_`** during dev — normal
- Retry-once gate for any test FAIL

## Conformance / verification

- All 3 bugs fixed (or Bug 3 surfaced for follow-up if STRUCTURAL)
- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Userland re-test PASS

## Scope fence

- **IN**: main.rs mcp_register parser, commands.rs Doctor variant, doctor.rs output suppression, coordinator.rs auto-summon path, new tests
- **OUT**: everything else (CONTRACT.md, adapter, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, install.sh, CHANGELOG actual)
- **GRAY**: small comment improvements adjacent to your fixes

## Deliverables (5)

1. `docs/proposals/userland-bugfix-impl-notes.md` — Phase 1 root cause + Phase 3 evidence
2. Bug 1 fix
3. Bug 2 fix
4. Bug 3 fix or STOP report
5. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+1.5d** (1.5 days). Likely ~1 day if Bug 3 is simple wiring.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + per-bug status (`fixed` | `STOP-structural` | other). Advisor handles Phase C (version bump 0.6.7 → 0.6.8, CHANGELOG, install.sh parity, retrospective).

## Owner gates

Standard. BWS-backed: wrap any LLM-call test with `aiplus secret-broker run --aliases anthropic,openai -- <cmd>`.

## Dependencies

- v0.6.7 main ✓ (`aiplus-public@3b0e29a`)
- Original v0.6.5 bug-source commits: G-AT-POLISH-1 `0ed8b30` (Bug 2 source), G-AT-AUTOSUMMON-INTENT-1 `642c1e7` (Bug 3 source)

---

— Advisor, 2026-05-19, post-userland-test bugfix. Day-0 narrow STOP + retry-once.
