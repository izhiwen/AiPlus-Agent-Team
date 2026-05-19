# Autoflow Validate — CEO Session C Briefing

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session C (Wave 2, SERIAL after Wave 1 ratified — do NOT start this until Owner says "Wave 1 ratified, start Validate")
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AUTOFLOW-VALIDATE-1.md`
- Post-Wave-1 baseline: `aiplus-public@<TBD>` (will be the v0.6.10 ship commit)
- SKILL.md × 3 + preamble + MCP descriptions: in `aiplus-public@<TBD>`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation

- Worktree:  `~/Projects/AiPlus/aiplus-public.autoflow-validate/`
- Branch:    `feat/autoflow-validate`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.autoflow-validate -b feat/autoflow-validate`

Files YOU OWN: `crates/aiplus-cli/tests/autoflow_coverage_matrix.rs` (new), `docs/proposals/autoflow-validate-1-impl-notes.md`, `docs/proposals/autoflow-validate-1-coverage-matrix.md`.

Files NOT to touch: any source code beyond the new test file; SKILL.md (Wave 1 owns); CONTRACT, scoring, calibration, adapter, version, CHANGELOG actual, install.sh.

## Scope — multi-runtime test harness + coverage matrix

This goal is **mostly automation + observation**, not feature development. Two phases:

### Phase A (CEO, ~1-2 days): build the test harness

1. Test design pattern:
   ```rust
   #[test]
   fn discovery_<intent>_<runtime>() {
       let result = run_natural_prompt(
           "<prompt>",
           Runtime::Codex,  // or OpenCode
       );
       assert!(result.tools_called.contains("<expected_tool>"));
   }
   ```
2. Test fixtures:
   - One prompt per MCP tool intent (14 tools × ~1 prompt = 14 cases)
   - One prompt per non-MCP feature (6 categories × ~1 prompt = 6 cases)
   - 20 test cases minimum, run × 2 runtimes (codex + opencode) = 40 data points
3. Test runner: subprocess-spawn codex/opencode in non-interactive mode with isolated config, prompt, parse stdout/stderr for tool-call markers
4. Output: coverage matrix populated programmatically into `docs/proposals/autoflow-validate-1-coverage-matrix.md`

### Phase B (Owner dogfood, 1 week — no CEO work)

Owner uses aiplus naturally:
- Open Claude Code / Codex / OpenCode sessions for real work
- Ask natural-language questions
- Use dispatch features
- Notice friction (agent didn't do expected thing) or wins (agent helpfully auto-routed)
- Record in `docs/proposals/dogfood-notes-2026-05-XX.md` (Owner-authored)

CEO doesn't work during Phase B.

### Phase C (Advisor, ~half day, after dogfood)

Advisor collates dogfood notes + harness results into program retrospective. May trigger v3 goal if friction patterns emerge.

## STOP rule

- `_REGRESSION_` → STOP
- Test harness can't reliably isolate runtime configs → STOP + ping Advisor
- Phase B is Owner-driven, not subject to STOP gates

## Deliverables

1. `tests/autoflow_coverage_matrix.rs`
2. `docs/proposals/autoflow-validate-1-coverage-matrix.md` populated by harness
3. `docs/proposals/autoflow-validate-1-impl-notes.md`
4. CHANGELOG draft text (this goal MAY not bump version — validation goals often don't)

## Time ceiling

CEO Phase A: T+2d hard ceiling. Owner Phase B: 1 calendar week. Advisor Phase C: T+0.5d after dogfood.

## Handoff endpoint

CEO reports `IMPL_VERDICT={...}` after Phase A. Owner reports dogfood notes when ready. Advisor closes both with retrospective.

## Owner gates

Standard. Phase A test harness uses LLM (codex/opencode runs) — cost estimate $5-15 across 40 runs.

## Dependencies

- Wave 1 Coverage + Multi-turn shipped to v0.6.10
- Owner availability for 1-week dogfood

---

— Advisor, 2026-05-19. Wave 2 serial goal C.
