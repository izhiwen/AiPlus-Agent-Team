# Goal G-AT-LOBBY-1 — Bare `aiplus` lobby UX (auto-install + role + runtime select)

**Status**: PROPOSED — Owner-authorized 2026-05-19 to mirror AEL Advisor's lobby flow in aiplus.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-LOBBY-1`

---

## TL;DR

AEL's user-facing entrypoint is the bare `ael` command — type one word in a project dir, get auto-install + role-picker + runtime-picker + launch. Owner wants the same UX for aiplus. This goal makes bare `aiplus` (in a project dir, no args) auto-set-up the project (if needed), then drop the user into a lobby that asks "who do you want to talk to?" → "which runtime?" → launches the chosen role into the chosen runtime with persona loaded.

Owner-stated design decisions (mirror AEL where applicable):
- **Auto-install all 3 runtime adapters** when `.aiplus/` is missing; silently skip runtimes whose CLI isn't on PATH
- **Single-runtime case skips the runtime prompt**; multi-runtime case asks
- **Default role on empty Enter = `ceo`** (aiplus-specific; AEL defaults to `pi`)
- **No team prompt in v1** — use the project's currently active team
- **Lobby is complementary to v0.6.10 discovery layer** — lobby gets user into a role+runtime quickly; discovery layer handles natural-language tool invocation inside that session

## Success criterion (single sentence)

> A user does `cd <some-project> && aiplus`; if `.aiplus/` doesn't exist, aiplus installs all available runtime adapters (silently skipping missing CLIs) and registers MCP; then prompts for a role (Enter = ceo); then if multiple runtimes installed prompts for runtime; then spawns the chosen runtime with the chosen role's persona, all without the user typing any subcommand.

## Goals (D1-D5)

- **D1**: Bare `aiplus` (no subcommand, project dir cwd) routes to new lobby flow. Existing `aiplus <subcommand>` paths unchanged.
- **D2**: Lobby detects whether project is set up (`.aiplus/agents/` present + at least one runtime adapter installed). If not set up, runs `aiplus install all --yes` internally, then `aiplus mcp-register` for available runtimes. Skips runtimes whose CLI isn't on PATH (claude / codex / opencode not found → silent skip + continue).
- **D3**: Role picker. Lists active team's roles with one-line description. Empty Enter → `ceo`. Accepts partial match (e.g., `eng` matches `engineer-a` if unique).
- **D4**: Runtime picker. Detects available runtimes (claude-code / codex / opencode). If exactly 1 → skip prompt, use it. If 2+ → prompt; accept partial match.
- **D5**: Spawn. Effectively invoke `aiplus agent talk --runtime <r> <role>` (or whatever the equivalent internal API is). Phase 1 confirms the right primitive to call.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/lobby-1-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/lobby-1-impl-notes.md` | PENDING — CEO |
| 4 | New `crates/aiplus-cli/src/lobby/` module (or `lobby.rs`) | PENDING |
| 5 | `crates/aiplus-cli/src/main.rs` dispatch for bare `aiplus` | PENDING |
| 6 | New integration tests covering lobby flow | PENDING |

## Out of scope

- Team selection (use active team only; switching is `aiplus agent set-team`)
- Slash-command-style invocation inside the runtime (existing v0.6.10 discovery handles natural language)
- New `aiplus agent talk` semantics (lobby calls existing primitive; refactor talk if Phase 1 shows it doesn't support runtime selection)
- Modifying behavior of `aiplus <subcommand>` paths (only adds bare-aiplus dispatch)
- Standard forbidden files: CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh fallback

## Acceptance criteria

- ✓ D1-D5 met
- ✓ `cargo test --workspace` PASS (585+ tests)
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase C live test (Advisor):
  - In a fresh `/tmp/lobby-test/` dir without `.aiplus/`, `aiplus` triggers auto-install
  - In a project with only claude-code CLI on PATH, lobby skips codex/opencode adapter install AND runtime picker (single-runtime case)
  - Empty Enter at role prompt → ceo launches
  - Partial match `eng` (when only engineer-a exists in team) auto-completes
  - Existing `aiplus agent route ...` and other subcommands unchanged

## Dependencies

- v0.6.10 main ✓
- Existing `aiplus install`, `aiplus mcp-register`, `aiplus agent talk` primitives

## Why this exists

Owner observed AEL's lobby UX (transcript copied into this conversation). Bare `ael` in a project dir is the single-word entry point that handles everything. Owner wants parity for aiplus. Same UX pattern reduces cognitive load when Owner switches between projects.

---

— Advisor, 2026-05-19. Lobby UX mirroring AEL pattern.
