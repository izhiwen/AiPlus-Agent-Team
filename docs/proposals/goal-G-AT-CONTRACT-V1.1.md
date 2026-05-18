# Goal G-AT-CONTRACT-V1.1 — Codify Lessons in CONTRACT v1.1

**Status**: PROPOSED — Owner sign-off pending.
**Drafted by**: Advisor (post G-AT-PROD-1 close, 2026-05-18)
**Goal ID**: `G-AT-CONTRACT-V1.1` (Goal, Agent-Team, Contract codification, milestone v1.1)

---

## TL;DR

G-AT-PROD-1 retrospective surfaced 4 architectural lessons currently scattered across `docs/decisions/`. CONTRACT.md is the single source of truth for adapter behavior, but lessons learned about **how to operate the contract** (worktree isolation, painpoint classification, spiral validation minimums, multi-Advisor coordination) live elsewhere. **G-AT-CONTRACT-V1.1 promotes CONTRACT v1 → v1.1 by appending 4 non-normative appendices** that codify these lessons, then re-validates the existing 3 adapters against v1.1 to confirm no regression.

v1.1 is **additive only** — §1-§11 normative sections are not modified. v1 clients (current 3 adapters) MUST still pass v1 conformance against v1.1 without modification.

## Success criterion (single sentence)

> A future Advisor (or future Owner re-onboarding after months away) reading CONTRACT.md alone can correctly run a parallel-session spiral round, classify any painpoint, and coordinate with another Advisor instance — without consulting any external `docs/decisions/` memo.

That sentence captures the "single source of truth" principle: CONTRACT.md should be sufficient for any operator to act correctly, not a stub that requires tribal knowledge.

## Goals (D1-D5)

- **D1**: CONTRACT v1 → v1.1 schema bump in §11 (schema version migration). v1.1 is **minor additive**; v1 clients valid as v1.1 clients with zero changes.
- **D2**: Appendix A — Painpoint classification protocol (`_CONTRACT-BUG_` vs `_<RUNTIME>-BUG_`) with decision tree.
- **D3**: Appendix B — Worktree isolation requirement for parallel adapter sessions (codifies `parallel-codex-session-protocol.md` Rules 1-6).
- **D4**: Appendix C — CONTRACT spiral validation minimums (2-adapter minimum for FROZEN, 3+ optional belt-and-braces).
- **D5**: Appendix D — Multi-Advisor coordination via claim-file mechanism (codifies `parallel-codex-session-protocol.md` Rule 5).
- **D6**: 3-adapter re-conformance — Claude Code, OpenCode, Codex all re-run conformance suite #01-#07 against v1.1 and PASS without code changes. Conformance is purely additive — v1.1 doesn't tighten any normative requirement, so passes are expected.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/v1.1-contract-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `adapters/CONTRACT.md` v1.1 with 4 appendices (Advisor authors) | PENDING — Advisor |
| 4 | 3-adapter re-conformance evidence | PENDING — CEO codex |
| 5 | `docs/proposals/v1.1-contract-impl-notes.md` (re-conformance trailer) | PENDING — CEO codex |

## Phases + sequencing

```
Phase A (Advisor, 1-2 days):
  - Author CONTRACT.md v1.1 in feature branch worktree
  - Author 4 appendices (Painpoint / Worktree / Spiral / Multi-Advisor)
  - Bump §11 schema version mapping

Phase B (CEO codex, 1-2 days):
  - Re-run conformance suite #01-#07 against v1.1 for 3 adapters
  - One codex session, 3 adapters serial (CONTRACT.md is read-only,
    no shared-file collision risk; serial is cleanest)
  - Document any unexpected fail as `_<RUNTIME>-BUG_` (NOT
    `_CONTRACT-BUG_` — v1.1 is additive so cannot break v1 client)

Phase C (Advisor, 1 day):
  - Verify all 21 conformance datapoints PASS (3 adapters × 7 specs)
  - Ratify v1.1 FROZEN
  - ff-merge feature branch to main
  - Update G-AT-CONTRACT-V1.1 retrospective
```

**Total target**: < 1 week wall-clock.

## Appendices content sketch (Phase A details)

### Appendix A — Painpoint Classification Protocol

For any painpoint surfaced during adapter implementation, conformance validation, or production operation, classify as exactly one of:

- **`_CONTRACT-BUG_`** — violation OR ambiguity in §1-§11 normative behavior. Triggers: spec doesn't match observed behavior, two adapters interpret same §X differently, CONTRACT silent on a real-world case. **Action**: STOP. Painpoint memo + unfreeze CONTRACT for v.next spiral.
- **`_<RUNTIME>-BUG_`** — provider/runtime-specific quirk, resolution local to that adapter. Triggers: only one runtime exhibits behavior, contract is unambiguous, runtime version-specific. **Action**: file as adapter-local issue, do NOT unfreeze CONTRACT.

Decision tree included in appendix body.

### Appendix B — Worktree Isolation for Parallel Adapter Sessions

When 2+ adapter sessions (or 2+ codex sessions running against adapters) operate on the same git repo within overlapping time windows, **each session MUST use a separate worktree on a separate feature branch**. Shared files (CONTRACT.md, conformance specs, adapter IMPLEMENTATION.md) need explicit owner designation per round.

Codifies `parallel-codex-session-protocol.md` Rules 1-6, with the protocol promoted from process doc to CONTRACT appendix (process doc becomes a pointer-back to CONTRACT App B).

### Appendix C — CONTRACT Spiral Validation Minimums

A CONTRACT version is eligible for FROZEN status when:
- **Minimum**: 2 architecturally-distinct adapter implementations independently pass conformance suite #01-#07, producing 14 PASS datapoints with zero `_CONTRACT-BUG_`.
- **Optional belt-and-braces**: 3rd adapter (this is what G-AT-PROD-1 did with Codex round 3). Adds 7 datapoints, higher confidence, not strictly required.

A version with 1 adapter passing is **PROPOSED** (not FROZEN). A version with 2 PASS but 1 outstanding `_CONTRACT-BUG_` is **DRAFT** (not FROZEN). Codifies the rule that emerged organically across v0/v0.1/v1 spiral rounds.

### Appendix D — Multi-Advisor Coordination

When 2+ Advisor sessions might be active simultaneously on the same goal:
- One session is **primary**; others defer.
- If primary is uncertain: first Advisor to write `.aiplus/agent-team/_advisor-claim.txt` with a UTC timestamp becomes primary. Other Advisors read this file before acting.
- Claim files younger than 10 minutes are valid locks. Older → stale, can be overridden after a 1-minute grace period.
- Primary removes claim file after coordination completes.

Codifies `parallel-codex-session-protocol.md` Rule 5. Without this, the cross-Advisor race observed during G-AT-PROD-1 combined commit (byte-identical 56ea0e5 commits from two Advisor instances) recurs.

## Out of scope

- **Any §1-§11 normative change** — those would require v2 (major bump + 2-adapter spiral re-validation against §1-§11 changes). v1.1 is additive only.
- **Schema migrations beyond §11 version-mapping entry** — no AdapterRequest/AdapterResult field changes.
- **Secret-broker rules** — Advisor process improvement (out of CONTRACT scope; CONTRACT is adapter behavior, not Advisor instruction quality).
- **Consultant API stability rules** — separate architectural concern; tracked elsewhere.
- **DESIGN.md §15.1 hardware signing OR §22 cross-provider auditor** — those are G-AT-SEC-1's scope.

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| An appendix wording accidentally tightens a §1-§11 rule → existing adapter fails re-conformance | LOW | MEDIUM (rework appendix, not contract redesign) | Advisor reviews each appendix for "is this adding a new normative requirement?" — if yes, demote to recommendation or move to v2 scope |
| Multi-Advisor claim file mechanism untested — could deadlock or stale-lock in practice | LOW | LOW (Advisor manual override always available) | Phase C includes one simulated 2-Advisor scenario; if mechanism fails, demote App D from normative to recommendation |
| Re-conformance discovers a `_CONTRACT-BUG_` in v1 that wasn't caught earlier | LOW | HIGH (unfreezes v1, blocks goal) | Conformance suite already passed 21 datapoints at v1 freeze; v1.1 is additive so cannot introduce new bugs; any fail is `_<RUNTIME>-BUG_` |
| Schema version bump confuses external readers who pin to `v1` exactly | LOW | LOW | §11 entry explicitly states "v1 and v1.1 are mutually compatible; clients pinning to v1 remain valid" |

## Acceptance criteria (for goal completion)

- ✓ D1-D6 all met
- ✓ CONTRACT.md has 4 new appendices A-D + §11 v1.1 schema entry
- ✓ 21 conformance datapoints PASS (3 adapters × 7 specs)
- ✓ Zero `_CONTRACT-BUG_` from re-conformance (additive guarantee)
- ✓ Existing 3 adapter IMPLEMENTATION.md files unchanged
- ✓ `parallel-codex-session-protocol.md` updated to reference CONTRACT v1.1 App B + App D (process doc becomes pointer)
- ✓ Phase C ratification commit + push within 1 week of Phase A start

## What happens when goal completes

- CONTRACT advances to **v1.1 FROZEN** state
- `docs/decisions/parallel-codex-session-protocol.md` retains but downgrades to pointer-back to CONTRACT App B + App D
- Future spiral rounds reference v1.1 (next likely: v2 if G-AT-DAEMON-1 daemon handshake forces it; otherwise v1.1 holds)
- Phase B's checkpoint outcome informs whether to start G-AT-PROD-2 immediately or pause for dogfood

## Dependencies

- G-AT-PROD-1 retrospective ✓ (commit `9869a27`, aiplus-agent-team)
- `parallel-codex-session-protocol.md` ✓ (same commit)
- 3 adapter implementations + 7 conformance specs ✓ (CONTRACT v1 FROZEN at `2affaa9`)
- Owner approval to enter Phase A drafting

## Next action

Owner reads this goal doc + the companion briefing → ack or push-back → Advisor enters Phase A (drafts CONTRACT.md v1.1 in feature branch worktree). Phase B kicks off when v1.1 draft is committed to `feat/contract-v1.1` branch.

---

— Advisor, 2026-05-18, against CONTRACT v1 FROZEN at `2affaa9`, G-AT-PROD-1 closed
