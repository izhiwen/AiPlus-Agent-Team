# Goal G-AT-AGENT-AUTOFLOW-DISCOVERY-1 — Make agent auto-discover aiplus MCP tools from natural language

**Status**: PROPOSED — Owner-authorized 2026-05-19 (Discovery comes after bugfix per priority decision).
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AGENT-AUTOFLOW-DISCOVERY-1`

---

## TL;DR

Stage 6 userland testing (forensic: `docs/proposals/stage-6-userland-mcp-discovery-findings.md`) empirically proved that even with all 14 MCP tools registered and well-described, real LLM agents (codex 0.130.0 tested) **prefer shell + internal knowledge over MCP tools** when given natural user prompts. The 3 new MCP tools (`agent_token_cost`, `agent_audit_verify_log`, `agent_route_score_only`) shipped in v0.6.7 are effectively unreachable by users in normal conversation.

This goal adds a **discovery layer** that tells the agent WHEN to prefer aiplus MCP tools over shell/built-in alternatives:
- **Option B**: SKILL.md content per runtime (Claude Code / Codex / OpenCode) that gets read at session start
- **Option C**: `aiplus install` writes a project-level preamble (`CLAUDE.md` / `AGENTS.md` / equivalent) that becomes part of system context

Acceptance is empirical: re-run Stage 6 natural-language prompts after this ships and verify agent **calls the right MCP tool** instead of bypassing.

## Success criterion (single sentence)

> After this goal ships, a user in a fresh `aiplus install all`-equipped project, opening a Claude Code / Codex / OpenCode session, asking "How much money am I spending on AI tools this week?" (cost) or "I'm about to implement a payment API — anything I should think about?" (planning) or "Is my dispatch log intact?" (audit), gets the agent calling the relevant aiplus MCP tool (`agent_token_cost`, `agent_route_score_only`, `agent_audit_verify_log` respectively) and surfacing the structured result, **without the user ever mentioning "MCP" or "aiplus" by name**.

## Goals (D1-D5)

- **D1**: SKILL.md content authored for each of 3 runtimes (Claude Code, Codex, OpenCode) describing the 3 new MCP tools' triggering conditions in natural-language terms. Files land at runtime-appropriate paths during `aiplus install`.
- **D2**: `aiplus install` writes a project-level preamble file (one per runtime: `CLAUDE.md` for Claude Code project root, `AGENTS.md` or augmentation for Codex, runtime-appropriate for OpenCode) telling the agent "this project uses AiPlus; prefer these MCP tools for these query types".
- **D3**: Existing 11 MCP tools also benefit — the preamble briefly explains all 14 (with emphasis on the 3 new ones that need it most).
- **D4**: Live Stage 6 re-test after impl: at least **2 of 3 natural prompts** trigger the expected MCP tool on first try (one full pass through codex; LLM stochasticity allows 1/3 retry-once latitude).
- **D5**: Documentation: README "What you get" + "Daily commands" sections mention that the agent auto-recognizes natural-language queries.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/agent-autoflow-discovery-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/agent-autoflow-discovery-impl-notes.md` (Phase 1 + Phase 3) | PENDING — CEO |
| 4 | SKILL.md content × 3 runtimes (file paths to be determined in Phase 1) | PENDING — CEO |
| 5 | `aiplus install` write logic: emit `CLAUDE.md` / `AGENTS.md` / OpenCode equivalent into project root | PENDING — CEO |
| 6 | README + README.zh-CN: brief mention of natural-language discovery | PENDING — CEO |
| 7 | New tests: file write happens during install + content has expected shape | PENDING — CEO |
| 8 | Phase 3 evidence: real codex run for ≥ 2 of 3 prompts succeeding | PENDING — CEO |

## Phases

```
Phase 1 (CEO, ~0.5-1 day, INVESTIGATION-HEAVY):
  - Read each runtime's documentation / existing aiplus install behavior to
    learn: where does SKILL.md live for Claude Code? for Codex? for OpenCode?
    What's the format? What's already there from aiplus install today?
  - Read aiplus install code to see what it currently writes for each runtime
  - Design SKILL.md content (3 sections per runtime: when to use each new tool)
  - Design CLAUDE.md / AGENTS.md / opencode equivalent preamble content
  - Plan file writes + idempotency
  - Test plan

Phase 2 (CEO, ~1-1.5 day):
  - Write SKILL.md × 3 (or runtime-equivalent locations)
  - Extend aiplus install to emit project-root preamble
  - Update aiplus-module.json schema if needed for SKILL.md asset declaration
  - Write tests (file-write + content shape, NOT live LLM call)

Phase 3 (CEO, ~0.5 day + ~$3-5 LLM cost):
  - Live Stage 6 re-test: real codex exec --sandbox workspace-write with
    3 natural prompts; observe MCP tool calls
  - At least 2/3 prompts must trigger expected MCP tool
  - Document each result in evidence trailer
  - cargo test --workspace + cargo clippy clean
```

**Total target**: ≤ 3 days. Likely ~2 days per pattern.

## Out of scope

- **Slash commands** like `/aiplus token-cost` — separate ergonomic feature, future goal if useful
- **Auto-update of preamble** when running new `aiplus install` versions — Phase C iteration if needed
- **Tool description improvements** in `mcp_server.rs` — small enhancement possible, but the spec focuses on external discovery layer
- **Per-runtime UX tuning beyond SKILL.md + preamble** — keep scope tight
- CONTRACT.md, adapter code, scoring rubric, calibration fixture, `crates/aiplus-token-cost/`, version field, install.sh fallback (Advisor Phase C)

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Phase 1 investigation reveals SKILL.md format varies wildly across runtimes → can't unify | Document per-runtime decision in impl-notes; acceptable to ship 3 different file shapes if behavior is equivalent |
| `aiplus install` writes preamble that clobbers user's existing CLAUDE.md / AGENTS.md | Detect existing file, append managed-block (similar to how Codex AGENTS.md managed-block pattern works today) or refuse to clobber + warn |
| Stage 6 prompts don't trigger MCP tools even with discovery layer | (a) iterate on SKILL.md / preamble language (b) STOP after 1 day of tuning and surface to Advisor — may need to redesign |
| LLM stochasticity makes ≥ 2/3 acceptance hard to verify deterministically | Run each prompt 2× with retry-once allowance; document outcomes; if 1 prompt consistently fails AND analysis shows it's expected (e.g., score_only is genuinely lower-discoverability), accept 2/3 |
| User's existing CLAUDE.md / AGENTS.md gets aiplus content overlaid in confusing way | Phase 1 design includes "first-run vs upgrade" behavior; idempotency + clear sentinel markers like `<!-- aiplus-managed-block -->...<!-- /aiplus-managed-block -->` |
| Discovery layer doesn't actually move the needle and codex still bypasses | Phase 3 catches this empirically; STOP and surface — that means MCP tool descriptions need redesign, separate goal |

## Acceptance criteria

- ✓ D1-D5 met
- ✓ `aiplus install all` on fresh project writes the discovery layer files (SKILL.md + project-root preamble) per runtime
- ✓ Idempotency: re-running `aiplus install` doesn't duplicate or clobber
- ✓ Live Stage 6 re-test: cost prompt + planning prompt + audit prompt → at least 2/3 produce correct MCP tool call on first run (with one retry-once latitude)
- ✓ `cargo test --workspace` PASS
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` clean
- ✓ Phase 3 evidence in impl-notes includes a per-prompt outcome table (tool called / response excerpt / verdict)

## Dependencies

- v0.6.8 main ✓ (post-userland-bugfix)
- 14 MCP tools (11 existing + 3 from v0.6.7) registered and functional
- Stage 6 forensic findings (`stage-6-userland-mcp-discovery-findings.md`) as design input
- aiplus install infrastructure that already writes runtime adapter files

## Why this goal exists (forensic)

Owner asked 2026-05-19: "用户可以使用自然语言。然后 agent 判断用户意图，然后主动或自动帮助用户调用 aiplus 的各个功能和 agent. 我希望实现的是这种。请问我们实现了吗？" Stage 6 empirical answer: NOT IMPLEMENTED. MCP plumbing alone is not sufficient. Real codex bypasses MCP for natural prompts and uses shell + internal knowledge.

This goal closes that gap. After it ships, the answer to Owner's question should be YES (empirically verified by Stage 6 re-test).

## Next action

Owner reads goal + briefing → ack → Advisor dispatches CEO.

---

— Advisor, 2026-05-19, the discovery-layer goal Stage 6 proved was necessary.
