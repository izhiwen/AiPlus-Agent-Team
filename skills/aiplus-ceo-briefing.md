# AiPlus CEO Briefing Template Skill

Read this Skill at the start of any new CEO implementation briefing draft. Copy
the template, fill placeholders, then customize the narrow parts for the goal.
Keep the STOP rules explicit and keep file ownership concrete so parallel Codex
sessions can work without stepping on each other.

## Template

```markdown
# <GOAL-NAME> — Implementation Briefing for CEO Codex

**Drafted**: <YYYY-MM-DD> by Advisor
**Target executor**: CEO Codex session (<wave-or-program-context>)
**Authoritative sources**:
- Goal: `docs/proposals/<goal-doc>.md`
- Baseline code: `<primary-code-path-or-commit>`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree: `~/Projects/AiPlus/<repo>.<work-name>/`
- Branch: `<branch-name>`
- Setup:
  `git -C ~/Projects/AiPlus/<repo> worktree add ../<repo>.<work-name> -b <branch-name>`
- All implementation work stays inside this worktree unless a cross-repo file is
  explicitly listed below.

Shared-file ownership (CRITICAL if parallel sessions are active):
- This session OWNS: `<owned-files-or-regions>`
- This session must NOT modify: `<forbidden-files-or-regions>`
- Collision note: `<how this session avoids adjacent shared-file work>`

## Scope

### Sub-feature 1 (<D1-ID>): <title>

**Current state**: <what exists>
**Target state**: <what should exist>
**Care points**:
- <risk-or-compatibility-note>
- <test-cascade-note>

### Sub-feature 2 (<D2-ID>): <title>

<repeat as needed>

## Phase structure

### Phase 1 — Write `<impl-notes-path>` BEFORE coding

Required sections:
1. <design-question-1>
2. <design-question-2>
3. <verification-plan>
4. CHANGELOG draft text for <version-or-release>

### Phase 2 — Implement

Order:
1. <smallest-or-highest-blast-radius-first-step>
2. <next-step>
3. <tests-after-each-step-or-known-exception>

### Phase 3 — Evidence

- `<test-command>` PASS
- `<lint-command>` PASS, if applicable
- Live smoke: `<command-or-workflow>`
- Scope invariant: `<forbidden-file>` unchanged
- Evidence trailer appended to `<impl-notes-path>`

## STOP rule

- STOP only on `_REGRESSION_` (existing non-owned tests fail after retry) or
  `_SCOPE-BREACH_` (touched forbidden files / crossed Owner gate).
- Do NOT STOP on `_IMPL-BUG_`; fix and continue.
- Retry-once gate is mandatory before classifying a failure.
- Special semantic-change rule: <goal-specific STOP, such as "lint cleanup must
  not change runtime behavior" or "CONTRACT-BUG stops immediately">.

## Scope fence

- IN: `<allowed-files-and-behaviors>`
- OUT: `<forbidden-files-and-behaviors>`
- GRAY: `<allowed-with-care-items>`

## Deliverables

1. `<impl-notes-path>` with Phase 1 design and Phase 3 evidence
2. `<code-artifact-1>`
3. `<tests-or-fixtures>`
4. `<docs-or-skill-artifact>`
5. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+<duration>**. This is a hard stop, not a target.

## Handoff endpoint

Report:
- `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}`
- Per-sub-feature status
- Test/lint counts
- Deviations and known gaps
- Whether Advisor or Owner action is needed

Advisor coordinates Phase C / merge / release ratification unless this briefing
explicitly says otherwise.

## Owner gates

Standard gates remain active:
- No push to main unless Owner explicitly authorizes it for this task.
- No publish, deploy, release tag, version bump, or CHANGELOG promotion unless
  listed as in scope.
- No global config edits.
- No secret exposure or external account mutation.
- For BWS-backed runtime keys, use:
  `aiplus secret-broker run --aliases <aliases> -- <cmd>`
  and do not use `secret-broker need` for BWS.

## Dependencies

- <dependency-1>
- <dependency-2>
- <parallel-session-or-merge-order-dependency>

---

-- Advisor, <YYYY-MM-DD>.
```

## Drafting Checks

- Name every owned file or region concretely.
- Name every forbidden file that parallel work could plausibly touch.
- Put the impl-notes requirement before any coding phase.
- Classify STOP rules narrowly; do not make ordinary implementation bugs
  blocking.
- Include retry-once language when test or runtime flakiness is plausible.
- Put exact command examples in verification, not only prose.
- Say who receives the handoff: Advisor, Owner, or another named role.
