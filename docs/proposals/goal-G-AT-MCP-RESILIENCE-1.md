# Goal G-AT-MCP-RESILIENCE-1 — Diagnose discovery-bypass when codex MCP startup is degraded

**Status**: PROPOSED — Owner-authorized 2026-05-19 after observing real Owner dogfood transcript where CEO bypassed `agent_token_cost` MCP tool and instead did 20+ rtk commands to manually parse `~/.codex/state_5.sqlite`.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-MCP-RESILIENCE-1`
**Mode**: **Investigation-only**. NO code fixes in this goal. Output is a findings doc that Advisor uses to decide the appropriate follow-up goal.

---

## TL;DR

In a real Owner dogfood session on MailCue project (v0.6.11 lobby), CEO asked "看下我这周烧了多少 token" — and instead of calling `agent_token_cost` MCP tool (which was built exactly for this query), CEO ran 20+ rtk commands manually digging into codex's local sqlite. This is a **discovery layer failure** worth investigating before we ship more MCP tools that nobody triggers.

Codex's MCP startup output during that session showed:

```
⚠ The cloudflare-api MCP server is not logged in. Run `codex mcp login cloudflare-api`.
⚠ The cloudflare-api MCP server is not logged in. Run `codex mcp login cloudflare-api`.
⚠ MCP startup interrupted. The following servers were not initialized:
  xcodebuildmcp
⚠ MCP startup incomplete (failed: cloudflare-api)
```

This is the *suspect* environment in which `agent_token_cost` didn't fire. Was aiplus MCP server even visible to CEO? Was its tool descriptions readable? Did the SKILL.md preamble in MailCue's persona fire the natural-language route?

This goal is **investigation only**: identify the root cause, categorize, and document. Fixes are out of scope and will be spawned as separate goals based on what we find.

## Success criterion (single sentence)

> A diagnostic findings document at `docs/proposals/mcp-resilience-1-findings.md` that categorizes the root cause as exactly one of (A) codex MCP startup cascade, (B) MailCue persona out of date, (C) SKILL.md / tool description gap, (D) aiplus mcp-register issue, or (E) other — with evidence for each, and a recommended follow-up goal scope.

## Goals (D1)

- **D1**: Findings doc with: root cause category (A/B/C/D/E), evidence table, recommended next goal scope (or "no fix needed, doc the workaround"), and a minimal reproduction harness if applicable.

## Out of scope

- ANY code change to aiplus-cli, agent-team adapters, SKILL.md, persona templates, or scoring logic
- Modifying MailCue project files (it's Owner's separate project, read-only for our investigation)
- Modifying global `~/.codex/config.toml` (Owner gates: no global config edits)
- Modifying `~/.aiplus/` global state
- Tagging, version bumping, CHANGELOG, install.sh

## Phase 1 — Investigation (HARD CAP 2 hours)

### Step 1.1 — Read evidence from Owner's actual session (~15 min)

Sources (read-only, no edits):
- `/Users/steve/.codex/sessions/2026/05/19/rollout-*.jsonl` files from the lobby session — search for `mcp_list_tools`, `mcp_tool_use`, error events
- `/Users/steve/Projects/MailCue/.aiplus/agents/personas/ceo.md` — verify what persona content CEO actually loaded
- `/Users/steve/.codex/config.toml` — verify which MCP servers are configured and how aiplus is registered

Expected outputs:
- Was `agent_token_cost` tool advertised to CEO? (search rollout JSONL for tool list event)
- Was the SKILL.md preamble in MailCue's CEO persona the current Discovery v2 version? (compare to `assets/aiplus-agent-team/adapters/codex/skills/aiplus/SKILL.md` in aiplus-public)
- Was the aiplus MCP server `mcp.servers.aiplus` entry healthy in config.toml? (look for stderr/socket errors)

### Step 1.2 — Build minimal reproduction harness (~30 min)

Create a temp sandbox that replicates the degraded MCP environment:
- Fake codex MCP config with: aiplus + cloudflare-api (intentionally broken) + xcodebuildmcp (intentionally broken)
- Launch a codex-MCP-client-equivalent (or use the actual codex binary) and observe whether aiplus tools are listed
- Variation: vary order, vary which broken MCP fails first

Output: harness script + observation log.

### Step 1.3 — Categorize root cause

Match findings to ONE of these categories. Provide evidence for each:

| Category | Indicator | Evidence to record |
|---|---|---|
| **(A) codex MCP startup cascade** | Other failing MCPs prevent aiplus from registering, OR aiplus tools missing from CEO's tool list | Tool list event from rollout JSONL, stderr from codex MCP startup, reproduction harness output |
| **(B) MailCue persona out of date** | MailCue's `.aiplus/agents/personas/ceo.md` lacks Discovery v2 SKILL.md preamble | Diff between MailCue's ceo.md and current adapter template |
| **(C) SKILL.md / tool description gap** | Persona is current AND tools are advertised, but "本周烧了多少 token" doesn't match SKILL.md patterns or tool descriptions | Pattern grep on SKILL.md + agent_token_cost tool description |
| **(D) mcp-register issue** | aiplus MCP server isn't registered in codex config at all, or wrong runtime, or wrong binary path | `~/.codex/config.toml` mcp.servers section |
| **(E) other** | None of the above | Justify what category is needed |

### Step 1.4 — Recommend follow-up

Based on category:
- **(A)**: Document workaround (`codex mcp login cloudflare-api` or remove broken servers) + propose long-term aiplus-side mitigation if any (e.g., wrapper that retries MCP discovery)
- **(B)**: Propose Goal `G-AT-PERSONA-REFRESH-1` to add persona-update mechanism
- **(C)**: Propose Goal `G-AT-DISCOVERY-V3` to extend SKILL.md patterns for token-cost queries
- **(D)**: Propose Goal `G-AT-MCP-REGISTER-FIX-1` to fix registration bug
- **(E)**: Custom recommendation

## Phase 2 — Findings doc

Output: `docs/proposals/mcp-resilience-1-findings.md` with:

```markdown
# MCP Resilience 1 — Findings

## Root cause
**Category: (one of A/B/C/D/E)**
**Confidence: high / medium / low**

## Evidence
[table of evidence per category considered]

## Reproduction
[harness script + reproducible commands, OR "could not reproduce, found existing evidence in <path>"]

## Recommended follow-up
[name of next goal, brief scope, time estimate, whether Owner should authorize now or after dogfood]

## Open questions for Advisor
[anything that needs Advisor / Owner decision]
```

## Acceptance criteria

- ✓ Findings doc exists and follows the template
- ✓ Root cause categorized with evidence
- ✓ Recommended follow-up goal name + 1-paragraph scope
- ✓ NO code changes anywhere in aiplus-public, aiplus-agent-team, MailCue, or global config
- ✓ Hard cap 2h respected; if 2h elapsed without categorization, STOP with category (E) and document what was tried

## Why this exists

Discovery layer + MCP tools are the entire point of Wave 1 / Discovery v2 work. If they don't fire when Owner actually uses lobby on a real project, the past 3 days of MCP investment have lower ROI than we thought. Before doing more MCP tool work (which would multiply the failure), figure out why this one didn't fire.

Investigation-first because the right *fix* depends on the root cause, and we don't want to fire-and-forget a guess that turns out to address the wrong layer.

---

— Advisor, 2026-05-19. Half-day investigation goal. T+2h Phase 1 hard cap.
