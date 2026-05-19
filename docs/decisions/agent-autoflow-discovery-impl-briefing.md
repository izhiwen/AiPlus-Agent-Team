# Agent Autoflow Discovery — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AGENT-AUTOFLOW-DISCOVERY-1.md`
- Empirical motivation: `docs/proposals/stage-6-userland-mcp-discovery-findings.md` (real codex tests showing MCP bypass)
- Existing aiplus install behavior: read `crates/aiplus-cli/src/main.rs` (`fn handle_install` or similar) + `assets/aiplus-agent-team/` for current runtime adapter file emissions
- Existing MCP tools: `crates/aiplus-cli/src/mcp_server.rs` (14 tools)
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.discovery-layer/`
- Branch:    `feat/discovery-layer`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.discovery-layer -b feat/discovery-layer`

Shared-file ownership:
- This session OWNS: `crates/aiplus-cli/src/main.rs` (install logic extension), `assets/aiplus-agent-team/adapters/<runtime>/` if SKILL.md goes there, new files in `assets/aiplus-token-cost/adapters/<runtime>/` etc., README.md + README.zh-CN.md, new tests
- This session must NOT modify: CONTRACT.md (FROZEN), `crates/aiplus-token-cost/` (subtree mirror), `crates/aiplus-cli/src/agent/coordinator.rs` (scoring rubric), `tests/fixtures/coordinator_calibration.toml` existing entries, adapter code, `crates/aiplus-cli/src/mcp_server.rs` (don't redesign MCP tools; this goal is external discovery), Cargo.toml version, CHANGELOG actual, install.sh
- If discovering that SKILL.md format / runtime mechanism needs significant new design beyond "write a file in the right place with the right content" → STOP and ping Advisor

## Scope — discovery layer in 2 forms

### Form 1: SKILL.md (per-runtime, project-local)

Each runtime has a different mechanism for "session-start instructions the agent reads":
- **Claude Code**: likely `.claude/skills/aiplus/SKILL.md` (Phase 1 confirms exact path)
- **Codex**: managed block in `AGENTS.md` (Phase 1 confirms — Codex's skill-cycling pattern)
- **OpenCode**: likely `.opencode/skills/` or similar (Phase 1 confirms)

Content should be **natural-language, not API-style**. Tell the agent:

> When the user asks about token consumption, USD spending, AI cost, or recent burn — call `agent_token_cost` MCP tool first, not shell grep.
>
> When the user asks for help planning a code task, especially one that sounds non-trivial (involves multiple files, security concerns, refactoring, new features) — call `agent_route_score_only` first to surface the coordinator's would-staffing plan. Then ask user if they want to dispatch.
>
> When the user asks about log integrity, audit, tampering, or wants to verify the dispatch history — call `agent_audit_verify_log`.
>
> When in doubt about which aiplus tool fits a query — list the available `agent_*` MCP tools to the user.

(CEO finalizes wording; above is starting template.)

### Form 2: Project-root preamble (auto-written by `aiplus install`)

Each runtime reads a different project-root file as session context:
- **Claude Code**: `CLAUDE.md` (project root)
- **Codex**: `AGENTS.md` (project root, can use managed-block pattern)
- **OpenCode**: investigate, possibly `.opencode/system-prompt.md` or similar

`aiplus install` should APPEND (not clobber) a managed block that says:

```markdown
<!-- aiplus-discovery-block:start -->
## This project uses AiPlus

AiPlus provides a team of AI agents + 14 MCP tools accessible via `agent_*`
names. When you (the agent) get a user query, prefer these tools over shell
grep / internal-knowledge bypass:

- Cost / spending / token usage questions → `agent_token_cost` 
- Task planning / preview before dispatch → `agent_route_score_only`
- Log integrity / audit / verification → `agent_audit_verify_log`
- Dispatching real work to roles → `agent_route` or `agent_invite`
- Team status / configuration → `agent_status`, `agent_doctor`, `agent_list`

The full list is in `tools/list`. For ANY natural-language query that touches
the agent team, costs, or aiplus features, check the MCP tool list before
falling back to shell.

When dispatching expensive tasks, consider `agent_route_score_only` first
to show the user the planned staffing + estimated cost.
<!-- aiplus-discovery-block:end -->
```

**Idempotency**: detect existing block via sentinels, replace contents on re-install, don't duplicate.

## Phase structure

### Phase 1 — Investigation + Design (~0.5-1 day)

Required impl-notes sections:
1. **Per-runtime mechanism**:
   - Claude Code: where does session context come from? SKILL.md vs CLAUDE.md vs both? File paths.
   - Codex: AGENTS.md managed-block pattern; current aiplus install behavior; how to extend.
   - OpenCode: investigate; document file paths + format.
2. **Current state of `aiplus install` for each runtime** — what files does it write today? List them. Where is the install logic in main.rs?
3. **Design decisions**:
   - SKILL.md content (3 variants if runtimes diverge, 1 shared if they don't)
   - Preamble content for each runtime's project-root file
   - Sentinel markers for managed-block idempotency
   - Detect-existing-file behavior (append vs replace vs refuse)
4. **CHANGELOG draft text** for v0.6.9

### Phase 2 — Implement (~1-1.5 day)

Order:
1. Add SKILL.md content as static assets under `assets/` (one file or three; CEO decides)
2. Extend `aiplus install` runtime-specific logic to emit:
   - SKILL.md to runtime-appropriate path
   - Preamble managed-block into project-root file (CLAUDE.md / AGENTS.md / opencode-equivalent)
3. Idempotency tests (re-run install, verify no duplication)
4. Update README + README.zh-CN with brief mention

### Phase 3 — Live Stage 6 re-test (~0.5 day + ~$3-5 LLM)

Mirror Stage 6 from `stage-6-userland-mcp-discovery-findings.md`:

```bash
# Fresh sandbox
rm -rf /tmp/discovery-test && mkdir -p /tmp/discovery-test/project
cd /tmp/discovery-test/project
git init -q && git commit --allow-empty -m "init / 初始化" -q

# Copy new aiplus binary
cp ~/Projects/AiPlus/aiplus-public.discovery-layer/target/debug/aiplus /tmp/discovery-test/bin/aiplus

# Install all runtimes (which now writes discovery files)
/tmp/discovery-test/bin/aiplus install all --yes

# Isolated CODEX_HOME for safe testing (Bug 4 fix from v0.6.8 makes this honor env now)
mkdir -p /tmp/discovery-test/codex-home
cp ~/.codex/auth.json /tmp/discovery-test/codex-home/auth.json

CODEX_HOME=/tmp/discovery-test/codex-home /tmp/discovery-test/bin/aiplus mcp-register --runtime codex --force

# 3 natural prompts; each codex exec is a fresh session
for prompt in \
  "How much money am I spending on AI tools this week?" \
  "I'm about to implement a payment API for the backend. Can you help me think through this before I start?" \
  "Is my dispatch log intact?"
do
  echo "=== Prompt: $prompt ==="
  CODEX_HOME=/tmp/discovery-test/codex-home \
    aiplus secret-broker run --aliases anthropic,openai -- \
    codex exec --sandbox workspace-write "$prompt" 2>&1 | tail -50
  echo
done
```

**Acceptance gate**: at least **2 of 3 prompts** produce visible MCP tool call (`exec` block showing `aiplus mcp-serve` invocation OR `agent_*` tool name in transcript). Each prompt has retry-once allowance. Document per-prompt outcome in evidence trailer.

If 0/3 or 1/3 → discovery layer didn't work → STOP and surface (may need redesign in next iteration).

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** (existing 579 tests fail) or `_SCOPE-BREACH_` (touched forbidden file)
- **STOP if Phase 3 shows 0/3 or 1/3** → discovery layer insufficient, needs Advisor design rethink
- **STOP if SKILL.md mechanism requires major new infrastructure** (e.g., a new runtime feature negotiation protocol) → ping Advisor; should be just-file-writes
- **Do NOT STOP on `_IMPL-BUG_`** — fix and continue
- Retry-once gate for all FAIL classification AND for Phase 3 LLM stochasticity

## Conformance / verification

- All D1-D5 met
- `cargo test --workspace` PASS (existing 579 + new tests)
- `cargo clippy --workspace --all-targets -- -D warnings` clean
- File-write tests verify SKILL.md + preamble both land at expected paths after install
- Phase 3 live test: 2/3 prompts trigger MCP tool

## Scope fence

- **IN**: SKILL.md content, install logic for preamble emission, README updates, new tests, impl-notes
- **OUT**: CONTRACT.md, adapter code, scoring rubric, calibration fixture, mcp_server.rs (don't redesign tool descriptions — separate concern), `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh
- **GRAY**: small clarification edits adjacent to your additions

## Deliverables (6)

1. `docs/proposals/agent-autoflow-discovery-impl-notes.md` — Phase 1 investigation + design + Phase 3 evidence
2. SKILL.md content (file path TBD per Phase 1) + asset wiring
3. Project-root preamble emission in `aiplus install` (per runtime)
4. README + README.zh-CN brief update
5. New tests covering file-write + idempotency
6. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+3d**. Likely ~2 days per pattern.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + per-runtime status + Phase 3 prompt-by-prompt outcome table (which prompt → which tool called → verdict). Advisor handles Phase C (version bump, CHANGELOG, install.sh parity, retrospective; will also do Advisor's own live re-test of Stage 6 prompts as part of ratification).

## Owner gates

Standard. BWS for Stage 6 testing: wrap with `aiplus secret-broker run --aliases anthropic,openai -- <cmd>`. Phase 3 budget: ~$3-5 of LLM token spend acceptable to Owner per earlier authorization.

**Special**: Phase 3 modifies the user's actual `~/.codex/config.toml` if you forget to set `CODEX_HOME` (Bug 4 from v0.6.8 fixed this BUT only if you remember to use it). Always set `CODEX_HOME=/tmp/discovery-test/codex-home` for any test run.

## Dependencies

- v0.6.8 main ✓ (post-userland-bugfix, with Bug 4 fix making CODEX_HOME isolation work)
- 14 MCP tools registered + working
- aiplus install runtime adapter infrastructure (existing)
- Stage 6 forensic memo (`stage-6-userland-mcp-discovery-findings.md`)

---

— Advisor, 2026-05-19. Day-0 narrow STOP + retry-once. The discovery-layer goal Stage 6 proved necessary.
