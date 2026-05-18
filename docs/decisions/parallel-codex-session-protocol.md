# Parallel Codex Session Protocol (process doc)

**Drafted**: 2026-05-18 by Advisor
**Status**: NORMATIVE for all future parallel-codex-session work on `aiplus-public` and `aiplus-agent-team`.
**Forensic origin**: G-AT-PROD-1 route.rs collision incident — see `goal-G-AT-PROD-1-retrospective.md` "What went wrong" #1.

---

## When this protocol applies

Whenever Advisor briefs **two or more** parallel codex sessions to do work on the **same git repository within overlapping time windows**. Even if briefings target nominally-disjoint code paths, shared files (Cargo.toml, CHANGELOG.md, mod.rs, route.rs, lib.rs, etc.) collide more often than expected.

## Rule 1 — Each codex session works in its own worktree

Briefing MUST specify:

```text
- Worktree:  ~/Projects/AiPlus/<repo>.<work-name>/
- Branch:    feat/<work-name>
```

Example for G-AT-PROD-1:
- Disk cache: `aiplus-public.disk-cache/` on `feat/v0.2-disk-cache`
- v0.3 coordinator: `aiplus-public.v03-coord/` on `feat/v0.3.0-p0-coordinator`
- Codex adapter: `aiplus-agent-team.codex-r3/` on `feat/codex-adapter-round-3`

Codex session begins by creating the worktree:

```bash
git -C <repo> worktree add ../<repo>.<work-name> -b feat/<work-name>
cd ../<repo>.<work-name>
```

All edits, commits, tests happen inside the worktree. Session never touches the main worktree's working tree.

## Rule 2 — Feature branch, never direct to main

Each session pushes to its feature branch only:

```bash
git push origin feat/<work-name>
```

Never `git push origin main` from a codex session. Main pushes are Advisor-coordinated.

## Rule 3 — Advisor merges via ff after CEO codex confirms done

When CEO codex reports the work complete + tests PASS:

1. Advisor (or Advisor-supervised sub-agent) verifies feature branch:
   ```bash
   git fetch origin
   git diff main..origin/feat/<work-name> --stat
   cargo test --workspace  # on the feature branch
   ```
2. If clean: ff merge to main + push:
   ```bash
   git checkout main
   git merge --ff-only origin/feat/<work-name>
   git push origin main
   ```
3. Feature branch is preserved (NOT deleted) for forensic trail. Tiny disk cost; large incident-recovery value.

## Rule 4 — Shared files (route.rs, Cargo.toml, etc.) need explicit owner

If parallel sessions BOTH need to modify the same shared file, briefing MUST designate one as **owner** of the shared file in this round:

- Owner session: makes edits, commits, pushes feature branch first.
- Non-owner session(s): wait for owner's branch to merge into main, then rebase their feature branch on the new main, integrate, commit, push.

If briefing fails to designate an owner → second session to touch the shared file STOPs and pings Advisor for coordination. NEVER silently merge ad-hoc.

## Rule 5 — Multi-Advisor coordination

Multiple Advisor sessions on the same goal can also race (this happened in G-AT-PROD-1 during the combined commit). Guidelines:

- One Advisor session is **primary** for a given goal. Others defer.
- If primary is uncertain, the first Advisor session to commit-stash-claim the working state by writing `.aiplus/agent-team/_advisor-claim.txt` becomes primary; others read this file before acting.
- If claim file exists and is younger than 10 minutes, secondary Advisor waits or pings primary.
- After primary completes coordination, primary removes claim file.

## Rule 6 — Briefing template addition

Every future codex briefing MUST include this section verbatim near the top, with `<work-name>` filled in:

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

## Anti-patterns (FORBIDDEN)

- ❌ Parallel codex sessions sharing `~/Projects/AiPlus/<repo>/` working tree
- ❌ Codex session pushing directly to main
- ❌ Silent ad-hoc merge of conflicting working-tree changes
- ❌ Advisor doing combined commits without first checking if another Advisor instance is active
- ❌ Deleting feature branches before merging to main and verifying

## Validation

This protocol is validated by:
- G-AT-PROD-1 incident showing what happens WITHOUT it (route.rs collision, combined commit, cross-Advisor race)
- D5 evidence patch in G-AT-PROD-1 successfully using the protocol (`feat/v0.3-d5-evidence` branch, clean ff merge)

## Future revisions

- v1.0 (this) — initial protocol from G-AT-PROD-1 retrospective.
- v1.x (anticipated) — add specifics for cross-repo parallel work (e.g., aiplus-public + aiplus-agent-team simultaneously).
- v2 (potential) — if claim-file mechanism (Rule 5) proves insufficient, upgrade to file-locking or coordinator agent.

---

— Advisor, 2026-05-18, post G-AT-PROD-1 close
