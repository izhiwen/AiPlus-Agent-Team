# Goal G-AT-LOBBY-1 — Retrospective (COMPLETE)

**Status**: **DONE** — Bare-`aiplus` lobby shipped in v0.6.11 on 2026-05-19.
**Original goal**: `docs/proposals/goal-G-AT-LOBBY-1.md`
**Closing date**: 2026-05-19 (opened + closed same day)
**Duration**: ~0.5 work day (CEO ~20 min + Advisor Phase C ~15 min) vs 2-day budget.

---

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | Goal doc + CEO briefing | `89ea5ab` | aiplus-agent-team |
| 2 | Lobby module + main.rs dispatch + tests + impl-notes | `bf5e860` | aiplus-public |
| 3 | Phase C: 0.6.10 → 0.6.11 + CHANGELOG + install.sh parity + this retrospective | post-`bf5e860` | both |

**Total impl**: 922 lines (536 lobby module + 6 main.rs change + 189 integration tests + 195 impl-notes).

## D-goal status

- D1 (bare aiplus → lobby dispatch): ✓
- D2 (auto-install when `.aiplus/` missing + silently skip missing runtime CLIs): ✓
- D3 (role picker, Enter default = ceo, partial match): ✓
- D4 (runtime picker, skip if 1 available, partial match): ✓
- D5 (spawn role+runtime): ✓ — calls `aiplus agent talk --runtime <r> <role>` underneath

## Advisor live re-test (per new methodology)

| Test | Result |
|---|---|
| Fresh project, Enter twice | ✅ Auto-install runs for all 3 runtimes; role list shows 8 core + 11 experts with Chinese descriptions; Enter defaults to ceo; Enter defaults to claude-code (first available); persona loaded correctly |
| Already-setup project | ✅ Recognizes existing setup ("project is set up", not "not set up yet"); role list displayed |
| Regression: `aiplus agent route --score-only` | ✅ Existing subcommand path unaffected; LIGHT_NO_CODE + auto-summon still works |

All 3 mock-up flow elements from the goal doc reproduced exactly in my own live test, not just in CEO's reported transcript.

## What went well

1. **CEO produced pure PASS verdict.** No PASS_WITH_DEVIATIONS, no STOP. Briefing template + scope discipline producing reliable output continues to be the steady state.
2. **Lobby UX matches AEL pattern.** Mock-up flow from goal doc came out almost word-for-word in actual implementation. Cross-project UX consistency (AEL bare `ael` + aiplus bare `aiplus` use same pattern) reduces cognitive load when Owner switches projects.
3. **Role list polish exceeded spec.** CEO included Chinese descriptions for each role (per spec template) AND organized into "Core team" vs "Experts" sections, matching the AEL transcript's visual hierarchy. Polish without prompting.
4. **Partial-match completion** works (verified in CEO's test file). Type `eng` → `engineer-a`. Improves UX for power users.
5. **Auto-install silent skip** for missing runtime CLIs is the correct behavior — no friction for users who only have one of the three runtimes installed.
6. **Phase 3 evidence in CEO's report matched my Phase C live re-test.** No surprises this time (unlike Discovery v2 where CEO 3/3 vs Advisor 1/3 diverged). Methodology stays useful even when no divergence — confirms quality.

## What went wrong

1. **Nothing in this goal.** Pure PASS, Advisor live re-test confirms CEO's claims. The pattern continues to deliver.

## Architectural lessons codified

1. **Cross-project UX pattern mirroring is high-leverage.** Owner saw AEL's lobby flow, decided "same for aiplus", and within a half-day aiplus has the same UX. When two projects share a primary user (Owner), copying validated UX patterns saves the second project from re-discovering the design.

2. **Default value matters in lobby UX.** AEL defaults to `pi` (Principal Investigator, econ-research term); aiplus defaults to `ceo` (different team composition). Capturing this difference in the briefing pre-empted a bug where CEO might have copied AEL's default literally.

3. **"Lobby + discovery layer" is the canonical multi-layer UX architecture.** Lobby gets user into a role+runtime; discovery layer (v0.6.9-v0.6.10) gets natural-language inside the session routed to MCP tools. Two distinct concerns; two distinct mechanisms; combined they cover the full UX.

## Open follow-ups (post-dogfood)

- **Lobby in non-interactive mode** — currently lobby is interactive prompt only. Future: `aiplus --role ceo --runtime claude-code` as one-shot non-interactive alternative for scripts.
- **Lobby preferences persistence** — if user always picks same role+runtime, future: remember last selection and offer as default.
- **Empty Enter at runtime prompt** — current behavior picks first available; future: be explicit about which is "first" (maybe preference order: claude-code > codex > opencode).
- **Multi-team lobby** — if user has both agent-team AND aieconlab installed, lobby could prompt team first. Deferred per Owner's "no team prompt in v1".

## Next goal / dogfood interaction

This was a fast goal slotted between dogfood batches. Lobby UX is now available for Owner to dogfood as part of v0.6.11 — the next time Owner `cd`s into any project and types `aiplus`, the lobby is the experience.

Dogfood notes (Phase B of Wave 2 VALIDATE) can now include lobby observations.

---

— Advisor, 2026-05-19. Lobby UX in 0.5 day (CEO 20 min + Advisor 15 min). 17th goal in 3 calendar days.
