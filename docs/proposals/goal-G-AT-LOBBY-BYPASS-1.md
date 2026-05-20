# Goal G-AT-LOBBY-BYPASS-1 — Lobby spawns runtimes with trusted-owner bypass flags

**Status**: PROPOSED — Owner-authorized 2026-05-19 after first lobby use revealed that runtimes default to per-action approval prompts, friction-y in normal Owner workflow.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-LOBBY-BYPASS-1`

---

## TL;DR

v0.6.11 lobby spawns runtimes (codex / claude-code / opencode) with their **default** flags, which means every action the agent takes prompts Owner for approval. This adds significant friction for Owner working on their own project on their own machine. Owner wants lobby to default to "trusted owner" mode: per-runtime bypass flags that skip approval prompts.

Example: `aiplus` → pick ceo → pick codex → lobby launches `codex --dangerously-bypass-approvals-and-sandbox` (not just `codex`).

Default ON; provide an escape hatch (`aiplus --safe` flag OR env var `AIPLUS_LOBBY_SAFE=1`) for users who want approval-mode despite running through lobby.

## Success criterion (single sentence)

> `aiplus` lobby spawn step launches the chosen runtime with the appropriate bypass flag included (codex with `--dangerously-bypass-approvals-and-sandbox`, claude with `--dangerously-skip-permissions`, opencode with its equivalent), unless `--safe` flag passed or `AIPLUS_LOBBY_SAFE=1` env var is set.

## Goals (D1-D4)

- **D1**: Lobby spawn step uses per-runtime bypass flags by default.
- **D2**: `aiplus --safe` CLI flag (or equivalent) disables bypass mode.
- **D3**: `AIPLUS_LOBBY_SAFE=1` env var also disables bypass mode (for shell sessions / scripts).
- **D4**: Lobby print line ("[Launching codex with ceo persona...]") expanded to include flag info: e.g., "[Launching codex with ceo persona (bypass-mode)...]" or "[Launching codex with ceo persona (safe-mode)...]" so user knows which mode they're in.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/lobby-bypass-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/lobby-bypass-impl-notes.md` | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/lobby/mod.rs` extension: bypass flag per runtime | PENDING |
| 5 | `crates/aiplus-cli/src/main.rs` extension: `--safe` flag on bare aiplus | PENDING |
| 6 | Tests covering bypass-default + safe-mode-via-flag + safe-mode-via-env | PENDING |

## Out of scope

- Modifying `aiplus agent talk` subcommand behavior (lobby calls it; non-lobby invocations stay as-is)
- Changing default flags for non-lobby code paths (`aiplus agent route ...` etc.)
- New non-bypass features in lobby
- Forbidden files: CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh

## Acceptance criteria

- ✓ D1-D4 met
- ✓ `cargo test --workspace` PASS (592+ tests)
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase C live test (Advisor): in a fresh sandbox, `aiplus` → pick role → pick codex → process tree should show `codex --dangerously-bypass-approvals-and-sandbox` (verify via `ps` or by examining `aiplus agent talk` invocation logs)
- ✓ `aiplus --safe` and `AIPLUS_LOBBY_SAFE=1 aiplus` both work and emit "(safe-mode)" indicator
- ✓ Existing v0.6.11 lobby flow unchanged for users who haven't passed --safe / env

## Dependencies

- v0.6.11 main ✓ (lobby code in place)
- Lobby's spawn mechanism (need to identify exactly where the spawn happens — Phase 1)

## Why this exists

Owner ran lobby for the first time on a real project, and noted that codex (and likely claude-code / opencode) defaults to "ask for approval before every action". This friction makes lobby less useful than intended. Owner already trusts the agent on their own machine in their own project, so should not be re-asked at every step.

Note: this CHANGES THE SAFETY POSTURE of lobby. Document clearly in CHANGELOG; clarify that bypass mode is intended for personal use, not multi-user / shared / production deployment.

---

— Advisor, 2026-05-19. Half-day follow-up to v0.6.11 lobby.
