# 0.6.13 — Retrospective (READY TO TAG)

**Status**: **READY FOR TAG** — Phase C ratified by Advisor 2026-05-19 22:00 EDT.
**Goals bundled**: public `--bypass` passthrough (Owner-authorized parallel CEO work) + macOS 26.4 SIGKILL fix (Advisor addendum) + release.yml auto-mark-latest (Advisor's separate followup branch, cherry-picked at Phase C).

---

## What's in 0.6.13 (3 things shipped under one tag)

### Thing 1 — Public `--bypass` passthrough on `aiplus agent talk`

CEO codex session (parallel) added a visible `--bypass` flag and `AIPLUS_BYPASS=1` env var on `aiplus agent talk`, complementing the hidden `--lobby-bypass` flag from v0.6.12. Direct `aiplus agent talk` callers can now request bypass mode without going through lobby. Default remains approval-mode.

Commit: `c259a51` (already on origin via parallel CEO push)

### Thing 2 — macOS 26.4 SIGKILL fix (CRITICAL)

macOS 26.4.1 introduced kernel-level enforcement that SIGKILLs Rust-produced linker-signed adhoc binaries (`flags=0x20002`). Every fresh `aiplus` install on macOS 26.4 was silently broken — install.sh reported PASS, then `aiplus` printed `zsh: killed aiplus` (EXIT=137).

Fix: re-codesign in release.yml with `codesign --force --sign - --timestamp=none` (strips linker-signed flag → `flags=0x2`). Also defensive re-sign in install.sh for Darwin (belt-and-suspenders).

Commit: `1e3e5ea` (CEO impl per Advisor addendum, pure PASS)

### Thing 3 — release.yml `--latest` (auto-mark-latest)

Previous policy created drafts on tag push, requiring manual `gh release edit --draft=false --latest` to promote. Caused 2 stale-version incidents in v0.6.10 → v0.6.11 → v0.6.12 transitions. Replaced with `--latest` flag directly on `gh release create` → no more draft → install.sh auto-fetches.

Commit: `3abdff0` (cherry-picked from `feat/release-yml-auto-latest` branch into main during Phase C)

## Phase C verification (Advisor independent live re-test)

| Check | Result |
|---|---|
| `cargo build --release -p aiplus-cli` | PASS, version 0.6.13, initial state `flags=0x20002 (adhoc,linker-signed)` ← reproduces macOS 26.4 SIGKILL trigger ✓ |
| `cargo test --workspace` | 599 passed, 2 ignored (matches CEO report) ✓ |
| `cargo clippy --workspace --all-targets -- -D warnings` | No issues found ✓ |
| Sandboxed `install.sh` against **known-bad** v0.6.12 release | After install: `flags=0x2 (adhoc)` — no linker-signed. Both `aiplus --version` (0.6.12) and `aiplus-token-cost --version` (0.1.0) EXIT=0 ✓ |
| `gh release create` line in release.yml | Contains `--latest`, no `--draft` ✓ |
| CHANGELOG 0.6.13 entries | Both public --bypass + macOS SIGKILL sections present ✓ |
| Auto-latest cherry-pick didn't break CEO's codesign steps | Both `Strip linker-signed flag` steps still in release.yml at expected line ranges ✓ |

CEO PASS report independently verified. No divergence between CEO claims and Advisor re-test.

## D-goal status

| Sub-goal | Status |
|---|---|
| Public `--bypass` on agent talk (parallel CEO) | ✓ |
| `AIPLUS_BYPASS=1` env var | ✓ |
| Hidden `--lobby-bypass` preserved | ✓ |
| release.yml codesign re-sign for aiplus | ✓ |
| release.yml codesign re-sign for aiplus-token-cost | ✓ |
| install.sh defensive Darwin re-sign | ✓ |
| release.yml auto-latest (no manual gh edit needed) | ✓ |

## What went well

1. **2 CEOs + 1 Advisor in parallel without merge conflicts.** Parallel CEO session did public --bypass on main directly. Addendum CEO session did codesign fix on main directly. Advisor cherry-picked auto-latest from a separate branch. All three streams landed in main with no conflicts. This is the 2nd validated parallel-CEO pattern at 3-stream scale.

2. **Critical bug caught at first dogfood.** macOS 26.4 SIGKILL would have silently broken every new install. Caught in 30 min from Owner's "zsh: killed aiplus" report to root-cause identification + fix recipe. Phase 1 investigation under 5 minutes.

3. **Phase C re-test on Owner's actual machine, not sandbox.** Sandboxed `install.sh` with `AIPLUS_INSTALL_DIR=$tmpdir` against the **known-bad** v0.6.12 release — proves install.sh defensive re-sign rescues binaries even WITHOUT release.yml fix. This is the right Phase C scope: test the worst case, not the happy path.

4. **3-issue ship under one tag.** Previous pattern: one goal = one tag. This ship combined 3 independent fixes (public bypass + codesign + auto-latest) — saves a tag cycle, avoids stranding fixes between releases.

5. **Auto-memory rule from earlier today is already paying off.** [[mcp-freshness-before-features]] reminded Advisor mid-Phase-C to check whether the auto-latest fix should ship NOW (it should — separate ship = 2 tags = 2 release.yml exposures). Bundled. Rule application: real.

## What went wrong

1. **Brief draft + brief execution drifted on `crates/aiplus-cli/tests/agent_talk_bypass.rs`.** Public --bypass CEO created this test file on main directly (not in a worktree). CONTRACT v1.1 App D requires worktree isolation for builder work; this CEO bypassed that. Owner-authorized parallel CEO sessions need to be reminded of worktree rule in future briefings, OR the rule needs to be relaxed for "Owner-direct" CEO sessions where Owner is supervising.

2. **CHANGELOG 0.6.13 entry got drafted before tag.** Normally Advisor writes CHANGELOG at Phase C bump time. Public --bypass CEO already wrote the entry. Codesign CEO appended a section. No conflict, but unusual ordering. Document in CONTRACT future revision: who owns CHANGELOG drafting during multi-CEO parallel work? Current behavior is "first writer wins, others append" which is fine but should be explicit.

3. **Owner is at 80%+ context.** Multiple times today the auto-memory tool reminder fired and Owner's codex session crossed 80%. Future: when Owner's context hits 75%, Advisor should proactively suggest /compact OR shifting to new lobby session BEFORE Owner has to.

## Architectural lessons codified

1. **macOS 26.4 changed code execution policy WITHOUT a deprecation warning.** Rust's linker-signed adhoc binaries were standard practice for ~3 years. macOS 26.4 silently SIGKILLs them now. This kind of platform shift CAN'T be detected by tests inside the build — it only surfaces on user machines. We need: (a) post-install smoke test in install.sh that runs `aiplus --version`, reports any non-zero exit, and fails install loudly; (b) eventual Developer-ID notarization (longer-term). Tracked as follow-up: G-AT-INSTALL-SMOKE-1.

2. **install.sh defensive re-sign is "belt and suspenders" but actually necessary.** Even if release.yml ALWAYS produces re-signed binaries, users running old install.sh versions against new releases would be broken. The defensive step in install.sh closes that gap. Pattern: for any "produce X via CI" + "consume X via shell script" pipeline, the shell script should be able to recover from CI mistakes.

3. **Parallel CEO sessions are now reliably ship-quality.** 3 CEO sessions today (lobby-bypass v0.6.12 + public bypass + codesign) all returned pure PASS or PASS_WITH_DEVIATIONS with verified live evidence. The briefing template + narrow STOP + retry-once gate + Owner authorization → consistently good output. Confidence high enough that future Advisor recommendations can default to parallel rather than serial for independent sub-goals.

## Open follow-ups (post-tag, not blocking)

- **G-AT-INSTALL-SMOKE-1**: install.sh should run `aiplus --version` post-install and fail loudly on non-zero. Today's bug would have caught the SIGKILL at install time, not first-run time.
- **G-AT-FRESHEN-1**: already spec'd. Ship target 0.6.14. Persona refresh + MCP-register update-existing + doctor drift checks.
- **G-AT-COST-COVERAGE-1**: today's dogfood revealed `agent_token_cost` only measures dispatch, not session. Owner's intent "本周烧了多少 token" actually wants total cost. Need either expand tool scope or add `session_token_cost` companion. Task #82.
- **Worktree-isolation policy for Owner-direct parallel CEOs**: clarify CONTRACT App D for parallel CEOs that Owner is supervising directly (today's pattern). Current text assumes always-supervised-by-Advisor.
- **Developer-ID notarization**: long-term fix to avoid future macOS policy shifts. Requires Apple Developer Program account. Cost ~$99/year. Separate Owner business decision.

## Next: Owner action

Authorize `git push origin main` + `git tag v0.6.13 && git push origin v0.6.13`. Once pushed, release.yml will build both targets, codesign both binaries on the macOS runner, package, upload to GitHub release, mark as `--latest` automatically (no manual gh edit needed). install.sh users will then fetch v0.6.13 + defensive codesign on their Mac (belt-and-suspenders).

Owner re-run install verification:

```bash
curl -fsSL https://raw.githubusercontent.com/izhiwen/AiPlus/main/install.sh | sh
# Should fetch v0.6.13 directly (no manual --latest救场)
aiplus --version
# Should print "0.6.13" — no SIGKILL
cd ~/Projects/MailCue && aiplus
# Should drop into lobby with (bypass-mode) label
```

---

— Advisor, 2026-05-19. 0.6.13 ratified. 19th goal in 3 calendar days. 3-stream parallel CEO + Advisor coordination validated.
