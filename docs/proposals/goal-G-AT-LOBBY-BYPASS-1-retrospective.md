# Goal G-AT-LOBBY-BYPASS-1 — Retrospective (COMPLETE)

**Status**: **DONE** — Lobby trusted-owner bypass mode shipped in v0.6.12 on 2026-05-19.
**Original goal**: `docs/proposals/goal-G-AT-LOBBY-BYPASS-1.md`
**Closing date**: 2026-05-19 (opened + closed same day)
**Duration**: ~0.5 work day (CEO ~14 min impl + Advisor Phase C ~10 min) vs T+1d budget. 18th goal in 3 calendar days.

---

## Why this goal existed

Owner ran v0.6.11 lobby on their real MailCue project — first true new-user UX — and realized: codex (and the other runtimes) default to "ask for approval before every action". Friction-y for the actual primary use case (Owner working on own machine on own project, agent trust already established). Owner asked for lobby to default to "trusted owner" mode with per-runtime bypass flags, plus an escape hatch for shared/CI/prod-like contexts.

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | Goal doc + CEO briefing | `e55d605` | aiplus-agent-team |
| 2 | Lobby spawn + agent talk plumbing + --safe + env-var + tests + impl-notes | `27b89af` | aiplus-public |
| 3 | Phase C: 0.6.11 → 0.6.12 + CHANGELOG + install.sh parity | `571a9d1` | aiplus-public |
| 4 | This retrospective | post-`571a9d1` | aiplus-agent-team |

**Total impl**: ~279 lines (35 lobby + 35 talk + 5 main.rs + 100 tests + 112 impl-notes + 60 CHANGELOG).

## D-goal status

- D1 (lobby default uses per-runtime bypass flag): ✓
- D2 (`aiplus --safe` flag disables bypass): ✓
- D3 (`AIPLUS_LOBBY_SAFE=1` env var disables bypass; accepts 1/true/yes/on): ✓
- D4 (spawn line shows `(bypass-mode)` vs `(safe-mode)`): ✓

## Per-runtime flag confirmation

| Runtime | Bypass flag | Source of confirmation |
|---|---|---|
| Codex | `--dangerously-bypass-approvals-and-sandbox` | `codex --help` top-level |
| Claude Code | `--dangerously-skip-permissions` | `claude --help` top-level |
| OpenCode | `--dangerously-skip-permissions` | `opencode run --help` (NOT visible in top-level help, but accepted by top-level parser) |

Lesson archived: **opencode's bypass flag is documented at subcommand level, not top-level**. Future contributors should look at `<runtime> <subcommand> --help` if the top-level help is silent on a flag.

## Advisor live re-test (per new methodology)

Sandbox: isolated `HOME`, `CODEX_HOME`, `PATH`, project dir; fake codex binary that logs argv. All three cases tested:

| Case | Expected mode | Actual mode label | Actual spawn argv |
|---|---|---|---|
| Bare `aiplus` (default) | bypass-mode | `(bypass-mode)` ✓ | `--dangerously-bypass-approvals-and-sandbox …` ✓ |
| `aiplus --safe` | safe-mode | `(safe-mode)` ✓ | (no bypass flag) ✓ |
| `AIPLUS_LOBBY_SAFE=1 aiplus` | safe-mode | `(safe-mode)` ✓ | (no bypass flag) ✓ |

CEO's PASS was real. No divergence between CEO's reported transcript and Advisor's independent re-test.

Test suite: `cargo test --workspace` 595 passed / 2 ignored. Clippy strict clean. `cargo build` produced 0.6.12 binary.

## What went well

1. **CEO produced pure PASS verdict.** No PASS_WITH_DEVIATIONS, no STOP, no retry-once. Briefing template + scope discipline + narrow STOP rule continue to deliver predictable execution. This is the 4th consecutive goal with pure PASS (LOBBY-1, then BYPASS-1).
2. **Hidden lobby-only `--lobby-bypass` switch on `agent talk`.** CEO chose to add a hidden flag (`hide = true` on clap) instead of changing `agent talk`'s public surface. Surgical: lobby gets bypass, non-lobby callers get no behavior change, no public CLI surface area pollution. The right design choice surfaced from CEO's investigation, not the briefing.
3. **Spawn line mode label.** "(bypass-mode)" / "(safe-mode)" appended to existing launch line — minimal UI change, maximum signal. Prevents the "wait, am I in safe or bypass mode?" question.
4. **Env var accepts truthy variants.** Owner didn't ask for this; CEO added `true/yes/on` parsing on top of `1`. Small ergonomic improvement, no asked-for-it tax.
5. **Phase C took 10 minutes.** Methodology overhead is now nearly free. CEO PASS + Advisor independent live-test + ff-merge + bump + CHANGELOG + retro fits in <15 min on a small goal.

## What went wrong

1. **Advisor's first live-test attempt botched HOME ordering.** `export HOME=$SANDBOX/home; AIPLUS="$HOME/Projects/…"` — second line resolved against the just-changed HOME. Trivially fixed (capture binary path before exporting HOME). Note for future Phase C scripts: **resolve binary paths BEFORE mutating HOME**.

## Architectural lessons codified

1. **Hidden CLI flags for lobby-only behavior is a strong pattern.** When a feature is conceptually "the lobby wants extra behavior from a subcommand it calls", the surgical solution is a hidden, lobby-only flag on that subcommand — not changing the subcommand's default. Public CLI surface stays clean, lobby gets what it needs, regression risk is contained to lobby callers.

2. **Default UX should match the primary user.** Owner is the primary user of lobby (and most of aiplus). Designing lobby's default for "personal machine, trusted agent" is right; the safe escape hatch covers the secondary "shared/CI" case. This inverts the classic "default to safe, opt into convenience" — but the inversion is correct when the primary user IS the trusted party.

3. **CHANGELOG safety-posture note is non-optional for behavior-mode changes.** Bypass mode skips approval prompts — that's a real safety posture change. CHANGELOG explicitly states the intended deployment context (personal, not shared/CI/prod). Future "default behavior change" goals should follow the same pattern: state the new default, state the intended context, state the escape hatch.

## Open follow-ups

- **Release workflow auto-mark-latest** — separate followup (not BYPASS-1 scope) noted earlier today, ~5-line release.yml change so v0.6.12 (and future) tags are automatically marked latest without manual `gh release edit --latest`.
- **Lobby preferences persistence** — still deferred (was already an open follow-up from LOBBY-1).
- **OpenCode top-level help silence on `--dangerously-skip-permissions`** — upstream concern (opencode bug?) not aiplus scope. Document in impl-notes.

## Owner action item

After ratification, Owner should explicitly authorize tag push (no auto-tag per Owner gates). Then re-run `curl install.sh | sh` on the MailCue project to upgrade from v0.6.11 → v0.6.12 and verify the bypass-mode label appears.

---

— Advisor, 2026-05-19. Lobby bypass in 0.5 day (CEO 14 min + Advisor 10 min). Phase C methodology overhead approaching free. 18th goal in 3 calendar days.
