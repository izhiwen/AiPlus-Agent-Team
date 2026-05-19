# Agent Autoflow Discovery v2 — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor (post v1 STOP)
**Target executor**: CEO codex session
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AGENT-AUTOFLOW-DISCOVERY-2.md`
- v1 code in main at `aiplus-public@651b25b` (the SKILL.md / preamble files YOU will replace)
- v1 empirical findings: in v2 goal doc TL;DR (Advisor's 3 prompts → 1 PASS strict, 2 partial, 0 complete-success)
- Stage 6 original forensic: `docs/proposals/stage-6-userland-mcp-discovery-findings.md`
- Existing MCP tool definitions: `crates/aiplus-cli/src/mcp_server.rs`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.discovery-v2/`
- Branch:    `feat/discovery-v2`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.discovery-v2 -b feat/discovery-v2`

Shared-file ownership:
- This session OWNS: `crates/aiplus-cli/src/main.rs` (install logic), `crates/aiplus-cli/src/mcp_server.rs` (**NEWLY IN SCOPE for this goal** — 3 tool descriptions only, do NOT touch existing 11 tool definitions or core MCP server logic), `assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md` (× 3 files, REPLACE v1 content), README.md + README.zh-CN.md, new tests
- This session must NOT modify: CONTRACT.md (FROZEN), `crates/aiplus-token-cost/`, `coordinator.rs` scoring rubric, calibration fixture, adapter code, the 11 existing MCP tool definitions in `mcp_server.rs`, `Cargo.toml` version, `CHANGELOG.md` actual, `install.sh`
- If you find that "prefer MCP over CLI" framing requires structural changes (e.g., CLI subcommands deprecated, agent_route_score_only renamed, etc.) → STOP and ping Advisor

## v1 root causes you're fixing

Read the v2 goal doc TL;DR for full context. Summary:

1. **Cost prompt**: codex chose `aiplus agent dispatch-history` (CLI) over `agent_token_cost` (MCP). Fix via explicit prefer-MCP-over-CLI language + sharper MCP tool description.
2. **Planning prompt**: codex bypassed aiplus entirely. Fix via concrete dialogue examples in SKILL.md / preamble.
3. **Audit prompt**: codex correctly tried MCP `agent_audit_verify_log`, but codex non-interactive harness cancelled mid-flight. Document as known limitation; multi-runtime test in Phase 3 to confirm it's codex-specific.

## Improvements

### Improvement 1: SKILL.md language upgrade (× 3 runtimes)

Current v1 SKILL.md (from `assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md` in main at `651b25b`) is a bullet list of "intent → tool". Replace with:

**Section A: Prefer MCP over CLI** (new, explicit)

```markdown
## Prefer MCP Tools Over CLI Subcommands

This project ships `aiplus agent <verb>` CLI subcommands AND `agent_*` MCP
tools that serve overlapping use cases. **You should prefer the MCP tools**
because:

- MCP tools return structured JSON you can reason over directly.
- CLI subcommands return plain text that you'd have to parse with shell pipes.
- MCP tools are documented as the "intended programmatic surface"; CLIs are
  documented as the "human-typing surface".

For example:
- ❌ DON'T call `aiplus agent dispatch-history --since-days 7 --json` to
  answer "how much money have I spent". The structured query is
  `agent_token_cost`.
- ✅ DO call `agent_token_cost` MCP tool first.
- After the MCP call returns, you may verify or extend with CLI if needed,
  but MCP is the source of truth.
```

**Section B: Concrete dialogue examples** (new — addresses planning-bypass)

```markdown
## Example Flows

### Example 1 — User asks about spending / cost
User: "How much have I been burning on AI tools lately?"
Your first action: call `agent_token_cost` (with window="1h" / "8h" / "24h" — default all three).
Then: surface the totals + top tasks. Do NOT grep .aiplus/ files.

### Example 2 — User describes a coding task
User: "I'm about to implement a payment API for the backend. Help me think through this."
**Do NOT immediately answer with design checklists from your training data.**
Your first action: call `agent_route_score_only` with task="implement payment API for backend"
Then: surface the coordinator's analysis (complexity, risk, staffing) and ASK whether the user wants to dispatch via `agent_route`, or proceed without dispatching.

This pattern applies to any non-trivial coding task: refactoring, feature implementation, multi-file work. Default to scoring + surfacing.

### Example 3 — User asks about log / dispatch / audit integrity
User: "Is my dispatch log intact?" or "Audit my recent agent work"
Your first action: call `agent_audit_verify_log`.
Then: surface PASS or FAIL (with line number if FAIL).

⚠️ Note for codex non-interactive: MCP calls may be cancelled mid-flight by the
harness. If you see "user cancelled MCP tool call" in your trace, mention this
to the user and offer to retry interactively. This is a codex runtime
limitation, not an aiplus tool issue.
```

**Section C: Anti-bypass reminder** (preserved from v1, sharper)

```markdown
## Avoid Bypass

Do NOT answer aiplus questions by:
- ❌ `rg --files ... | tail -20 | parse JSON manually`
- ❌ Reading dispatch-log.jsonl with shell tools
- ❌ Answering from training data alone when the user asks about *this project's*
  agent team, costs, or dispatch history

The structured MCP tool is the source of truth.
```

### Improvement 2: MCP tool description enhancement (mcp_server.rs)

For each of `agent_token_cost`, `agent_audit_verify_log`, `agent_route_score_only`, prepend the tool's `description` field with:

```
PREFERRED programmatic surface for <intent>. Use this MCP tool instead of `aiplus agent <verb>` CLI subcommands when answering user queries — MCP returns structured JSON you can reason over directly. <existing description>
```

Specifically:
- `agent_token_cost` — prepend prefer-over-CLI line referencing `aiplus agent dispatch-history` as the CLI alternative
- `agent_audit_verify_log` — prepend prefer-over-CLI line
- `agent_route_score_only` — prepend prefer-over-CLI line + a one-line example: "Example: user says 'I'm about to implement X' → call this with task='implement X' BEFORE answering from training data."

Keep description ≤ 400 chars per tool to stay within MCP client expectations.

### Improvement 3: Project-root preamble upgrade

The preamble that `aiplus install` writes into `CLAUDE.md` / `AGENTS.md` / opencode-equivalent. Replace the v1 generic intent-list with v2's:

```markdown
<!-- aiplus-discovery-block:start -->
## This project uses AiPlus

When the user asks about token costs, agent team status, dispatch history, or
wants to plan a coding task, **prefer aiplus `agent_*` MCP tools** over shell
grep or `aiplus agent <verb>` CLI subcommands. MCP returns structured JSON.

**Most important patterns:**

- "How much am I spending" / cost / burn / USD → `agent_token_cost`
- "Help me plan / implement X" → `agent_route_score_only` FIRST, then surface to user
- "Is my log intact" / audit / verify → `agent_audit_verify_log`

For coding tasks: do NOT answer from training data first. Score the task via
`agent_route_score_only`, surface the result, then ask user whether to dispatch.

Full tool list: 11 existing `agent_*` tools + 3 from v0.6.7. Run `tools/list`
to enumerate.
<!-- /aiplus-discovery-block -->
```

## Phase structure

### Phase 1 — Write `docs/proposals/agent-autoflow-discovery-2-impl-notes.md` BEFORE coding

Required sections:
1. Confirm v1 file paths (`assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md` × 3)
2. Final SKILL.md content per runtime (drafted with the 3 sections above as starting templates; you may adjust wording but keep the structural elements)
3. MCP tool description final form for 3 tools (within 400 chars each)
4. Project-root preamble final form
5. opencode Phase 3 setup: how to isolate opencode config, how to register MCP, how to send a non-interactive prompt
6. Test plan
7. CHANGELOG draft text for combined v0.6.9 ship (which will include both v1 + v2 work as "discovery layer landing")

### Phase 2 — Implement

Order:
1. SKILL.md upgrades (3 files) — easy, file replacements
2. MCP tool description enhancement (3 entries in mcp_server.rs)
3. Project-root preamble upgrade (modify the string template in main.rs)
4. Tests for file content + tool description

### Phase 3 — Multi-runtime Phase 3 (~1 day + ~$8-12 LLM)

```bash
# Fresh sandbox
rm -rf /tmp/discovery-v2-ratify && mkdir -p /tmp/discovery-v2-ratify/{bin,project,codex-home}
cp ~/Projects/AiPlus/aiplus-public.discovery-v2/target/debug/aiplus /tmp/discovery-v2-ratify/bin/aiplus
cp ~/.codex/auth.json /tmp/discovery-v2-ratify/codex-home/auth.json
cd /tmp/discovery-v2-ratify/project
git init -q && git commit --allow-empty -q -m "init / 初始化"
AIPLUS_SKIP_VERSION_CHECK=1 /tmp/discovery-v2-ratify/bin/aiplus install all --yes

# Configure isolated CODEX_HOME (Bug 4 fix lets this work)
CODEX_HOME=/tmp/discovery-v2-ratify/codex-home /tmp/discovery-v2-ratify/bin/aiplus mcp-register --runtime codex --force

# Codex 3 prompts
for prompt in \
  "How much money am I spending on AI tools this week?" \
  "I'm about to implement a payment API for the backend. Can you help me think through this before I start?" \
  "Is my dispatch log intact?"
do
  echo "=== CODEX === $prompt"
  CODEX_HOME=/tmp/discovery-v2-ratify/codex-home \
    aiplus secret-broker run --aliases anthropic,openai -- \
    codex exec --sandbox workspace-write "$prompt" 2>&1 | tail -50
done

# OpenCode 3 prompts (Phase 1 investigates the equivalent for opencode)
# Likely:
#   - opencode-equivalent of CODEX_HOME isolation
#   - opencode equivalent of `mcp-register`
#   - opencode non-interactive exec (e.g., `opencode run`)
# Document the actual commands in Phase 3 evidence
```

**Acceptance gate**: ≥ 4 of 6 prompts trigger expected MCP tool (counting "MCP triggered then cancelled by harness" as PASS for codex-specific cases).

If 0/6, 1/6, 2/6, or 3/6 → STOP and surface to Advisor. Discovery v2 didn't move the needle; need rethink.

If 4-5/6 PASS → IMPL_VERDICT=PASS_WITH_DEVIATIONS (with the failed cases analyzed).

If 6/6 PASS → IMPL_VERDICT=PASS clean.

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** (existing 580 tests fail) or `_SCOPE-BREACH_` (touched forbidden file)
- **STOP if Phase 3 ≤ 3/6 PASS** → discovery layer fundamentally insufficient; needs Advisor design rethink (slash commands? aggressive auto-route?)
- **STOP if MCP tool description changes break MCP client tool listing** → revert + ping Advisor
- **Do NOT STOP on `_IMPL-BUG_`** — fix and continue
- Retry-once gate for all FAIL classification AND Phase 3 LLM stochasticity

## Conformance / verification

- All D1-D4 met
- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- File-content tests verify SKILL.md has key sections (prefer-MCP-over-CLI, example flows)
- MCP tool descriptions ≤ 400 chars each
- Phase 3 multi-runtime evidence with per-prompt outcomes

## Scope fence

- **IN**: SKILL.md content × 3 runtimes, 3 MCP tool descriptions in `mcp_server.rs`, preamble emission logic in `main.rs`, README × 2, new tests, impl-notes
- **OUT**: CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, the 11 existing MCP tool definitions, MCP server core logic (just enhance descriptions on 3 tools, don't restructure), Cargo.toml version, CHANGELOG actual, install.sh
- **GRAY**: small comment improvements adjacent to edits

## Deliverables (5)

1. `docs/proposals/agent-autoflow-discovery-2-impl-notes.md` — Phase 1 design + Phase 3 evidence
2. SKILL.md × 3 runtimes (replace v1 content)
3. MCP tool descriptions enhanced for 3 tools in `mcp_server.rs`
4. Project-root preamble emission upgraded in `main.rs`
5. Tests + CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+3d**. Likely ~2 days actual.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + multi-runtime Phase 3 outcome table (6 rows: runtime × prompt × tool-called × verdict). Advisor handles Phase C (independent live re-test mirror, version bump 0.6.8 → 0.6.9, CHANGELOG covering BOTH v1 AND v2 work, install.sh parity, retrospective folding both goals).

## Owner gates

Standard. BWS for LLM tests. Phase 3 budget ~$8-12 acceptable per Owner authorization. **Always set isolated CODEX_HOME / opencode-equivalent**; never write to user's real config.

## Dependencies

- v0.6.8 main + v1 discovery code at `651b25b` ✓
- v1 retrospective findings (in v2 goal doc TL;DR)
- 14 MCP tools registered ✓

---

— Advisor, 2026-05-19. v2 redesign per v1 STOP findings. Day-0 narrow STOP + retry-once.
