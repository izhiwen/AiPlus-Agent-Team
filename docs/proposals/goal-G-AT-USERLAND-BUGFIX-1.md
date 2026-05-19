# Goal G-AT-USERLAND-BUGFIX-1 — Fix 3 bugs from userland testing

**Status**: PROPOSED — Advisor-discovered via "approximating real user" testing on 2026-05-19.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-USERLAND-BUGFIX-1`

---

## TL;DR

574 internal cargo tests passed in v0.6.7. ~30 minutes of approximating-real-user testing (install in temp dir, run each new CLI command, MCP tools/call) found **3 real product bugs** that the synthetic test suite missed. Then Stage 6 setup (isolated `CODEX_HOME` for safe live-agent testing) revealed a **4th bug** + caused an Advisor side-effect on Owner's actual `~/.codex/config.toml` (repaired in place; backup at `~/.codex/config.toml.preuserland-test`). This goal fixes all 4. Time ceiling ≤ 2 days.

## Bugs

### Bug 1 — `mcp-register --runtime claude-code` rejected

**Observed**: `aiplus mcp-register --runtime claude-code` errors with `AIPLUS_UNEXPECTED_ERROR reason=uncaught detail=unknown --runtime 'claude-code'. Valid: codex, claude, opencode.`

**Why it matters**: Every other surface in aiplus (README install instructions, `aiplus install claude-code`, CHANGELOG language) uses the runtime name `claude-code` (matching the GitHub repo). Only `mcp-register` accepts the abbreviated `claude`. Real users following the README will hit this on first MCP setup attempt.

**Fix**: Accept `claude-code` as a synonym for `claude` in `mcp-register`'s `--runtime` arg parser. Update the help text to list `claude-code` (preferred) with `claude` documented as accepted alias. (Don't break the existing `claude` form — backward compat for any user-side scripts.)

**Severity**: HIGH (gates real-user MCP setup on the most-used runtime).

### Bug 2 — `aiplus doctor --quiet` flag missing entirely

**Observed**: `aiplus doctor --quiet` returns `error: unexpected argument '--quiet' found`. `aiplus doctor --help` lists only `--fix`, `--version`, `--check-keyring`, `--help` — no `--quiet`.

**Why it matters**: The flag was specced in G-AT-POLISH-1 D3 (v0.6.5). CEO's PASS_WITH_DEVIATIONS report for that goal claimed `Live: quiet doctor 无 INFO`. CHANGELOG `## 0.6.5` documents the flag verbatim. **None of that was true** — the flag was never actually added to clap's `Doctor` subcommand definition, so the binary rejects it.

**Why this happened**: CEO's `polish_1_smoke.rs` test (per Phase B evidence) probably tested something else; Advisor's Phase C "live demo" verification was incomplete; Advisor's userland testing today caught it.

**Fix**: Add `--quiet` flag (boolean) to the `Doctor` subcommand variant in clap. Thread the flag through to `agent::doctor::run()` to suppress INFO-level lines (keep WARN+ and FAIL). Add a smoke test that explicitly invokes `aiplus doctor --quiet` and asserts no INFO line appears.

**Severity**: HIGH (CHANGELOG says it exists; users will try; binary lies).

### Bug 3 — LLM intent auto-summon doesn't fire

**Observed**: `aiplus agent route --score-only "实现支付接口"` (or "实现支付接口需要 secure auth") returns `auto_summoned: []` even though:
- `.aiplus/agents/experts/security-reviewer.toml` has `[autosummon] intent_hint = "支付、认证、敏感数据、credentials、凭据、安全漏洞或隐私相关的软件工作"` (priority 90)
- `OPENAI_API_KEY` is in env
- Task text clearly matches the intent hint by human reading

**Why it matters**: G-AT-AUTOSUMMON-INTENT-1 (v0.6.5) was the goal that replaced keyword auto-summon with LLM intent classification. Its CHANGELOG entry says "the coordinator asks a small LLM 'does this task match this intent?' per dispatched task and joins matching experts to staffing." But the LLM call is never visible (no log, no warning) and the result is empty.

**Why this happened (hypothesis)**: 
- LLM intent classifier may fail-closed silently when its preferred provider key is missing (e.g., expects `ANTHROPIC_API_KEY` for Haiku but only `OPENAI_API_KEY` is set)
- OR fail-closed when network is slow
- OR not actually wired (returns `[]` unconditionally — even worse)

**Fix path**:
1. **Investigate first** — run the intent classifier code path with verbose logging to find where the empty result comes from
2. If LLM call is failing silently → make it surface a warning (e.g., `coordinator_decision` event gets `intent_classifier_status=skipped|failed|...`)
3. If using wrong provider → respect any of `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` (whichever is configured)
4. If not wired at all → wire it
5. Add an integration test that mocks the LLM call and asserts auto-summon fires for a clear-match task

**Severity**: MEDIUM — main feature of an entire v0.6.5 goal doesn't work in real-user scenarios. But: G-AT-AGENT-AUTOFLOW-MCP-1 just shipped without depending on it, so it's not actively breaking other things.

**STOP rule for Bug 3**: if investigation reveals this is structural (API design needs redesign, not wiring), STOP and ping Advisor. Don't reinvent the intent classifier in this goal — fix or surface.

### Bug 4 — `aiplus mcp-register` ignores `CODEX_HOME` env var

**Observed**: with `CODEX_HOME=/tmp/aiplus-userland-test/codex-home` exported, `aiplus mcp-register --runtime codex --force` still writes to `/Users/steve/.codex/config.toml` (Owner's actual global config). The output line `MCP_REGISTER_CODEX=WROTE path=/Users/steve/.codex/config.toml` is the hard-coded path.

**Why it matters**: Two cumulative impacts.
1. **No way to test mcp-register safely in isolation.** Per codex's own documentation, `CODEX_HOME` is the canonical way to point codex at an alternate config dir. `aiplus mcp-register` should honor it for consistency, otherwise any setup test on a developer machine inevitably touches the real `~/.codex/`.
2. **Advisor's userland-test side-effected Owner's real codex config.** During Stage 6 setup on 2026-05-19, Advisor exported `CODEX_HOME` expecting isolation, but mcp-register wrote `[mcp_servers.aiplus] command = "/tmp/aiplus-userland-test/bin/aiplus"` into the real config. Advisor manually patched it to point at Owner's actual aiplus binary (`/Users/steve/.cargo/bin/aiplus`) and saved a backup (`~/.codex/config.toml.preuserland-test`). No data loss; one minor mutation Owner is now aware of.

Equivalent paths for the other two runtimes need verification too:
- Claude Code: respects `CLAUDE_CONFIG_DIR` env per Claude docs — does aiplus mcp-register honor it?
- OpenCode: where does it look? aiplus mcp-register writes to project-relative `opencode.json` in CWD, not user-global, so probably less of an issue.

**Fix**: in mcp-register's config-path resolution, check `CODEX_HOME` before defaulting to `~/.codex/`. Same pattern for `CLAUDE_CONFIG_DIR` (claude-code). Add a `--config-dir` flag for explicit override. Document the behavior in `--help`.

**Test**: 
- `CODEX_HOME=/tmp/x aiplus mcp-register --runtime codex --dry-run` says `WOULD_WRITE path=/tmp/x/config.toml`
- Without env, falls back to `~/.codex/config.toml` (unchanged default)
- Same isolation check for claude-code

**Severity**: HIGH — silently writes to user's real config when caller intended otherwise. Caused real Advisor incident.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/userland-bugfix-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/userland-bugfix-impl-notes.md` (Phase 1 + Phase 3) | PENDING — CEO |
| 4 | Bug 1 fix in `crates/aiplus-cli/src/main.rs` (mcp_register arg parser) + test | PENDING — CEO |
| 5 | Bug 2 fix: `--quiet` flag added to Doctor subcommand + doctor.rs threading + test | PENDING — CEO |
| 6 | Bug 3 root-cause investigation + fix (or STOP if structural) + test | PENDING — CEO |
| 7 | Bug 4 fix: `mcp-register` honors `CODEX_HOME` / `CLAUDE_CONFIG_DIR` env + `--config-dir` flag + test | PENDING — CEO |

## Phases

```
Phase 1 (CEO, ~0.5 day):
  - Reproduce all 3 bugs in worktree
  - Investigate Bug 3 root cause:
    - Grep coordinator.rs for intent classification code path
    - Find where LLM call is made (or supposed to be made)
    - Identify failure mode
  - Write impl-notes design

Phase 2 (CEO, ~0.5-1 day):
  - Bug 1: alias in mcp_register
  - Bug 2: doctor --quiet flag
  - Bug 3: root-cause-driven fix (or STOP)
  - Tests for all three

Phase 3 (CEO, ~0.5 day):
  - cargo test --workspace all green
  - Repeat userland test: run install + each command + assert all three bugs fixed
```

**Total target**: ≤ 1.5 days.

## Out of scope

- **`agent_route_score_only` SKILL.md guidance** (Option B from agent-autoflow analysis) — separate decision after dogfood
- **`aiplus install` opt-in prompts** (Option C) — same
- CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, version field, install.sh, CHANGELOG actual (Advisor handles Phase C)

## Acceptance criteria

- ✓ `aiplus mcp-register --runtime claude-code --dry-run` succeeds
- ✓ `aiplus doctor --quiet` runs, returns 0 INFO lines (only WARN+/FAIL/PASS summary)
- ✓ `aiplus doctor --help` lists `--quiet` 
- ✓ `aiplus agent route --score-only "实现支付接口"` produces non-empty `auto_summoned` array containing `security-reviewer` (when OPENAI_API_KEY or ANTHROPIC_API_KEY available) OR produces a visible warning if intent classifier cannot run
- ✓ `CODEX_HOME=/tmp/x aiplus mcp-register --runtime codex --dry-run` reports `WOULD_WRITE path=/tmp/x/config.toml` (not `~/.codex/config.toml`)
- ✓ Default behavior unchanged when env not set (writes to `~/.codex/`)
- ✓ `cargo test --workspace` PASS
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase 3 evidence includes a re-run of the relevant userland-test steps with each bug demonstrably fixed

## Dependencies

- v0.6.7 main ✓ (`aiplus-public@3b0e29a`)
- Userland test sandbox at `/tmp/aiplus-userland-test/` (preserved for CEO re-test if desired)

## Why this goal exists (forensic note)

Owner asked "可以做接近真人的使用测试吗？" on 2026-05-19. Advisor ran 5-stage userland test (install dry-run, mcp-register dry-run, all new CLI commands, live MCP, theoretical agent-decision simulation) in ~30 min and found these 3 bugs. The synthetic 574 cargo tests had passed cleanly. **This is the pattern: real-user-shape testing finds bugs unit tests cannot, even for small goals like G-AT-AGENT-AUTOFLOW-MCP-1 that returned pure PASS.**

This goal is also retroactive accountability — Advisor's Phase C ratification of G-AT-POLISH-1 (which shipped Bug 2's source) and G-AT-AUTOSUMMON-INTENT-1 (Bug 3's source) didn't actually live-test the claimed behavior. Going forward, **every Phase C must include a real CLI invocation of each new flag/feature**, not just trusting the CEO's reported test pass.

---

— Advisor, 2026-05-19, post-userland-test bugfix.
