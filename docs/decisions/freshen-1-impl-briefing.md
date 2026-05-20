# G-AT-FRESHEN-1 — CEO Impl Briefing

**Drafted**: 2026-05-19 by Advisor
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-FRESHEN-1.md`
- Auto-memory: `advisor_briefing_stop_rules.md`, `parallel_ceo_tag_gate.md`, `mcp_freshness_before_features.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.freshen-1/`
- Branch:    `feat/freshen-1`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.freshen-1 -b feat/freshen-1`

## Scope — D1 + D2 + D3 only (D4 moved to G-AT-COST-COVERAGE-1)

### D1: mcp-register detect existing config + offer update

In `crates/aiplus-cli/src/mcp_register.rs` (or wherever `aiplus mcp-register` lives), before writing the config:

1. Read existing config (codex: `~/.codex/config.toml` `[mcp_servers.aiplus]`, claude-code: similar, opencode: similar)
2. If `command` field exists and points to a binary path that's NOT the current `aiplus` binary (use `std::env::current_exe()` or `which aiplus`), print:
   ```
   MCP_REGISTER_EXISTING=path=/old/path/aiplus
   MCP_REGISTER_CURRENT=path=/current/aiplus
   MCP_REGISTER_PROMPT=Update to current? [y/N]
   ```
3. If TTY: wait for input. If NOT TTY (CI / hook): default to update with notice line.
4. If update accepted: replace path, write, print `MCP_REGISTER_UPDATED=true`.
5. If existing matches current: skip prompt, print `MCP_REGISTER_FRESH=true`.

### D2: new `aiplus persona refresh [--dry-run]` command

In `crates/aiplus-cli/src/persona/refresh.rs` (new file):

1. Walk `.aiplus/agents/personas/*.md` in current project
2. For each persona file:
   - Find its canonical template under `assets/aiplus-agent-team/...` (by file name match)
   - Compare the "AiPlus Tool Discovery" block / SKILL preamble section:
     - If absent in current persona → "DRIFT_NEW_PREAMBLE"
     - If present but differs from canonical → "DRIFT_OUTDATED_PREAMBLE"
     - If matches canonical → "FRESH"
3. With `--dry-run`: print diff per persona, no writes
4. Without `--dry-run`: prompt Owner per persona (or `--yes` to bulk-accept), then replace ONLY the SKILL preamble block (preserve user-edited persona body after `# CEO — AiPlus Agent Team v0.1` or equivalent header)
5. Same logic for `.aiplus/AGENTS.aiplus.md` preamble block

Identification heuristic for the preamble block:
- Start: first line starting with `# AiPlus Tool Discovery` (the prepended hotfix block I wrote today)
- End: first occurrence of `---\n\n# CEO —` or equivalent separator (next persona section)
- If no `# AiPlus Tool Discovery` header → block absent, prepend full canonical preamble before existing content

### D3: `aiplus doctor` 2 new WARN checks

In `crates/aiplus-cli/src/doctor.rs`:

1. **MCP server binary path mismatch**:
   - For each runtime (codex / claude-code / opencode) where config exists:
     - Read `[mcp_servers.aiplus].command` (or equivalent)
     - Compare to `std::env::current_exe()` or `which aiplus`
     - If different: emit `DOCTOR_WARN_MCP_BINARY_STALE=runtime=<r> configured=<path> current=<path>`
2. **Project persona/preamble drift** (only if current dir is an aiplus project):
   - For each persona in `.aiplus/agents/personas/*.md`:
     - Compute drift status per D2 logic
     - If `DRIFT_NEW_PREAMBLE` or `DRIFT_OUTDATED_PREAMBLE`: emit `DOCTOR_WARN_PERSONA_DRIFT=file=<path>`
   - Same for `.aiplus/AGENTS.aiplus.md`
3. Both WARN (not FAIL): user may intentionally vendor old binary or customize persona. Print remediation:
   ```
   DOCTOR_WARN: persona drift detected. Run `aiplus persona refresh [--dry-run]` to inspect.
   DOCTOR_WARN: MCP binary stale. Run `aiplus mcp-register --runtime <r>` to update.
   ```

## STOP rules (per auto-memory `parallel_ceo_tag_gate.md`)

- `_REGRESSION_` → STOP (existing tests fail)
- `_SCOPE-BREACH_` → STOP (touched forbidden file)
- `_TAG-BREACH_` → STOP (NEVER tag, NEVER bump Cargo.toml, NEVER push tags — Advisor handles at Phase C)
- Retry-once gate standard

**Forbidden files**:
- CONTRACT.md (frozen at 6cd4e9b)
- Cargo.toml version bump
- CHANGELOG.md actual entry
- install.sh
- release.yml
- crates/aiplus-token-cost/
- scoring rubric, calibration fixture
- ALL `crates/aiplus-cli/src/agent/cost/` (G-AT-COST-COVERAGE-1 owns this)
- `crates/aiplus-cli/src/mcp_server.rs` `agent_token_cost` handler body + description (G-AT-COST-COVERAGE-1 owns)
- install.sh (G-AT-INSTALL-SMOKE-1 owns)

## Test plan (Phase 3 evidence)

- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Live test in sandbox:
  - **mcp-register update**: set up fake `~/.codex/config.toml` with old path, run `aiplus mcp-register --runtime codex`, verify it prompts + updates
  - **persona refresh dry-run**: in a fixture project with stale persona, run `aiplus persona refresh --dry-run`, verify diff output
  - **persona refresh apply**: same fixture, run `aiplus persona refresh --yes`, verify only preamble block was replaced
  - **doctor WARN**: set up project with stale persona + stale MCP binary path, run `aiplus doctor`, verify 2 WARN lines emitted

## Deliverables (5)

1. `docs/proposals/freshen-1-impl-notes.md` — Phase 1 + Phase 3 evidence
2. `crates/aiplus-cli/src/mcp_register.rs` D1 extension
3. `crates/aiplus-cli/src/persona/refresh.rs` (new) + module wiring in main.rs / lib.rs / clap
4. `crates/aiplus-cli/src/doctor.rs` 2 WARN checks
5. Tests for all 3 sub-deliverables

## Owner gates

Standard. No global config edits (`~/.codex/`, `~/.aiplus/`). Sandbox tests use temp HOME.

## Handoff

Report IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}. **NO TAG. NO VERSION BUMP. NO PUSH TO ORIGIN MAIN.** Branch push (feat/freshen-1) is OK for PR review. Advisor Phase C: independent re-test + handle bump + CHANGELOG + tag.

## Time ceiling

Hard cap T+2d (1-2 day estimate).

---

— Advisor, 2026-05-19. Parallel-safe with INSTALL-SMOKE-1 + COST-COVERAGE-1.
