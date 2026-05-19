# Autoflow Coverage Phase 1 — CEO Session A Briefing

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session A (parallel with Session B)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AUTOFLOW-COVERAGE-1.md`
- Companion parallel goal (Session B): `goal-G-AT-AUTOFLOW-MULTITURN-1.md` — read to know what Session B owns so you don't collide
- v0.6.9 baseline in main: `aiplus-public@e6f3870`
- Existing MCP tools in `crates/aiplus-cli/src/mcp_server.rs` (14 total)
- v0.6.9 SKILL.md content in `assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.autoflow-coverage/`
- Branch:    `feat/autoflow-coverage`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.autoflow-coverage -b feat/autoflow-coverage`

## CRITICAL — shared-file ownership matrix (Wave 1 parallel with Session B)

Session B (G-AT-AUTOFLOW-MULTITURN-1) runs concurrently on `feat/autoflow-multiturn` branch. **You will both touch SKILL.md × 3 and the install-logic preamble code**. To avoid conflicts:

### Files YOU OWN
- `crates/aiplus-cli/src/mcp_server.rs` — enhance the **11 existing tool descriptions** (NOT the 3 already done in v0.6.9: `agent_token_cost`, `agent_audit_verify_log`, `agent_route_score_only`)
- `assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md` × 3 — **only** modify the existing "Use These Tools First" section (extend its bullet list with the 11 MCP tools + non-MCP features). Do NOT touch other sections.
- `crates/aiplus-cli/src/main.rs` install logic — only the **intent list** content inside the preamble managed-block string. Do NOT add new sections; that's B's domain.

### Files Session B OWNS — DO NOT TOUCH
- SKILL.md × 3 new sections "Dispatch Flow" + "Multi-turn Patterns" (B will append after existing sections)
- Project-root preamble managed-block "Dispatch Flow" sub-section (B appends after your intent list)

### Files BOTH must NOT modify
- CONTRACT.md (FROZEN)
- `crates/aiplus-token-cost/` (subtree mirror)
- `coordinator.rs` scoring rubric
- `tests/fixtures/coordinator_calibration.toml` existing entries
- Adapter code
- The 3 v0.6.9-shipped MCP tool descriptions (already perfect; don't double-edit)
- `Cargo.toml` version
- `CHANGELOG.md` actual
- `install.sh` fallback

### Conflict prevention
- If you find SKILL.md edits conflict with B's areas: STOP and ping Advisor
- Don't merge B's work into your branch; Advisor handles cross-branch merge at Phase C
- Each session pushes own branch only

## Scope — extend discovery to 11 other MCP tools + 6 non-MCP feature categories

### MCP tool description enhancement (mcp_server.rs)

For each of the 11 existing tools, prepend description with "PREFERRED programmatic surface for <intent>. Use this MCP tool instead of <CLI alternative if any>." Specifically:

| Tool | Intent | CLI alternative |
|---|---|---|
| `agent_route` | dispatch a task to a specific role | `aiplus agent route <role> "<task>"` |
| `agent_status` | get current team status | `aiplus agent status` |
| `agent_set_team` | switch active team (e.g., to AEL) | `aiplus agent set-team <name>` |
| `agent_list` | list all available roles | `aiplus agent list` |
| `agent_doctor` | run agent-specific health checks | `aiplus agent doctor` |
| `agent_invite` | bring a role into active team | `aiplus agent invite <role>` |
| `agent_dismiss` | remove a role from active team | `aiplus agent dismiss <role>` |
| `agent_disable` | disable a role | `aiplus agent disable <role>` |
| `agent_enable` | re-enable a disabled role | `aiplus agent enable <role>` |
| `agent_integrate` | merge a role's worktree back to main | `aiplus agent integrate <role>` |
| `agent_talk` | hold a single-turn conversation with a role | `aiplus agent talk <role>` |

Description format example for `agent_status`:
```
"description": "PREFERRED programmatic surface for team status queries. Use this MCP tool instead of `aiplus agent status` CLI when the user asks about team configuration, active roles, or current state. Returns structured JSON listing each role's status, worktree state, and recent activity."
```

Each ≤ 400 chars.

### SKILL.md "Use These Tools First" section extension

Current section (v0.6.9) lists 3 tools. Extend to cover ALL 14 + 6 non-MCP categories. Group by topic:

```markdown
## Use These Tools First

### Cost / spending / token usage (MCP tools, returns structured JSON):
- `agent_token_cost` — token + USD rollups (1h / 8h / 24h windows)

### Planning / task preview / scoring (MCP):
- `agent_route_score_only` — pre-flight a task to see staffing + risk

### Audit / log integrity (MCP):
- `agent_audit_verify_log` — verify dispatch log hash chain

### Dispatching / role management (MCP):
- `agent_route` — assign a task to a specific role and start work
- `agent_invite` — bring a role into active team
- `agent_dismiss` — remove a role from active team
- `agent_disable` / `agent_enable` — temporarily disable / re-enable a role
- `agent_integrate` — merge a role's worktree back
- `agent_talk` — single-turn chat with one role

### Team status / configuration (MCP):
- `agent_status` — current team status, active roles, recent activity
- `agent_list` — list all available roles
- `agent_set_team` — switch active team (e.g., to AiEconLab)
- `agent_doctor` — agent-specific health checks

### Memory / context (non-MCP CLI, ALSO PREFERRED over shell grep):
- `aiplus memory record` — store project conventions / naming rules / facts
- `aiplus memory context --runtime <runtime>` — build memory context for an agent session
- `aiplus memory status` — see what's in memory

### Compact / session token efficiency (non-MCP CLI):
- `aiplus compact prepare` — build handoff capsule before /compact
- `aiplus compact resume` — restore state after /compact
- `aiplus compact savings` — see token + cost savings from compact-prep

### Velocity / time tracking (non-MCP CLI):
- `aiplus velocity estimate --task-type <type> --human-estimate <hours>` — log an estimate
- `aiplus velocity report` — see your AI-native p50/p90 calibrated from history

### Identity / commit signing (non-MCP CLI):
- `aiplus identity setup-signing [--dry-run]` — set up Mac Secure Enclave commit signing

### Doctor (cross-cutting health):
- `aiplus doctor [--quiet] [--check-keyring]` — full health check
```

### Project-root preamble intent list extension

In the existing `<!-- aiplus-discovery-block -->` managed-block (CLAUDE.md / AGENTS.md / `.opencode/instructions/aiplus.md`), extend the **intent list** (the bullets between the header and the "for coding tasks" guidance). Keep the existing 3 high-priority bullets at top; add the new ones grouped by topic similar to SKILL.md but more compact.

Do NOT add new sections inside the managed-block — Session B will append a "Dispatch Flow" section after your intent list. Your edit is bullet-list extension only.

## Phase structure

### Phase 1 — Write `docs/proposals/autoflow-coverage-1-impl-notes.md` BEFORE coding

Required:
1. Confirm Session B doesn't touch the "Use These Tools First" section (read Session B's briefing)
2. Design final SKILL.md "Use These Tools First" content (one variant works for all 3 runtimes if possible)
3. Design final 11 MCP tool descriptions (≤ 400 chars each)
4. Design preamble intent-list extension
5. Plan idempotency tests
6. Test plan (file-content + idempotency, NOT live LLM)
7. CHANGELOG draft text

### Phase 2 — Implement

Order:
1. mcp_server.rs descriptions for 11 tools
2. SKILL.md × 3 "Use These Tools First" extension
3. Preamble intent-list extension in main.rs install logic
4. Tests

### Phase 3 — Evidence

- `cargo test --workspace` PASS (581+ tests)
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- File content tests assert key phrases present
- (Advisor will run multi-runtime live re-test at Phase C; you don't need to do live LLM tests for this goal — Wave 2 does that systematically)
- Evidence trailer in impl-notes

## STOP rule

- `_REGRESSION_` (existing 581 tests fail) → STOP
- `_SCOPE-BREACH_` (touched forbidden file OR Session B's SKILL.md sections) → STOP + revert that change
- Discovery of structural issue (e.g., MCP tool description schema validation fails on new content) → STOP + ping Advisor
- Retry-once gate standard

## Conformance / verification

- All D1-D5 met
- Tests + clippy green
- File-content tests pass
- Phase 3 evidence has count of files modified per runtime + sample description excerpts

## Scope fence

- **IN**: 11 MCP descriptions in mcp_server.rs, "Use These Tools First" section × 3 SKILL.md, preamble intent list, tests, impl-notes
- **OUT**: 3 v0.6.9 MCP tool descriptions (already done), Session B's new SKILL.md sections, dispatch flow examples, multi-turn patterns, validation harness, all forbidden files

## Deliverables (5)

1. `docs/proposals/autoflow-coverage-1-impl-notes.md`
2. mcp_server.rs 11 tool description enhancements
3. SKILL.md × 3 "Use These Tools First" extension
4. Preamble intent-list extension
5. CHANGELOG draft in impl-notes

## Time ceiling

**Hard ceiling: T+5d**. Likely ~3 days per pattern.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + per-runtime file modification summary. Advisor handles Phase C jointly with Session B's branch (ff-merge in safe order, version bump 0.6.9 → 0.6.10, combined CHANGELOG entry).

## Owner gates

Standard. No LLM-test cost in this goal (file edits only). Wave 2 validation goal handles the LLM testing.

## Dependencies

- v0.6.9 main ✓
- Read Session B's briefing first to confirm ownership boundaries

---

— Advisor, 2026-05-19. Wave 1 Session A. Day-0 narrow STOP + retry-once.
