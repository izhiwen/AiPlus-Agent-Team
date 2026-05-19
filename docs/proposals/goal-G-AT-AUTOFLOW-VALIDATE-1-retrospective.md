# Goal G-AT-AUTOFLOW-VALIDATE-1 — Phase A Retrospective (Phase B in progress)

**Status**: **Phase A DONE**. Phase B (Owner 1-week dogfood) is in-progress / available to start.
**Original goal**: `docs/proposals/goal-G-AT-AUTOFLOW-VALIDATE-1.md`
**Phase A closing date**: 2026-05-19 (opened same day)
**Phase A duration**: ~half day CEO + Advisor Phase C (vs 2-day budget).

---

## Phase A — what shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | Goal doc + briefing (Wave 2 spec, drafted at start of 3-goal program) | `5e72f3d` | aiplus-agent-team |
| 2 | Coverage harness `tests/autoflow_coverage_matrix.rs` (884 lines) + impl-notes + matrix doc | `5e60fa9` | aiplus-public |
| 3 | Phase C: CHANGELOG Unreleased entry, this retrospective. **No version bump** (test/doc only, no production code change) | post-`5e60fa9` | both |

**Total**: 1137 lines (884 harness + 214 impl-notes + 39 matrix).

## Phase A acceptance result

**Header numbers** (claimed by CEO, verified by Advisor):
- 40 cells populated (20 prompts × 2 runtimes)
- 18 strict PASS
- 2 strict FAIL (OpenCode `mcp_route` and `cli_doctor`)
- 20 RUNTIME-LIMITATION (all 20 codex runs)

**Critical re-examination of the 2 strict FAILs**:

| Cell | Test expected | Agent did | Analysis |
|---|---|---|---|
| `mcp_route` | `agent_route` direct dispatch | `agent_route_score_only` preview first | Agent EXACTLY followed SKILL.md "Dispatch Flow" rule: "Step 1 — Preview the task: Call `agent_route_score_only` with `task=...`". Test expectation contradicted our own spec. **Should reclassify as PASS.** |
| `cli_doctor` | `aiplus doctor` CLI | `agent_doctor` MCP | Agent followed "Prefer MCP Tools Over CLI Subcommands" rule. `agent_doctor` is the MCP equivalent. Test expectation contradicted our own spec. **Should reclassify as PASS.** |

**Effective acceptance**: 20/20 OpenCode (100%) when test expectations are aligned with SKILL.md guidance.

**Codex 20/20 RUNTIME-LIMITATION root cause**: harness's isolated CODEX_HOME setup omits the `auth.json` copy that Owner's real `~/.codex/auth.json` provides. Causes 401 Unauthorized on codex websocket. Advisor's manual v0.6.9 testing worked because Advisor explicitly `cp ~/.codex/auth.json <isolated>/auth.json`. Harness should do the same OR document the requirement.

## What went well

1. **Test harness is a real engineering artifact.** 884 lines of code spawning isolated runtimes, programmatic JSON-RPC + output parsing, populating a structured matrix. Reusable for future discovery layer changes / runtime additions.
2. **CEO scope discipline maintained.** Only 3 files touched, all in new test/doc paths. No production code touched at all. Cleanest Phase A in any goal so far.
3. **Discovery layer empirically validated as ≥ 90% strict PASS in opencode** — exceeds the ≥ 70% acceptance gate even reading the 2 "FAIL" cells strictly. Reclassified, it's 100%.
4. **The "FAILs are actually PASSes per spec" insight** is exactly the kind of empirical data this goal was designed to capture. Shows the spec (SKILL.md) is internally consistent and the discovery layer enforces it.
5. **0.5-day execution vs 2-day budget** — fastest Phase A in the program (~75% under).

## What went wrong

1. **2 test expectations contradicted SKILL.md.** Not a goal-blocking issue but worth fixing in a follow-up (update test expected values to match what the spec actually wants). Documents a subtle hazard in writing validation harnesses: easy to encode "what I think the agent should do" rather than "what the spec says the agent should do". Always cross-reference SKILL.md when writing test expectations.

2. **Codex auth not handled in harness.** All 20 codex cells came back as RUNTIME-LIMITATION because the harness doesn't copy `auth.json` into the isolated CODEX_HOME. CEO documented this in the matrix; Advisor noted it in this retrospective. Future iteration: harness should accept an `--auth-source` flag or auto-detect Owner's real auth.json path.

## Architectural lessons codified

1. **"Validation goals shouldn't bump version" pattern works.** v0.6.10 binary unchanged; validation infrastructure ships under Unreleased. Release tagging happens only when binary semantics change. Cleaner CHANGELOG.

2. **Programmatic coverage matrix > ad-hoc Stage 6.** Stage 6 from earlier in this sprint was 3 prompts × 1 runtime, ad-hoc. This harness is 20 prompts × 2 runtimes, structured. Future "does discovery still work?" questions can be answered by running the harness rather than re-doing Stage 6 from scratch.

3. **Test expectations are themselves a kind of spec, and can drift from THE spec.** When SKILL.md says "agent should prefer X over Y for case Z", the validation test for case Z should expect X. Easy to forget; codify as briefing template ("test expectations MUST cite the SKILL.md line that defines the expected behavior").

## Open follow-ups (post-dogfood)

- **Harness codex auth fix** — copy `auth.json` from Owner's real `~/.codex/` into isolated `CODEX_HOME` before each codex run. Small follow-up (~half day) — could be folded into Phase B Owner dogfood notes or done as a tiny separate task.
- **2 "FAIL" cell expectations** — update to match SKILL.md (`mcp_route` should expect score_only-first; `cli_doctor` should expect MCP agent_doctor). Same follow-up scope.
- **Validate-1 v1.1 follow-up** — once codex auth handled, re-run harness for full 40-cell strict PASS coverage.

## Phase B — Owner dogfood (in progress / available to start)

Phase B is not CEO work; it's Owner using v0.6.10 naturally for ~1 week and recording friction notes. Per Wave 2 spec, Owner can start Phase B in parallel with Phase A (already done) or any time after.

Phase B notes file expected location: `docs/proposals/dogfood-notes-2026-05-XX.md` (Owner-authored when ready).

## Phase C (post-dogfood) — Advisor

After Phase B notes land, Advisor collates harness data + Owner dogfood signal into combined program retrospective and decides whether further goals are needed.

## Next action

Owner starts (or has started) Phase B dogfood. Advisor stand-by until Owner shares notes.

---

— Advisor, 2026-05-19. Phase A of Validate-1 closed.
