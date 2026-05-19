# Goal G-AT-AGENT-AUTOFLOW-DISCOVERY-2 — Redesign discovery to fix 3 root causes found in v1 re-test

**Status**: PROPOSED — Owner-authorized 2026-05-19 after STOP-triggered Advisor Phase C re-test of v1.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AGENT-AUTOFLOW-DISCOVERY-2`

---

## TL;DR

v1 (G-AT-AGENT-AUTOFLOW-DISCOVERY-1, shipped to main as commit `651b25b` but NOT released as 0.6.9 — STOPPED at Phase C ratification) achieved 1/3 strict MCP-trigger rate in Advisor's independent re-test. 3 distinct root causes diagnosed; this goal redesigns the discovery layer to fix each. Hard ceiling ≤ 3 days.

**v1 empirical data** (recorded in `goal-G-AT-AGENT-AUTOFLOW-DISCOVERY-1-retrospective.md` — will be drafted Phase C of THIS goal as combined retrospective):
- Cost prompt: codex called CLI `aiplus agent dispatch-history` instead of MCP `agent_token_cost`. *Discovery routed agent to aiplus, but to wrong surface.*
- Planning prompt: codex bypassed aiplus entirely, answered from internal knowledge. *Discovery didn't fire at all.*
- Audit prompt: codex called MCP `agent_audit_verify_log` correctly but codex-non-interactive harness cancelled the call. *Discovery worked, runtime cancelled.*

## 3 root causes + v2 fixes

### Cause A: Agent prefers CLI subcommand over MCP tool (cost case)

Both surfaces reach aiplus; codex picked the more "familiar" CLI pattern (`aiplus agent <verb>` is shell-shaped, simple) over the structured MCP call.

**Fix**: SKILL.md + MCP tool descriptions both upgraded to explicitly steer toward MCP:
- SKILL.md: "**Prefer MCP `agent_*` tools over the `aiplus agent <verb>` CLI subcommands**. The MCP tools return structured JSON that you can reason over; CLI returns plain text you have to parse."
- MCP tool description (in `mcp_server.rs` — newly IN SCOPE for this goal): each of the 3 new tools' description starts with "Preferred over `aiplus agent <verb>` CLI for programmatic queries..." and includes one example "when to call" line.

### Cause B: Agent doesn't connect "implement X" → "aiplus can dispatch X" (planning case)

Codex saw "implement payment API" as a knowledge question (answered from training). The link to "aiplus can stage this" is too abstract for the agent to make from current SKILL.md language.

**Fix**: SKILL.md + project-root preamble add **concrete dialogue examples** showing the desired flow:

```
EXAMPLE — when the user wants to start a coding task:

User: "I'm about to implement a payment API for the backend. Help me think through this."

You should NOT immediately answer with design checklists from your training data.
Your FIRST action should be:
    → call agent_route_score_only with task="implement payment API for backend"

The tool returns the coordinator's analysis (complexity, risk, would-staff list).
You then surface this to the user and ask if they want to dispatch the staffed team
via agent_route, or proceed solo.

This pattern applies to ANY non-trivial coding task: refactoring, feature
implementation, bug fixing, security review. Default to scoring + surfacing
before answering from training data alone.
```

### Cause C: Codex non-interactive cancels MCP calls mid-flight (audit case)

This is a `codex exec --sandbox workspace-write` harness bug independent of aiplus. Documented but not fixable in SKILL.md.

**Fix**: 
- Document the limitation in SKILL.md ("Note: codex non-interactive mode currently cancels MCP calls mid-flight; for codex use, prefer interactive mode or `--ask-for-approval untrusted`.")
- **Multi-runtime cross-check** in Phase 3: also test in opencode (which has different MCP execution model). If opencode 3/3 triggers MCP correctly, that proves the discovery layer works and codex non-interactive is the special case.

## Success criterion (single sentence)

> After v2 ships, Advisor's independent multi-runtime re-test produces ≥ **4 of 6 prompts** (codex 3 + opencode 3) where the agent calls the **expected MCP tool** (not CLI fallback, not pure-knowledge-bypass) — with the understanding that codex-non-interactive cancellation may downgrade some codex results from "tool succeeded" to "tool triggered then cancelled" (latter still counts as PASS for D1).

## Goals (D1-D4)

- **D1**: SKILL.md per runtime upgraded with (a) explicit "prefer MCP over CLI" language, (b) ≥ 1 concrete dialogue example per intent (cost / planning / audit), (c) codex-non-interactive limitation note.
- **D2**: MCP tool descriptions in `mcp_server.rs` for `agent_token_cost`, `agent_route_score_only`, `agent_audit_verify_log` enhanced with "preferred over CLI" framing + "when to call" examples. (Newly IN SCOPE for this goal — was OUT in v1.)
- **D3**: Project-root preamble (CLAUDE.md / AGENTS.md / opencode equivalent) gets the same dialogue examples + prefer-MCP-over-CLI framing.
- **D4**: Phase 3 multi-runtime acceptance: codex 3 prompts + opencode 3 prompts → **≥ 4/6** trigger the expected MCP tool (counting "triggered-then-cancelled" as PASS).

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/agent-autoflow-discovery-2-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/agent-autoflow-discovery-2-impl-notes.md` (Phase 1 + Phase 3) | PENDING — CEO |
| 4 | SKILL.md content × 3 runtimes (upgraded; replaces v1) | PENDING — CEO |
| 5 | MCP tool descriptions for 3 new tools enhanced in `mcp_server.rs` | PENDING — CEO |
| 6 | Project-root preamble logic in `aiplus install` upgraded | PENDING — CEO |
| 7 | New tests + multi-runtime Phase 3 evidence | PENDING — CEO |

## Phases

```
Phase 1 (CEO, ~0.5 day):
  - Read v1 retrospective findings (in this goal's TL;DR)
  - Read existing SKILL.md content (v1 in main at 651b25b)
  - Read MCP tool descriptions in mcp_server.rs (currently brief)
  - Design upgraded language: prefer-MCP-over-CLI + dialogue examples
  - Plan opencode test setup (Phase 3 will need this; might be more involved than codex)
  - CHANGELOG draft text

Phase 2 (CEO, ~1-1.5 day):
  - Upgrade 3 SKILL.md files with new language + examples
  - Enhance 3 MCP tool descriptions in mcp_server.rs
  - Upgrade preamble emission in main.rs install logic
  - Tests for the file content (assert key phrases present)

Phase 3 (CEO, ~1 day + ~$8-12 LLM):
  - Set up fresh /tmp sandbox + install + isolated CODEX_HOME
  - Run 3 prompts × codex (3 data points)
  - Set up isolated opencode config (similar pattern; investigate how)
  - Run 3 prompts × opencode (3 data points)
  - Track each: which MCP tool (if any) was triggered, succeed/cancel
  - Acceptance gate: ≥ 4/6 expected MCP triggered
  - Phase 3 evidence trailer
```

**Total target**: ≤ 3 days. Budget cost: ~$8-12 for Phase 3 (6 LLM runs).

## Out of scope

- **Codex non-interactive harness bug fix** — upstream codex issue, not aiplus's to solve. Document only.
- **Slash commands** — separate ergonomic feature, not this goal.
- **Tool description redesign beyond the 3 new tools** — keep scope to v0.6.7 additions; the 11 existing tools' descriptions stay unchanged.
- CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, Cargo.toml version, CHANGELOG actual, install.sh fallback.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Even with v2, codex still picks CLI over MCP | Phase 3 catches this; if pattern persists, root cause is deeper (model-level bias toward shell-style invocation) — surface to Advisor for redesign #3 |
| OpenCode also has MCP execution oddities | Multi-runtime test will surface this; document; doesn't block v2 if codex 3/3 trigger MCP after redesign |
| Dialogue examples in SKILL.md / preamble bloat the file size and dilute key signal | Keep ≤ 2 examples per intent; aim for ≤ 50 lines per SKILL.md |
| MCP tool description enhancements break some downstream consumer that parses descriptions | Description is free-text; no downstream parser depends on exact format |
| Phase 3 multi-runtime takes longer than estimated | Hard ceiling T+3d catches this; if not done, STOP and surface partial data |

## Acceptance criteria

- ✓ D1-D4 met
- ✓ `cargo test --workspace` PASS (580+ tests)
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase 3: 6 prompts across 2 runtimes → ≥ 4/6 trigger expected MCP tool (counting cancel-after-trigger as PASS)
- ✓ Phase 3 evidence has per-prompt-per-runtime outcome table
- ✓ v1 retrospective folded into v2's retrospective at Phase C

## Dependencies

- v1 discovery code in main at `651b25b` ✓
- v0.6.8 features ✓
- Stage 6 forensic memo (`stage-6-userland-mcp-discovery-findings.md`) ✓
- v1 retrospective findings (this goal's TL;DR captures them)

## Why this goal exists (forensic)

Owner asked 2026-05-19 "user 自然语言 → agent 自动用 aiplus". v1 attempted with SKILL.md + project-root preamble; Stage 6 re-test by Advisor showed 1/3 strict MCP-trigger (cost went to CLI, planning bypassed entirely, audit triggered MCP but cancelled by codex harness). Owner picked "STOP per spec strict rule" and authorized v2 design. This goal codifies the redesign per the 3 root causes Advisor diagnosed.

## Next action

Owner reads goal + briefing → ack → Advisor dispatches CEO with v2 opener prompt.

---

— Advisor, 2026-05-19. Post v1-STOP redesign goal.
