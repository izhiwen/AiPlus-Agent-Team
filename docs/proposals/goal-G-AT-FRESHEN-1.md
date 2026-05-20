# Goal G-AT-FRESHEN-1 — Project freshness: MCP server, persona, preamble refresh + doctor warnings

**Status**: PROPOSED — Owner-authorized 2026-05-19 after G-AT-MCP-RESILIENCE-1 findings revealed that (a) MailCue persona stuck at pre-Discovery v2 wording AND (b) codex global MCP config still points to AEL's vendored v0.6.8 binary, not the installed v0.6.12 — meaning ALL Owner projects using codex see stale tool descriptions.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-FRESHEN-1`
**Ship target**: v0.6.14 (NOT 0.6.13 — codesign + public --bypass already bundled there)

---

## TL;DR

After G-AT-MCP-RESILIENCE-1 findings, three real freshness gaps exist on Owner's machine and likely on any aiplus user who upgrades the binary without re-running setup:

1. **Stale MCP server binary path**: `~/.codex/config.toml` `[mcp_servers.aiplus]` points to `/Users/steve/Projects/AiEconLab/vendor/aiplus/target/release/aiplus` (v0.6.8 — AEL's vendored copy), not `~/.local/bin/aiplus` (current v0.6.12). Every codex session sees v0.6.8 MCP tool descriptions. THIS is the silent killer behind the discovery layer not firing.

2. **Stale project persona files**: MailCue's `.aiplus/agents/personas/ceo.md` is the pre-Discovery-v2 version — no "Use These Tools First", no `agent_token_cost` mention, no MCP-first guidance. Manual hotfix applied today; same risk exists in every project Owner has set up before v0.6.9.

3. **No visibility into either drift**: `aiplus doctor` currently doesn't check binary-path freshness or template-vs-installed-persona drift. Owner can run `aiplus doctor` and pass, while still being silently stuck on v0.6.8 MCP and v0.1 personas.

This goal makes drift detectable + fixable.

## Success criterion (single sentence)

> A user upgrading the `aiplus` binary can run `aiplus doctor` to detect stale MCP-server binary paths + project persona drift, and `aiplus mcp-register --runtime <r>` + `aiplus persona refresh [--dry-run]` to fix both, with diff prompts before overwrite.

## Goals (D1-D4)

- **D1**: `aiplus mcp-register --runtime codex` (and claude-code / opencode) detects existing config entries pointing to a different binary path, and either auto-updates to the current `aiplus` binary or prompts the user to confirm overwrite.

- **D2**: New command `aiplus persona refresh [--dry-run]`:
  - Walks `.aiplus/agents/personas/*.md` in current project
  - Compares each persona to its canonical template (via known template digest or content match)
  - If drift detected: shows diff, asks Owner to confirm refresh (or prints diff only with `--dry-run`)
  - Preserves project-specific persona customizations the user has clearly added (heuristic: if persona has Owner-edited sections, keep them; if it's drifting only on the SKILL/preamble block, replace just that block)
  - Same for `.aiplus/AGENTS.aiplus.md` preamble block

- **D3**: `aiplus doctor` adds two new checks:
  - **WARN**: MCP server binary path mismatch (configured binary vs `which aiplus`)
  - **WARN**: project persona / preamble drift vs canonical templates
  - Both report as WARN (not FAIL) — user may intentionally vendor old binary or customize persona

- **D4 (small, separate sub-deliverable)**: SKILL.md + `agent_token_cost` tool description enriched with Chinese examples:
  - SKILL.md "Cost / spending" trigger examples adds: `本周烧了多少 token`, `最近花了多少`, `cost this week`
  - `agent_token_cost` tool description in `mcp_server.rs` adds: "Use this for queries like '本周烧了多少 token', 'cost this week', 'token spending', 'AI burn'."

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/freshen-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/freshen-1-impl-notes.md` | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/mcp_register.rs` (or equivalent): existing-config detection + update | PENDING |
| 5 | New `crates/aiplus-cli/src/persona/refresh.rs` (or in install module) | PENDING |
| 6 | `crates/aiplus-cli/src/doctor.rs` two new WARN checks | PENDING |
| 7 | `assets/aiplus-agent-team/adapters/{codex,claude-code,opencode}/skills/aiplus/SKILL.md` Chinese example additions (3 files) | PENDING |
| 8 | `crates/aiplus-cli/src/mcp_server.rs` `agent_token_cost` description Chinese-trigger additions | PENDING |
| 9 | Tests for: persona refresh dry-run / persona refresh apply / doctor WARN cases / mcp-register update behavior | PENDING |

## Out of scope

- Updating MailCue or AEL or other Owner projects' files directly (CLI ships; Owner runs commands in their projects)
- Auto-refresh on `aiplus install <runtime>` rerun (D2 is a separate explicit command; D1 prompts on `mcp-register`)
- Migration of pre-v0.6.x persona/config formats (those are gone; v0.6+ is the floor)
- Forbidden files: CONTRACT.md, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh, release.yml

## Acceptance criteria

- ✓ D1-D4 met
- ✓ `cargo test --workspace` PASS (595+ tests + new tests)
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase C live test (Advisor):
  - In a sandbox with stale MCP config (binary path → v0.6.8 fixture), `aiplus doctor` WARNs
  - `aiplus mcp-register --runtime codex` updates the path with confirm prompt
  - `aiplus persona refresh --dry-run` shows diff against canonical for a known-stale persona fixture
  - `aiplus persona refresh` (without --dry-run) overwrites with prompt
  - Existing v0.6.12 behavior unchanged for users who don't run any new commands

## Dependencies

- v0.6.13 ship (codesign fix + public --bypass) — must land first, then this goal builds on 0.6.13
- v0.6.12 main ✓

## Why this exists

Discovery layer + MCP tools investment was supposed to make natural-language queries route to structured MCP results. After 2 weeks of building this, Owner's first real-world dogfood on MailCue showed it didn't fire — not because of Discovery layer design, but because Owner's machine had silent drift in 2 layers (MCP server binary path + project persona). Without freshness detection + refresh tooling, every binary upgrade silently breaks the discovery contract.

This is a sustainability fix, not a feature.

---

— Advisor, 2026-05-19. Spawned from G-AT-MCP-RESILIENCE-1 findings (Category B + secondary D). Ship target 0.6.14, after 0.6.13. 1-2 day goal.
