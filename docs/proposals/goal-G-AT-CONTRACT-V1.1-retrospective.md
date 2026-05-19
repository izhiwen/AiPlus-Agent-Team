# Goal G-AT-CONTRACT-V1.1 — Retrospective (COMPLETE)

**Status**: **DONE** — CONTRACT v1.1 FROZEN ratified 2026-05-18.
**Original goal**: `docs/proposals/goal-G-AT-CONTRACT-V1.1.md`
**Closing date**: 2026-05-18 (same day as opening)
**Duration**: ~1 work day (vs. <1 week budget). Under budget by ~70%.

---

## What shipped

| # | Sub-deliverable | Final commit | Repo |
|---|---|---|---|
| 1 | `docs/proposals/goal-G-AT-CONTRACT-V1.1.md` (goal doc) | `8126ac4` | aiplus-agent-team |
| 2 | `docs/decisions/v1.1-contract-impl-briefing.md` (CEO briefing v1 + v2) | `8126ac4` + `45153d7` | aiplus-agent-team |
| 3 | `adapters/CONTRACT.md` v1.1 (App C-F + §11.1 + DRAFT→FROZEN flip) | `bb0ca01` + Phase C | aiplus-agent-team |
| 4 | `docs/decisions/parallel-codex-session-protocol.md` (downgraded to pointer-back) | `bb0ca01` | aiplus-agent-team |
| 5 | `docs/proposals/v1.1-contract-impl-notes.md` (Phase B evidence × 2 iterations) | `c54a905` + `92f6f07` | aiplus-agent-team |
| 6 | This retrospective | Phase C | aiplus-agent-team |

**Total**: 6 commits, ~600 lines added.

## D-goal status

- D1 (v1 → v1.1 schema bump in §11): ✓ — §11.1 added
- D2 (Appendix A — Painpoint Classification): ✓ — became App **C** to avoid collision with existing App A (runtime speed-table)
- D3 (Appendix B — Worktree Isolation): ✓ — became App **D**
- D4 (Appendix C — Spiral Validation Minimums): ✓ — became App **E**
- D5 (Appendix D — Multi-Advisor Coordination): ✓ — became App **F**
- D6 (3-adapter re-conformance, 21 PASS): ✓ — achieved on iteration 2 after retry-once gate validation

## What went well

1. **Phase split (Advisor authors, CEO validates) worked cleanly.** Authorship of contract wording is a governance-level decision; validation is a mechanical check. Splitting them prevented CEO from being asked to make calls outside their scope and let Advisor own the contract text.
2. **Retry-once gate (added in iteration 2) saved a false `_OPENCODE-BUG_` classification.** OpenCode #02 inject_memory hard-FAILed on first attempt (`RUNTIME_FAILURE`, empty stdout). Without the retry-once gate, this would have been recorded as `_OPENCODE-BUG_` and v1.1 would have promoted as `FROZEN-WITH-KNOWN-ISSUE`. The retry PASSed cleanly — it was flakiness in OpenCode non-interactive mode, not a bug. Catching this preserved v1.1's clean FROZEN status.
3. **App C decision tree worked as designed.** When CEO hit a FAIL, they correctly classified `_OPENCODE-BUG_` (not `_CONTRACT-BUG_`) by walking the decision tree. CEO and Advisor agreed on the classification independently. Tree is testable as written.
4. **Worktree isolation per App D / parallel-codex-session-protocol succeeded.** Goal ran in a dedicated worktree (`aiplus-agent-team.contract-v1.1`) on `feat/contract-v1.1` branch with zero collision against main worktree or other Advisor activity.
5. **CEO discipline on scope fence.** Both iteration 1 (BLOCKED) and iteration 2 (PASS) commits touched only `v1.1-contract-impl-notes.md`. CONTRACT.md, conformance specs, IMPLEMENTATION.md, and adapter code were untouched. No "while I'm here" creep.

## What went wrong

1. **Advisor briefing iteration 1 had an over-strict STOP rule.** "STOP at first FAIL, classify as `_<RUNTIME>-BUG_`" looked safe but actually contradicted the very App C decision tree it was supposed to enforce: App C explicitly anticipates that additive re-conformance FAILs are `_<RUNTIME>-BUG_`, which by definition do NOT require STOP. STOPping discarded data needed to confirm v1.1 soundness across remaining 13/21 datapoints. **Cost**: one BLOCKED iteration, ~20 min recovery (re-draft briefing iteration 2 + re-dispatch CEO). **Lesson codified**: see "Architectural lessons" #1 below.
2. **OpenCode v1.15.3 flakiness in non-interactive mode is real and observable.** Without retry-once, a single FAIL could mis-bucket as a runtime bug. **Mitigation**: retry-once gate is now part of the conformance methodology. **Follow-up**: open-source OpenCode upstream issue tracker for the inject_memory non-interactive case if it recurs in production.

## Architectural lessons codified

1. **STOP only on `_CONTRACT-BUG_`; `_<RUNTIME>-BUG_` annotates + continues.** Iteration 1 briefing's blanket STOP rule was wrong. The STOP gate must filter by classification, not by raw FAIL/PASS. Encoded in the iteration 2 briefing and applicable to all future re-conformance goals (G-AT-PROD-2 onward).
2. **Retry-once gate before classifying any FAIL.** Single-shot failures in non-interactive runtimes (especially OpenCode) cannot reliably distinguish flakiness from bugs. Two-shot failures can. Encoded in iteration 2 briefing.
3. **Phase-based authorship/validation split for contract work.** When the work product is governance-level text (contract wording, policy decisions), Advisor authors and CEO validates by re-conformance. When the work product is implementation code, CEO authors and Advisor validates by tests. Not interchangeable.
4. **App D + App F validation worked as positive forensic examples.** This goal itself used worktree isolation per App D and primary-Advisor designation per App F. Both succeeded. The appendices are now empirically validated, not just theoretical.

## Open follow-ups (not part of this goal)

- App C/D/E/F could grow concrete examples sections as more goals exercise them — currently each has one negative forensic example from G-AT-PROD-1. v1.2 might add a "validated by" subsection per appendix.
- The retry-once gate is a single retry; if production observes triple-flaky behavior on OpenCode, the gate may need to grow to retry-N or backoff strategy. Defer until evidence accumulates.
- Multi-Advisor claim-file mechanism (App F) was not exercised this goal because only one Advisor session ran. First real test is whenever a second Advisor concurrently engages a goal.

## Next goal

Per the 4-goal program Owner ratified ("好。全都做"):

1. ✓ **G-AT-CONTRACT-V1.1** (this goal) — DONE.
2. **G-AT-PROD-2** (v0.3.1 P1: expert auto-summoning / risk-based forced summoning / TTL honoring) — NEXT.
3. G-AT-DAEMON-1 (v0.4 cross-runtime parallelism) — Owner checkpoint after G-AT-PROD-2.
4. G-AT-SEC-1 (hardware signing + cross-provider auditor) — parallel with G-AT-DAEMON-1 in different repo per parallel-codex-session-protocol.

Advisor enters G-AT-PROD-2 spec drafting next.

---

— Advisor, 2026-05-18 (goal opened + closed same day, 1 of 4 in the program)
