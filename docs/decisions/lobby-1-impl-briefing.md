# Lobby UX — CEO Impl Briefing

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-LOBBY-1.md`
- Reference UX (AEL): AEL Advisor's transcript pasted into this conversation by Owner. The mock-up flow at the top of the goal doc is the spec.
- Existing primitives in main.rs: `aiplus install all`, `aiplus mcp-register`, `aiplus agent talk` (verify in Phase 1)
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.lobby/`
- Branch:    `feat/lobby`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.lobby -b feat/lobby`

## Scope — lobby UX

### Phase 1 — Investigation + design (~0.5 day)

Required impl-notes sections:
1. Where in `main.rs` to add bare-aiplus dispatch (probably in clap's "no subcommand" branch; confirm the exact code path)
2. Existing `aiplus agent talk` interface: does it accept `--runtime <r>`? If not, decide: extend `agent talk` OR have lobby call lower-level adapter spawn directly. Document the decision.
3. Runtime detection logic: how to test if `codex`, `claude`, `opencode` CLIs are on PATH. Use `which::which` crate or `std::process::Command::new(...).spawn().is_ok()` check.
4. Role list source: read from `.aiplus/agents/*.toml` or from active team's snapshot. Phase 1 confirms.
5. Role descriptions: one-line each. Use `display_name` + `voice` or similar TOML field, OR hardcode a short description per role.
6. Partial-match implementation (prefix match unique-completes; fuzzy match if not unique).
7. Default role on empty Enter: **ceo** (NOT pi — pi is AEL terminology, aiplus has no pi role).
8. Test plan: interactive flow can't easily be unit-tested; use stdin redirection in integration tests; cover happy path + edge cases (no .aiplus/ → install; no runtime CLI → fail gracefully; partial role match disambiguation).
9. CHANGELOG draft text for v0.6.11.

### Phase 2 — Implementation (~0.5-1 day)

Suggested order:
1. New `lobby` module skeleton + clap dispatch for bare `aiplus`
2. Auto-install path (call `aiplus install all --yes` programmatically OR via re-invocation; document the choice)
3. Runtime detection
4. Role picker (with empty-Enter default + partial match)
5. Runtime picker (skip-if-single + partial match)
6. Spawn dispatch
7. Tests

### Phase 3 — Evidence (~0.5 day + manual)

- All tests green
- Run `aiplus` in fresh `/tmp/lobby-test/` and capture transcript matching the goal doc's mock-up
- Verify single-runtime skip works (e.g., uninstall opencode CLI temporarily, observe lobby skips runtime prompt)
- Verify ceo default on Enter
- Verify existing `aiplus <subcommand>` paths still work (regression)

## Specific UX spec

### Greeting (when `.aiplus/` exists)

```
✦ aiplus: project is set up.

✦ Who do you want to talk to?

  Core team (current active team: <team-name>):
    advisor / 顾问           反思、识别策略、框架
    ceo / 总司令             全局协调（默认）
    architect / 架构师       系统设计、技术选型
    pm / 项目经理            派单、跟进
    engineer-a / 工程师 A    实现
    engineer-b / 工程师 B    实现（并行）
    reviewer / 评审员        代码 review
    qa / 测试员              测试

  Experts (auto-summon by intent; you can also explicitly pick):
    security-reviewer / 安全审查
    tech-writer / 技术写手
    ai-integration-specialist / AI 集成专家
    ... (more)

> [Enter for ceo, or type role name / partial match]
```

### Greeting (when `.aiplus/` missing — auto-install)

```
✦ aiplus: this project is not set up yet. Setting up...
  → installing claude-code adapter ✓
  → installing codex adapter ✓
  → installing opencode adapter (skipped: opencode CLI not on PATH)
  → registered MCP server with available runtimes
  ✓ Team ready.

[then continue with role prompt as above]
```

### Runtime prompt

```
✦ Which runtime?

  1) claude-code
  2) codex
  3) opencode

> [type 1, 2, 3, or runtime name / partial match]
```

Skip entirely if only one runtime CLI is on PATH; pick it silently and continue.

### Spawn

After role + runtime chosen, log:
```
→ <role>
→ <runtime>
[Launching <runtime> with <role> persona...]
```

Then exec/spawn the chosen runtime with the chosen role's persona loaded. Best mechanism: invoke `aiplus agent talk --runtime <runtime> <role>` (extend `talk` to accept `--runtime` if it doesn't already, per Phase 1 decision).

## Edge cases

- **No CLI of any kind on PATH** → fail with clear error: "aiplus needs at least one of claude / codex / opencode installed. See https://github.com/izhiwen/AiPlus for setup."
- **`.aiplus/` exists but no team** → run `aiplus install all --yes` to repair
- **User types unknown role** → "Role '<x>' not found. Did you mean '<y>'?" with closest match
- **User types unique partial match** → silently complete
- **User types ambiguous partial match** → list candidates, ask again
- **User Ctrl+C at any prompt** → clean exit, no spawn

## STOP rule

- `_REGRESSION_` (existing 585 tests fail; especially regression on subcommands routing) → STOP
- `_SCOPE-BREACH_` (touched forbidden file) → STOP
- `aiplus agent talk` doesn't support runtime parameter and adding it requires non-trivial refactor → STOP and ping Advisor (might mean scope creep)
- Retry-once gate standard

## Conformance / verification

- All D1-D5 met
- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Phase 3 transcript shows the mock-up flow happening for real

## Scope fence

- **IN**: new `crates/aiplus-cli/src/lobby/` module (or `lobby.rs`), main.rs bare-aiplus dispatch + minor extension of `aiplus agent talk` if needed (in scope only if minimal), new tests
- **OUT**: CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh fallback, refactoring existing subcommands beyond the minimum needed
- **GRAY**: small comment improvements in touched files

## Deliverables (4)

1. `docs/proposals/lobby-1-impl-notes.md` — Phase 1 design + Phase 3 evidence
2. New `lobby` module + main.rs dispatch
3. Tests (interactive integration tests using stdin redirection)
4. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+2d**. Likely ~1-1.5 days per pattern.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + Phase 3 transcript excerpt. Advisor handles Phase C (live re-test in fresh sandbox + bump 0.6.10 → 0.6.11 + CHANGELOG + retrospective).

## Owner gates

Standard. No LLM cost in this goal (lobby is local interactive). Phase C live-test by Advisor doesn't need LLM either (lobby flow validates without launching a real agent session).

## Dependencies

- v0.6.10 main ✓
- Existing primitives (install, mcp-register, agent talk) — Phase 1 verifies these support what lobby needs

---

— Advisor, 2026-05-19. Day-0 narrow STOP + retry-once.
