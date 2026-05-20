# G-AT-COST-COVERAGE-1 — CEO Impl Briefing

**Drafted**: 2026-05-19 by Advisor
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-COST-COVERAGE-1.md`
- Auto-memory: `advisor_briefing_stop_rules.md`, `parallel_ceo_tag_gate.md`

---

## Worktree isolation (NON-OPTIONAL)

- Worktree:  `~/Projects/AiPlus/aiplus-public.cost-coverage/`
- Branch:    `feat/cost-coverage-1`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.cost-coverage -b feat/cost-coverage-1`

## Reference: Codex session DB schema (from Owner's actual data)

`/Users/steve/.codex/state_5.sqlite` `threads` table relevant columns:
- `id` (TEXT)
- `created_at` (INTEGER unix epoch)
- `updated_at` (INTEGER unix epoch)
- `tokens_used` (INTEGER, total tokens for thread)
- `cwd` (TEXT, working directory)
- `title` (TEXT)
- `rollout_path` (TEXT)

Example queries (already verified by Owner's manual dig earlier today):
```sql
-- This week total
SELECT SUM(tokens_used) FROM threads
WHERE created_at >= strftime('%s', '2026-05-18 00:00:00', 'utc');

-- Per project this week
SELECT cwd, COUNT(*), SUM(tokens_used) FROM threads
WHERE created_at >= strftime('%s', '2026-05-18 00:00:00', 'utc')
GROUP BY cwd;
```

This gives you session token counts. You'll need to convert tokens → USD using existing pricing logic in `aiplus-token-cost` crate (already there for dispatch cost).

## Scope

### Phase 1 — Investigation + design (~1.5h)

1. Read existing `agent_token_cost` MCP tool implementation in `crates/aiplus-cli/src/mcp_server.rs` and `crates/aiplus-cli/src/agent/cost/` (if it exists, else find where it lives).
2. Read pricing logic in `aiplus-token-cost` crate to understand the rates table and how it's queried.
3. Investigate claude-code session state location/schema (likely `~/.claude/projects/*/...jsonl` rollouts — Owner has these too). If too complex, document and stub out — codex is the primary case.
4. Design the new schema (D1):
   - Option A: add new fields (`session_tokens`, `session_usd`) alongside existing (`total_tokens` → keeps dispatch only). Backward-compat.
   - Option B: rename existing to `dispatch_tokens` / `dispatch_usd`, add `session_tokens` / `session_usd`, add `total_tokens` = sum. Breaks consumers but cleaner.
   - **Recommend**: A for first release (backward-compat priority); deprecation path documented for B.
5. Design new flags: `--include-session` (default ON for "tokens this week" intent), `--dispatch-only`, `--session-only`.
6. Decide window semantics: existing windows are 1h / 8h / 24h. Add weekly window? Or accept that weekly is `--window 168h` or `--window 7d` (extend parser).

### Phase 2 — Implementation (~3-4h)

1. Add `crates/aiplus-cli/src/agent/cost/session.rs` (or extend existing cost module):
   - Function: `session_tokens(start_unix, end_unix, cwd_filter: Option<&str>) -> Result<u64>`
   - Reads `~/.codex/state_5.sqlite` (graceful skip if missing)
   - Aggregates `tokens_used` by criteria
   - Converts to USD using same pricing logic as dispatch path
2. Extend `agent_token_cost` MCP tool:
   - Accept new params: `include_session: bool` (default true), `session_only: bool` (default false), `dispatch_only: bool` (default false)
   - Return windows with both dispatch + session breakdowns
3. Extend `aiplus agent token-cost` CLI with matching flags
4. Add weekly window option (`--window 7d` or `--window 168h`)
5. Tool description in mcp_server.rs:
   - Add Chinese trigger examples: "本周烧了多少 token", "最近花了多少", "cost this week", "今天用了多少"
   - Document new fields clearly: "session_tokens = your codex/claude session usage; dispatch_tokens = aiplus agent dispatch usage; total = sum"
6. SKILL.md updates (codex / claude-code / opencode adapters):
   - Cost trigger examples section adds the Chinese phrases
   - Example flow updated: "First call agent_token_cost. By default includes both session + dispatch. Use --session-only or --dispatch-only to slice."

### Phase 3 — Evidence (~1h)

- `cargo test --workspace` PASS (new tests for session.rs)
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- Live test on Owner's machine:
  - `aiplus agent token-cost --window 168h` returns weekly numbers, session > 0 (Owner's been burning sessions all day)
  - `--dispatch-only` returns dispatch only
  - `--session-only` returns session only
- Discovery test: in a sandboxed lobby+codex session with current SKILL.md, prompt "看下我这周烧了多少 token" — verify CEO calls agent_token_cost AND surfaces session number

## STOP rules (per auto-memory)

- `_REGRESSION_` → STOP
- `_SCOPE-BREACH_` → STOP
- `_TAG-BREACH_` → STOP (NEVER tag, NEVER bump Cargo.toml, NEVER push)
- Retry-once gate standard

**Forbidden files**:
- CONTRACT.md, install.sh, release.yml, Cargo.toml version, CHANGELOG.md
- crates/aiplus-token-cost/ (token-cost crate is read-only dependency; you may CALL pricing functions, do not modify)
- crates/aiplus-cli/src/{mcp_register,persona,doctor}.rs (G-AT-FRESHEN-1 owns)
- Anywhere outside `crates/aiplus-cli/src/agent/cost/`, `crates/aiplus-cli/src/mcp_server.rs` (limited to agent_token_cost handler), `assets/aiplus-agent-team/adapters/*/skills/aiplus/SKILL.md`
- `~/.codex/state_5.sqlite` is READ-ONLY (never write)

## Scope fence

- **IN**: agent/cost/ extension, agent_token_cost MCP handler + description, CLI flags, SKILL.md Chinese examples, tests
- **OUT**: persona refresh, mcp-register, doctor, install.sh, release.yml, version bump, CHANGELOG
- **GRAY**: minor refactor of existing dispatch-cost logic if needed to share window aggregation with session-cost

## Deliverables (5)

1. `docs/proposals/cost-coverage-1-impl-notes.md` — Phase 1 + Phase 3
2. `crates/aiplus-cli/src/agent/cost/session.rs` (new) + tests
3. `agent_token_cost` MCP tool extension (handler + description in mcp_server.rs)
4. `aiplus agent token-cost` CLI flag extensions
5. SKILL.md × 3 Chinese trigger example additions

## Time ceiling

Hard cap T+1d. Realistic estimate ~5-6h.

## Handoff

IMPL_VERDICT. NO TAG. NO VERSION BUMP. NO PUSH ORIGIN MAIN. Branch push for PR OK.

---

— Advisor, 2026-05-19. Parallel-safe with FRESHEN-1 (disjoint files) + INSTALL-SMOKE-1 (different file).
