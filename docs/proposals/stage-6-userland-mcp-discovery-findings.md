# Stage 6 — Real-LLM MCP Discovery Findings

**Drafted**: 2026-05-19 by Advisor (during userland-test session, post-Stage-1-through-5)
**Sources**: Real `codex exec` runs in `/tmp/aiplus-userland-test/codex-home` with v0.6.7 binary, isolated CODEX_HOME, valid Owner auth.json copied in, MCP server registered

---

## The empirical question

Owner asked on 2026-05-19: "用户可以使用自然语言。然后 agent 判断用户意图，然后主动或自动帮助用户调用 aiplus 的各个功能和 agent. 我希望实现的是这种。请问我们实现了吗？测试包括这些吗？"

i.e., **does the agent automatically discover + invoke aiplus MCP tools given natural user prompts, without being told to?**

## Setup

- Worktree: `/tmp/aiplus-userland-test/project` (fresh `aiplus install all --yes`)
- Codex 0.130.0 in `--sandbox workspace-write` mode
- `CODEX_HOME=/tmp/aiplus-userland-test/codex-home` (isolated config + copied auth.json)
- `config.toml` registers `[mcp_servers.aiplus] command = "/tmp/aiplus-userland-test/bin/aiplus" args = ["mcp-serve"]`
- v0.6.7 `aiplus mcp-serve` exposes 14 MCP tools including the 3 new ones (`agent_token_cost`, `agent_audit_verify_log`, `agent_route_score_only`)
- Stage 1-5 already confirmed: MCP plumbing works; codex CAN call these tools when explicitly told to
- Stage 6 question: does codex CHOOSE to call them on natural prompts?

## Test 1: cost query

**Prompt**: *"How much money am I spending on AI tools this week?"*

**Codex behavior**: Ran shell `rg --files -g '*cost*' -g '*token*'` to grep `.aiplus/` for relevant filenames; found `dispatch-log.jsonl` and `token-cost-snapshots.jsonl`; ran `tail -n 20` on each; parsed the JSON output by reading text; reported "$0.00" (correct).

**Did NOT call**: `agent_token_cost` MCP tool (which is registered and matches this query perfectly).

**Pattern**: right answer via wrong path. Codex picked the lowest-effort tool (shell + rg) over the structured MCP tool.

## Test 2: pre-flight task question

**Prompt**: *"I'm about to implement a payment API for the backend. Can you help me think through this before I start?"*

**Codex behavior**: Gave a 10-section payment-API design checklist (idempotency, security, atomicity, webhooks, refunds, observability, API shape, pitfalls). Drew entirely from its own training knowledge.

**Did NOT call**: `agent_route_score_only` (would have shown coordinator scoring + would-staff list). Did NOT call `agent_route`. Did NOT mention aiplus agent team exists.

**Pattern**: completely bypassed aiplus, treated user as if there were no aiplus features available.

## Conclusions

1. **MCP plumbing is necessary but NOT sufficient.** Even with 14 tools well-described and discoverable via `tools/list`, a real LLM agent presented with natural user prompts **prefers its own built-in capabilities** (shell + rg, internal knowledge) over MCP tools.

2. **The agent has no concept of "this project has aiplus capabilities I should default to."** There is no system prompt, no SKILL.md, no project-level CLAUDE.md / AGENTS.md instruction telling codex "for AiPlus projects: cost queries route through `agent_token_cost`; planning queries through `agent_route_score_only`; integrity through `agent_audit_verify_log`."

3. **Owner's vision is not implemented.** "用户自然语言 → agent 自动调用 aiplus" requires the agent to KNOW to prefer aiplus tools. That knowledge has to come from somewhere — a SKILL.md, a CLAUDE.md preamble, the MCP tool description being persuasive enough, OR explicit slash commands the user can type.

4. **Tool descriptions alone aren't persuasive enough.** Compare codex's reasoning:
   - "Show token consumption and USD cost rollups for AiPlus dispatch logs" (MCP description)
   - vs. "Look for token-cost files in the project directory" (codex's natural approach)
   
   Both arrive at the same answer but codex's path doesn't trigger the MCP description as preferred.

## Implications for the next goal

A discovery layer is needed:

| Mechanism | Effort | Effect |
|---|---|---|
| **SKILL.md** per runtime (Option B) | ~1 day | Read at session start; tells agent "use these tools for these prompts" |
| **CLAUDE.md / AGENTS.md project preamble** written by `aiplus install` (Option C) | ~0.5-1 day | Becomes part of system prompt; persistent across sessions |
| **Tool description enhancement** — make MCP tool descriptions more "discoverable" | ~0.5 day | Lowest effort, partial effect |
| **Slash commands** like `/aiplus token-cost` | ~1-2 days | User-driven invocation, not auto-discovery |

Recommended: **Option B + Option C together** as G-AT-AGENT-AUTOFLOW-DISCOVERY-1. Description enhancement and slash commands are smaller follow-ups.

## What this validates

Owner's instinct to ask "does it auto-use?" was right and Advisor's Stage 5 theoretical analysis was **wrong** on the HIGH-confidence predictions (predicted `agent_token_cost` would auto-fire for cost queries; empirically it didn't). The cost prompt was the strongest candidate per Stage 5 reasoning, and it still failed. This means the WEAKER candidates (`agent_route_score_only` for planning) certainly need explicit guidance.

**Lesson for future Advisor work**: don't trust theoretical predictions about LLM behavior — run the real test, even at $1-3 cost, before declaring "this should work."

## Cost of this test

Approximately $0.50-1.50 total across the 2 codex exec runs (gpt-5.5, low reasoning effort, short prompts, ~40k tokens combined per run).

---

— Advisor, 2026-05-19, Stage 6 of userland-test session.
