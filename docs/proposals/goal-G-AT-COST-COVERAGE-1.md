# Goal G-AT-COST-COVERAGE-1 — Token cost coverage: session + dispatch

**Status**: PROPOSED — Owner-authorized 2026-05-19 after MailCue dogfood revealed `agent_token_cost` only measures aiplus agent dispatch, not codex/claude session token cost. Owner's real question was "total burn this week", not "dispatch cost this week".
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-COST-COVERAGE-1`
**Ship target**: v0.6.14 (bundled with FRESHEN-1)

---

## TL;DR

Today's MailCue dogfood: Owner asked "看下我这周烧了多少 token". CEO routed correctly to `agent_token_cost` MCP tool (Discovery layer fired!). Tool returned 0 — accurate, because MailCue has no aiplus dispatch log yet. But Owner's actual intent was **total cost including the codex session itself** (where the bulk of the spend lives).

This goal extends `agent_token_cost` to optionally include session-level token cost by reading the local codex session DB (`~/.codex/state_5.sqlite`) and equivalent claude-code state. Plus adds Chinese trigger examples to SKILL.md + tool description (the former D4 of G-AT-FRESHEN-1, moved here to avoid mcp_server.rs conflict with FRESHEN-1).

## Success criterion (single sentence)

> User asks "本周烧了多少 token" → `agent_token_cost` MCP tool returns BOTH aiplus agent dispatch cost AND the user's codex/claude session token usage for that window, with clear field labels so CEO can surface either or both.

## Goals (D1-D4)

- **D1**: Extend `agent_token_cost` MCP tool result schema to include `session_tokens` and `session_usd` fields per window, alongside existing `total_tokens` / `total_usd` (rename existing to `dispatch_tokens` / `dispatch_usd` OR add new fields without renaming — Phase 1 decides for backward-compat).
- **D2**: Detect + read `~/.codex/state_5.sqlite` `threads` table; aggregate `tokens_used` filtered by `created_at >= window_start`. Optional `cwd=...` filter for project-scoped queries.
- **D3**: (If applicable) Detect + read claude-code session state — Phase 1 investigates location/schema; if too complex, document and defer to a separate goal.
- **D4**: SKILL.md + tool description Chinese trigger examples (moved from FRESHEN-1 D4):
  - SKILL.md "Cost / spending" section adds: `本周烧了多少 token`, `最近花了多少`, `cost this week`, `今天用了多少`, `AI burn this month`
  - `agent_token_cost` tool description in `mcp_server.rs` adds matching Chinese examples + clarifies new dispatch vs session field semantics
- **D5**: New `--include` / `--session-only` / `--dispatch-only` flags on `aiplus agent token-cost` CLI (and corresponding MCP tool params) for explicit slicing.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/cost-coverage-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/cost-coverage-1-impl-notes.md` | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/agent/cost/session.rs` (new) — codex sqlite reader | PENDING |
| 5 | `crates/aiplus-cli/src/agent/cost/mod.rs` — combine dispatch + session windows | PENDING |
| 6 | `crates/aiplus-cli/src/mcp_server.rs` — extend `agent_token_cost` handler signature + description | PENDING |
| 7 | SKILL.md × 3 (codex / claude-code / opencode adapters) | PENDING |
| 8 | Tests + fixtures | PENDING |

## Out of scope

- Reading codex's encrypted state in any way that requires non-public access
- Real-time monitoring of token usage (this is rollup reads, not stream)
- Cost projection / budgeting (`agent_budget` would be a separate goal)
- Mac Keychain / secure-enclave operations
- Modifying `~/.codex/state_5.sqlite` in any way (read-only access)
- Forbidden files: CONTRACT.md, install.sh, release.yml, Cargo.toml version, CHANGELOG.md, crates/aiplus-token-cost/

## Acceptance criteria

- ✓ D1-D5 met
- ✓ `cargo test --workspace` PASS
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Live test on Owner's machine (Advisor Phase C):
  - `aiplus agent token-cost` returns both session + dispatch numbers
  - Session number is non-zero (Owner has been burning codex sessions all week)
  - Field names clear in JSON output
  - Window filters work (1h, 8h, 24h, week)
- ✓ Discovery layer test: in a fresh lobby session, asking "看下我这周烧了多少 token" routes to `agent_token_cost` AND surfaces session number, not just dispatch

## Why this exists

Discovery layer working (verified today) is necessary but not sufficient — the tool the layer routes to has to actually answer the user's intent. Today's MailCue test showed the routing was correct but the answer was technically-true-but-not-useful ("0 dispatch cost" when Owner wanted "$X session cost"). This goal closes the loop.

---

— Advisor, 2026-05-19. Spawned from today's MailCue dogfood gap. 1-day goal. Ship 0.6.14.
