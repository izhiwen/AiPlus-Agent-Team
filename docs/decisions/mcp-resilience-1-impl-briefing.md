# MCP Resilience 1 — CEO Investigation Briefing

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session (separate from the Owner's dogfood session)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-MCP-RESILIENCE-1.md`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.mcp-resilience/`
- Branch:    `feat/mcp-resilience-1`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.mcp-resilience -b feat/mcp-resilience-1`

This is investigation-only. The worktree is used to produce a single doc; no source files in `crates/` should be touched.

## Scope — INVESTIGATION ONLY

**Critical**: this is NOT a code-fix goal. You produce a findings doc. NO edits to:

- `crates/**`
- `assets/aiplus-agent-team/**` (adapter templates / SKILL.md)
- `/Users/steve/Projects/MailCue/**` (Owner's project, read-only investigation)
- `~/.aiplus/`, `~/.claude/`, `~/.codex/`, `~/.config/opencode/` (global config, read-only)
- `CHANGELOG.md`, `install.sh`, `Cargo.toml` version

If during investigation you find you NEED to edit code to test something, STOP and ask Advisor.

## The observed failure (background)

Owner ran `aiplus` lobby on `/Users/steve/Projects/MailCue`, picked `ceo` + `codex`. CEO acknowledged in character. Owner then asked three questions; the third was:

> "看下我这周烧了多少 token"

CEO responded by running ~20 `rtk` shell commands to dig into `/Users/steve/.codex/state_5.sqlite`, parsing thread token_used values, joining by date and cwd. CEO produced a numeric answer (~739M tokens this week, ~584M for MailCue) — but the work was manual sqlite archaeology, not a call to `agent_token_cost` MCP tool which is exactly what we built for this query.

Codex's MCP startup logs from that session also showed:

```
⚠ The cloudflare-api MCP server is not logged in. ...
⚠ MCP startup interrupted. The following servers were not initialized: xcodebuildmcp
⚠ MCP startup incomplete (failed: cloudflare-api)
```

So aiplus MCP server may or may not have been healthy. Investigate.

## Phase 1 — Investigation steps (HARD CAP 2h)

### Step 1.1 (~15 min) — Read existing evidence

Read-only file inspection (use Read tool, no shell):

1. **Owner's codex session rollout JSONL** —
   `/Users/steve/.codex/sessions/2026/05/19/rollout-2026-05-19T20-50-49-*.jsonl` (or similar timestamp from ~20:50 today, MailCue cwd). Look for:
   - `mcp_list_tools` event: was `agent_token_cost` (or any aiplus_* tool) in the list?
   - Any `mcp_tool_use` events for aiplus tools during the "本周 token" turn
   - Errors / warnings about aiplus MCP server during startup

2. **MailCue's ceo persona file** —
   `/Users/steve/Projects/MailCue/.aiplus/agents/personas/ceo.md`. Verify:
   - Does it include the Discovery v2 SKILL.md preamble ("Use These Tools First" / "Dispatch Flow" sections)?
   - Compare to the canonical version at `~/Projects/AiPlus/aiplus-public/assets/aiplus-agent-team/adapters/codex/skills/aiplus/SKILL.md`

3. **Codex MCP config** —
   `/Users/steve/.codex/config.toml`. Check `[mcp.servers.aiplus]` section:
   - Does it exist?
   - Binary path correct (`/Users/steve/.local/bin/aiplus`)?
   - Args correct (`mcp-server` or whatever current `aiplus mcp-register` writes)?

### Step 1.2 (~30 min) — Build minimal reproduction harness

In a sandbox temp dir (NOT touching Owner's real config), create a fake codex MCP setup:
- Real `aiplus` binary (use `target/release/aiplus` from main)
- Fake/broken cloudflare-api MCP (e.g., a binary that prints to stderr and exits 1)
- Fake/broken xcodebuildmcp MCP (similar)
- Spawn the real codex with this temp config OR write a small Python/Rust script that simulates the MCP discovery protocol

Goal: determine if a broken cloudflare-api MCP causes the aiplus MCP tool list to be hidden from CEO.

You don't have to ship the harness — a one-time investigation script is enough. Just save it under `docs/proposals/` so we can re-run if needed.

### Step 1.3 (~30 min) — Categorize

Use the 5-category table in goal doc. Pick exactly one category with confidence label.

### Step 1.4 (~15 min) — Write findings doc

`docs/proposals/mcp-resilience-1-findings.md` per template in goal doc.

### Step 1.5 (~30 min budget for surprise / re-running)

If something doesn't reproduce or evidence is contradictory, you have 30 min of slack. **If you blow past 2h total, STOP** — log category (E) "investigation incomplete after 2h, here's what I tried" and call it done. Half-evidence is better than chasing endlessly.

## STOP rule

- **2-hour hard wall** — at 2h elapsed, you STOP regardless of progress and write findings as-is with category (E) if undetermined.
- **Scope-breach** (touched forbidden file) → STOP immediately.
- **Need to edit code to investigate** → STOP, escalate to Advisor.
- Retry-once gate standard.

## Conformance / verification

- Findings doc exists at the path specified
- Root cause categorized A/B/C/D/E with confidence label
- Evidence table populated
- Follow-up goal scope (1 paragraph) included
- NO code/config changes anywhere

## Scope fence

- **IN**: read-only inspection of rollout JSONL, MailCue persona file, codex config, aiplus binary behavior; building investigation-only repro harness; writing findings doc
- **OUT**: all code/config edits (see top-of-briefing)
- **GRAY**: creating throwaway script files in `docs/proposals/` for the harness — OK as long as they're documentation/repro, not source

## Deliverables (1)

1. `docs/proposals/mcp-resilience-1-findings.md` — per template

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + brief summary of category + confidence + recommended next goal. Advisor reviews and decides whether to spawn the follow-up.

## Owner gates

Standard. Investigation reads global config files (~/.codex/) but does not modify them.

---

— Advisor, 2026-05-19. Investigation-only. 2h hard cap. Output is a findings doc, not code.
