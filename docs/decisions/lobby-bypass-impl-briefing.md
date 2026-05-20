# Lobby Bypass — CEO Impl Briefing

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-LOBBY-BYPASS-1.md`
- Lobby code (v0.6.11): `crates/aiplus-cli/src/lobby/mod.rs`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.lobby-bypass/`
- Branch:    `feat/lobby-bypass`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.lobby-bypass -b feat/lobby-bypass`

## Scope — add bypass flags to lobby spawn

### Phase 1 — Investigation + design (~1 hour)

Required impl-notes:
1. **Verify per-runtime bypass flags by reading `<runtime> --help`**:
   - Codex: `--dangerously-bypass-approvals-and-sandbox` (Advisor pre-verified — confirm)
   - Claude Code: `--dangerously-skip-permissions` (Advisor unverified — confirm)
   - OpenCode: TBD (look in opencode --help; common pattern would be `--auto-approve` or `--dangerously-skip-permissions`; document what you find)
2. **Locate spawn logic**: in `crates/aiplus-cli/src/lobby/mod.rs`, find where the runtime is invoked (probably `aiplus agent talk --runtime ... <role>` or direct Command::new). Decide: extend the spawn to include bypass flags directly, OR extend `agent talk` to accept a `--bypass` parameter.
3. **CLI flag design**: `aiplus --safe` for safe-mode (bare-aiplus already exists as lobby entry; add an option). Per-clap implementation pattern.
4. **Env var**: `AIPLUS_LOBBY_SAFE=1` checked at lobby entry.
5. **Output message**: "[Launching codex with ceo persona (bypass-mode)...]" vs "(safe-mode)" — minor string change.
6. **Test plan**: integration tests can use stdin redirection + check the spawn command logged in dispatch-log OR via a fake-spawn shim.
7. **CHANGELOG draft text** (with "safety posture" note).

### Phase 2 — Implementation (~1-2 hours)

Suggested order:
1. Add `--safe` flag to bare-aiplus clap dispatch
2. Add `AIPLUS_LOBBY_SAFE` env var check in lobby entry
3. Modify lobby spawn to construct per-runtime command with bypass flags by default
4. Update output message to indicate mode
5. Tests

### Phase 3 — Evidence (~30 min)

- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Live test: in fresh sandbox `aiplus`, pick ceo, pick codex → verify codex command includes bypass flag (e.g., via `ps aux | grep codex` while lobby is launching, or by inspecting log output)
- Live test: `aiplus --safe` → verify NO bypass flag in spawn
- Live test: `AIPLUS_LOBBY_SAFE=1 aiplus` → verify NO bypass flag

## STOP rule

- `_REGRESSION_` (existing 592 tests fail; particularly lobby tests) → STOP
- `_SCOPE-BREACH_` (touched forbidden file) → STOP
- Discovered that a runtime's bypass flag has a different name than expected → continue with corrected flag, document in impl-notes
- Retry-once gate standard

## Conformance / verification

- All D1-D4 met
- Tests + clippy green
- 3 live scenarios verified (default bypass, --safe flag, env var)

## Scope fence

- **IN**: `crates/aiplus-cli/src/lobby/mod.rs` spawn logic, `crates/aiplus-cli/src/main.rs` --safe flag handling, tests, impl-notes
- **OUT**: `agent talk` subcommand changes for non-lobby callers (lobby may extend talk's interface but other callers' behavior unchanged), CONTRACT.md, adapter code, scoring, calibration, token-cost subtree, Cargo.toml version, CHANGELOG actual, install.sh
- **GRAY**: small wording updates to lobby output messages adjacent to your changes

## Deliverables (4)

1. `docs/proposals/lobby-bypass-impl-notes.md` — Phase 1 + Phase 3
2. Lobby spawn extension with per-runtime bypass flags + mode indicator
3. `--safe` flag + `AIPLUS_LOBBY_SAFE` env var handling
4. Tests covering all 3 modes (default-bypass, --safe, env-var)

## Time ceiling

**Hard ceiling: T+1d**. Likely ~half day.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + per-runtime flag confirmation + Phase 3 live-test evidence. Advisor handles Phase C (live re-test + bump 0.6.11 → 0.6.12 + CHANGELOG + retrospective).

## Owner gates

Standard. No LLM cost (lobby is local).

**Safety posture note**: bypass mode skips approvals for agent actions. CHANGELOG MUST clearly state this — bypass is intended for Owner's personal-machine personal-project use, NOT for multi-user / shared / production deployments. Phase C ratification verifies CHANGELOG language matches this guidance.

---

— Advisor, 2026-05-19. Day-0 narrow STOP + retry-once.
