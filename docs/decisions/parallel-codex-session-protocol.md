# Parallel Codex Session Protocol (process doc)

**Drafted**: 2026-05-18 by Advisor (initial codification)
**Status**: **SUPERSEDED by CONTRACT v1.1 Appendices D + F (2026-05-18 same day).**
**Forensic origin**: G-AT-PROD-1 route.rs collision incident — see `goal-G-AT-PROD-1-retrospective.md` "What went wrong" #1.

---

## Why this file is now a pointer

This document was the original codification of parallel-session safety rules, drafted same-day as the G-AT-PROD-1 route.rs collision incident. CONTRACT v1.1 (authored later the same day) promoted these rules into the authoritative contract surface so adapter implementations and operators have a single source of truth:

- **Worktree isolation** (former Rules 1-4, 6, anti-patterns) → **CONTRACT.md Appendix D**
- **Multi-Advisor race / claim-file mechanism** (former Rule 5) → **CONTRACT.md Appendix F**

This file is retained for:
1. Forensic trail (commit history shows when and why these rules emerged).
2. Grep discoverability — older briefings link to `parallel-codex-session-protocol.md` by name.
3. Operational copy of the worktree-isolation briefing template (below).

## Briefing template (still in active use)

Every codex-session briefing MUST include this block verbatim near the top, with `<work-name>` filled in:

```text
## Worktree isolation (NON-OPTIONAL)

- Worktree:  ~/Projects/AiPlus/<repo>.<work-name>/
- Branch:    feat/<work-name>
- Setup:     git -C <repo> worktree add ../<repo>.<work-name> -b feat/<work-name>
- All work inside this worktree. Do NOT touch main worktree.
- Push to feat/<work-name> only. Advisor handles main merge.

Shared-file ownership for this round:
- This session OWNS: <list>
- This session must NOT modify: <list>
- If you need to touch a not-owned shared file, STOP and ping Advisor.
```

This template is the operational form of CONTRACT App D Rules D.1 + D.3.

## Forensic record (preserved)

The original 6 rules were drafted 2026-05-18 post G-AT-PROD-1 route.rs collision.

**Negative example**: pre-protocol parallel sessions on `aiplus-public` produced an uncommitted-working-tree collision on `route.rs`. Two sessions (disk cache + v0.3 coordinator) both modified the same file. Resolution required a combined commit `56ea0e5` by Advisor coordination — ~20 minutes recovery cost.

**Positive example**: post-protocol `feat/v0.3-d5-evidence`, `feat/v0.3.0-polish`, and `feat/contract-v1.1` (the branch that authored this very downgrade) all merged via ff cleanly with zero collision.

Cross-Advisor race (the other half of G-AT-PROD-1 incident): two Advisor instances authored byte-identical `56ea0e5` commits. Now prevented by App F claim-file mechanism.

## Original rule history (archived; reference CONTRACT App D + F for authoritative form)

- Rule 1 (per-session worktree) → App D Rule D.1
- Rule 2 (feature branch, never direct to main) → App D Rule D.2
- Rule 3 (Advisor ff-merge after CEO done) → operational practice; documented per-goal in goal docs
- Rule 4 (shared-file owner designation) → App D Rule D.3
- Rule 5 (multi-Advisor claim file) → App F Rules F.1-F.4
- Rule 6 (briefing template) → see "Briefing template" section above; same content

---

— Advisor, 2026-05-18 (drafted + same-day superseded)
