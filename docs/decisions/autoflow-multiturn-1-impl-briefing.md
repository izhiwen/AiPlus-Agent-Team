# Autoflow Multi-turn — CEO Session B Briefing

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session B (parallel with Session A)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AUTOFLOW-MULTITURN-1.md`
- Companion parallel goal (Session A): `goal-G-AT-AUTOFLOW-COVERAGE-1.md` — read for ownership boundary
- v0.6.9 baseline: `aiplus-public@e6f3870`
- v0.6.9 SKILL.md content in `assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.autoflow-multiturn/`
- Branch:    `feat/autoflow-multiturn`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.autoflow-multiturn -b feat/autoflow-multiturn`

## CRITICAL — shared-file ownership matrix (Wave 1 parallel with Session A)

Session A (G-AT-AUTOFLOW-COVERAGE-1) runs concurrently. You both touch SKILL.md × 3 and install-logic preamble.

### Files YOU OWN
- `assets/aiplus-agent-team/adapters/<runtime>/skills/aiplus/SKILL.md` × 3 — **only** append two new sections at end:
  - `## Dispatch Flow` (between existing "Example Flows" and end of file)
  - `## Multi-turn Patterns` (after Dispatch Flow)
- `crates/aiplus-cli/src/main.rs` install logic — only the **dispatch-flow paragraph** appended to the preamble managed-block content string. Append-only.

### Files Session A OWNS — DO NOT TOUCH
- `mcp_server.rs` (any 11 existing tool descriptions)
- SKILL.md × 3 "Use These Tools First" section (A extends bullets)
- Preamble managed-block intent list (A extends bullets)

### Files BOTH must NOT modify
- CONTRACT.md (FROZEN)
- `crates/aiplus-token-cost/` (subtree mirror)
- `coordinator.rs` scoring rubric
- Calibration fixture existing entries
- Adapter code
- The 3 v0.6.9-shipped MCP tool descriptions
- The 3 v0.6.9-shipped SKILL.md sections (existing "Prefer MCP Over CLI" + "Example Flows" + "Avoid Bypass")
- Cargo.toml version
- CHANGELOG actual
- install.sh fallback

### Conflict prevention
- If you find your work needs to touch A's regions: STOP and ping Advisor
- Each session pushes own branch only
- Advisor handles cross-branch ff-merge order at Phase C

## Scope — dispatch flow + multi-turn patterns

### SKILL.md new "Dispatch Flow" section

Append after existing "Avoid Bypass" section in each runtime's SKILL.md. Suggested content (CEO finalizes wording):

```markdown
## Dispatch Flow (multi-step)

When the user describes a non-trivial coding task, follow this 4-step flow
rather than answering immediately:

### Step 1 — Preview the task
Call `agent_route_score_only` with `task="<user's task>"`. Surface to user:
- Tier (LIGHT_NO_CODE / LIGHT_CODE / MEDIUM / HEAVY)
- Complexity (1-5) + risk (0.0-1.0)
- Would-staff list (which roles the coordinator would assign)
- Any auto-summoned domain experts (e.g., security-reviewer)

### Step 2 — Confirm with user
Ask: "Would you like me to dispatch this work to the staffed team? 
(Estimated tier: <X>, would staff: <roles>)"

### Step 3 — Dispatch on confirm
On user "yes" / "go" / equivalent: call `agent_route` with the task. Each
staffed role spawns in its own worktree.

### Step 4 — Integrate when done
After roles complete: call `agent_integrate <role>` per role to merge work
back. Verify with `agent_status` that all work is merged.

### Example
User: "Help me refactor the user authentication module to support OAuth2."
You (turn 1):
  → call `agent_route_score_only` task="refactor user auth for OAuth2"
  → response: "Coordinator scored this MEDIUM tier (complexity 3, risk 0.6).
    Would staff: [engineer-a, reviewer, security-reviewer (auto-summoned)].
    Proceed?"
User (turn 2): "Yes."
You (turn 2 response):
  → call `agent_route` task="refactor user auth for OAuth2"
  → response: "Dispatched. Engineer-a starting in worktree
    ../<project>.engineer-a; reviewer + security-reviewer will run in
    parallel. Watch dispatch-log for progress."
User (turn 3, later): "How's it going?"
You: → call `agent_status` → report each role's state
```

### SKILL.md new "Multi-turn Patterns" section

Append after Dispatch Flow:

```markdown
## Multi-turn Patterns

### Pattern: follow-up cost question
Turn 1 user: "How much have I spent today?"
Turn 1 you: → call agent_token_cost; report total
Turn 2 user: "What about by role?"
Turn 2 you: → call agent_token_cost with by_role=true; report per-role

You do NOT need to grep dispatch-log between turns — each MCP call gets
fresh data.

### Pattern: mid-flight change
Turn 1 user: "Plan a payment API for me"
Turn 1 you: → score_only → surface plan → "Proceed?"
Turn 2 user: "Actually, change it to a refunds API instead."
Turn 2 you: → score_only with NEW task → surface new plan
  (do NOT proceed with old plan; the user changed scope)

### Pattern: ambiguous intent
User: "Audit my project"
Multiple tools could apply (agent_doctor, agent_audit_verify_log,
agent_status). Ask user which kind of audit:
- "code/runtime health" → agent_doctor
- "dispatch log integrity" → agent_audit_verify_log
- "current team / work state" → agent_status

If unsure between two, list them and ask user to pick. Do NOT silently
call the wrong one.
```

### Project-root preamble append

In the `<!-- aiplus-discovery-block -->` managed-block in CLAUDE.md / AGENTS.md / `.opencode/instructions/aiplus.md`, append a short paragraph after Session A's intent list:

```markdown
**Dispatch flow**: For non-trivial coding tasks, follow this sequence rather
than answering directly: (1) call `agent_route_score_only` to preview
staffing, (2) surface to user and ask confirm, (3) on confirm call
`agent_route`, (4) when done call `agent_integrate <role>`. Example: user
says "refactor user auth"; you score first, show plan, then dispatch.
```

Keep ≤ 5 lines so the preamble stays compact.

## Phase structure

### Phase 1 — Write `docs/proposals/autoflow-multiturn-1-impl-notes.md` BEFORE coding

Required:
1. Confirm Session A's ownership boundaries (read A's briefing)
2. Final wording for "Dispatch Flow" section
3. Final wording for "Multi-turn Patterns" section
4. Final preamble append paragraph
5. Idempotency: re-running install doesn't duplicate the appended sections
6. Test plan
7. CHANGELOG draft text

### Phase 2 — Implement

Order:
1. SKILL.md × 3 append two new sections
2. main.rs preamble append paragraph
3. Tests

### Phase 3 — Evidence

- Tests + clippy green
- File content tests assert new sections + dispatch flow paragraph present
- (Advisor + Wave 2 will do live LLM testing; you don't need to do it here)

## STOP rule

- `_REGRESSION_` → STOP
- `_SCOPE-BREACH_` (touched A's regions OR existing v0.6.9 sections) → STOP + revert
- Idempotency test fails → STOP + ping Advisor
- Retry-once gate standard

## Conformance / verification

- D1-D3 met
- Tests + clippy green
- File content has new "Dispatch Flow" + "Multi-turn Patterns" sections in each SKILL.md
- Preamble managed-block has dispatch-flow paragraph
- Re-running `aiplus install` doesn't duplicate

## Scope fence

- **IN**: SKILL.md × 3 two new sections, preamble dispatch-flow paragraph, tests
- **OUT**: 11 MCP descriptions (A), 3 v0.6.9 sections, intent list (A), all forbidden files

## Deliverables (4)

1. `docs/proposals/autoflow-multiturn-1-impl-notes.md`
2. SKILL.md × 3 new sections
3. Preamble dispatch-flow append
4. CHANGELOG draft in impl-notes

## Time ceiling

**Hard ceiling: T+3d**. Likely ~2 days per pattern.

## Handoff endpoint

Report `IMPL_VERDICT={...}` + file modification summary. Advisor handles Phase C jointly with Session A's branch.

## Owner gates

Standard. No LLM-test cost in this goal.

## Dependencies

- v0.6.9 main ✓
- Read Session A's briefing first

---

— Advisor, 2026-05-19. Wave 1 Session B. Day-0 narrow STOP + retry-once.
