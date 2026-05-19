# Goal G-AT-USERLAND-BUGFIX-1 — Retrospective (COMPLETE)

**Status**: **DONE** — All 4 userland-found bugs fixed, shipped in v0.6.8 on 2026-05-19.
**Original goal**: `docs/proposals/goal-G-AT-USERLAND-BUGFIX-1.md`
**Closing date**: 2026-05-19 (opened same day after userland-test surfaced bugs)
**Duration**: ~0.5 work day CEO + Phase C (vs. 2-day budget; ~75% under).

---

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | Goal doc (with Bug 4 amendment) + CEO briefing | `fb9c4bf` → `3f84556` | aiplus-agent-team |
| 2 | Stage 6 findings memo (forensic, post-test) | `448066e` | aiplus-agent-team |
| 3 | 4-bug fix commit + impl-notes + 5 new userland regression tests | `d5b97e4` | aiplus-public |
| 4 | Phase C: bump 0.6.7 → 0.6.8 + CHANGELOG + install.sh parity + this retrospective | post-`d5b97e4` (this commit) | both |

**Total**: 925 lines impl across 7 files + ~470 lines spec/notes/findings.

## Bug-by-bug

| Bug | Source | Severity | Fix | Live-verified |
|---|---|---|---|---|
| 1 | mcp-register naming inconsistency | HIGH | accept `claude-code` as alias for `claude` | ✅ exact failure command now passes |
| 2 | `doctor --quiet` claimed but absent | HIGH | add clap field + suppression logic | ✅ help lists flag, output 1 line |
| 3 | LLM intent auto-summon silently empty | MEDIUM | OpenAI fallback + visible warnings | ✅ `Auto-summoned experts: [security-reviewer]` fires |
| 4 | mcp-register ignored `CODEX_HOME` | HIGH | honor env + new `--config-dir` flag | ✅ writes to `/tmp/bug4-verify-home/config.toml` |

## D-goal status

- D1-D4 (Bugs 1-4): all ✓ fixed and live-verified
- All ACs met
- 5 new regression tests cover the failure modes; total tests 574 → 579
- `cargo clippy --workspace --all-targets -- -D warnings` clean

## What went well

1. **Userland testing methodology validated empirically.** 574 internal tests passed in v0.6.7; ~30 min of approximating-real-user testing found 4 product bugs. The synthetic test suite missed all of them. This pattern is now codified and will be used for all future Phase C verifications.

2. **Each bug fix live-verified before ratification.** Advisor ran the exact failure-mode command from the userland test for each of the 4 bugs against the post-fix binary. All four passed. This is in direct contrast to prior Phase C work where Advisor trusted CEO's reported PASS without independently running the user-facing command — which is how Bugs 2 and 3 escaped detection in their original shipping goals (v0.6.5).

3. **CEO Bug 3 fix exceeded spec by adding visible warnings.** Spec said "either fix the auto-summon OR add visible warning". CEO did both: now the classifier tries both providers AND surfaces skipped/failed reasons. Future debug is easier.

4. **Third consecutive pure PASS verdict** (no PASS_WITH_DEVIATIONS): CEO followed the briefing template + scope discipline + day-0 narrow STOP cleanly. No forbidden files touched; investigation found Bug 3 was wiring not structural; CEO didn't try to rebuild the classifier.

5. **Bug 4 was discovered mid-test by Advisor's own mistake.** Setting up Stage 6 isolation, Advisor exported `CODEX_HOME` expecting mcp-register to honor it. It didn't, and silently wrote to Owner's real `~/.codex/config.toml`. Self-corrected the side effect (sed-replaced the binary path, kept backup), and elevated the gap to a fixable bug in the goal. This is the right pattern: own the mistake, recover, codify the prevention.

## What went wrong

1. **Advisor Phase C work for v0.6.5 was incomplete.** The source of Bugs 2 and 3 was that Advisor's Phase C ratification for G-AT-POLISH-1 and G-AT-AUTOSUMMON-INTENT-1 trusted CEO's reported tests without independently running the user-facing commands. CEO's tests verified *something* (different test fixtures), but those tests didn't catch that:
   - `commands.rs` Doctor variant never got the `--quiet` field added
   - Coordinator's auto-summon path failed silently when only OpenAI key was present
   
   This whole bugfix goal was retroactive accountability. **The lesson is encoded going forward**: every new flag/command/feature in Phase C MUST be exercised via a real CLI invocation with the exact form a user would type, not just `cargo test` PASS.

2. **Briefing didn't get Bug 4 amendment after I added Bug 4 to the goal doc.** I patched goal-G-AT-USERLAND-BUGFIX-1.md to include Bug 4 but rationalized "briefing not amended (Bug 4 fix is small wiring change; CEO can infer scope from goal doc)". This worked here, but the right pattern is to amend both. Future amendments should hit both files.

## Architectural lessons codified

1. **Userland-test loop is the new Phase C baseline.** For every shipped feature: (a) install fresh in `/tmp/aiplus-userland-test/`, (b) run the exact user-facing command from the spec, (c) verify output matches expected. Time cost: ~5-15 min per Phase C. Catches what `cargo test` cannot.

2. **MCP discovery layer is necessary, not nice-to-have.** Validated by Stage 6 real-codex tests (separate forensic memo `stage-6-userland-mcp-discovery-findings.md`): even with all 14 MCP tools registered + good descriptions, codex on natural prompts prefers shell + internal-knowledge over MCP tools. Next goal: G-AT-AGENT-AUTOFLOW-DISCOVERY-1 (Option B SKILL.md + Option C install-time CLAUDE.md preamble).

3. **Provider-fallback pattern for LLM-call features.** Bug 3 root cause was hardcoded preferred-provider with silent fallback-to-empty. Fix: try-each-then-skip-with-warning. This pattern applies to G2 semantic dispatch gate, future intent classifiers, any other LLM-aided feature. Future spec should explicitly require "what does it do when the preferred provider isn't available?"

4. **CEO accountability vs Advisor accountability.** Bugs 2 + 3 were CEO's incomplete implementation but ALSO Advisor's incomplete ratification. Pure-PASS verdicts from CEO are inputs to Phase C decision, not the whole decision. Advisor's userland-test loop is the gate.

## Open follow-ups

- **G-AT-AGENT-AUTOFLOW-DISCOVERY-1** (Option B + Option C) — empirically necessary per Stage 6 findings. Next goal.
- **Cleanup `/tmp/aiplus-userland-test/`** — Owner can `rm -rf` whenever. Preserved for now as forensic.
- **`~/.codex/config.toml` Bug 4 side-effect repair** — Owner has the Advisor-patched entry pointing at real binary; backup at `~/.codex/config.toml.preuserland-test`. Owner can leave, re-register, or restore.
- **Future flag-add goals**: amend BOTH goal doc and briefing when scope changes mid-goal.

## Next goal

Per Owner's priority decision (2026-05-19): bugfix first (this goal) → then Discovery.

Advisor next-action: draft G-AT-AGENT-AUTOFLOW-DISCOVERY-1 covering Option B (SKILL.md per runtime) + Option C (`aiplus install` writes project-level `CLAUDE.md` / `AGENTS.md` preamble) + a Stage-6-natural-prompt-rerun acceptance criterion.

---

— Advisor, 2026-05-19, G-AT-USERLAND-BUGFIX-1 closed. 4 bugs fixed + ratification methodology upgraded.
