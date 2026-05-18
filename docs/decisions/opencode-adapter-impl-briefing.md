# OpenCode Adapter Reference Implementation — Briefing for CEO Codex (Round 2)

**Drafted**: 2026-05-18 by Advisor (CONTRACT v1 author)
**Target executor**: CEO codex session
**Authoritative source**: CONTRACT v1 at commit `8293544`
**Predecessor round**: Claude Code adapter at commit `d04797e` (round 1, all 7 conformance specs PASS)

---

You are CEO of AiPlus Agent Team. Implement `adapters/opencode/` as the **spiral-round-2** reference adapter against CONTRACT v1.

## Why round 2 matters (READ THIS FIRST)

Round 1 (Claude Code) showed CONTRACT v0.1 had real gaps that v1 fixes. Round 2's primary value is NOT "make OpenCode work" — it is **stress-testing whether CONTRACT v1 is Claude-shaped or truly runtime-agnostic**.

OpenCode has known upstream limitations (G1.1 V1.1 history: `opencode run` non-interactive mode + ready-marker capture instability). Expect to hit them. **When you hit one, distinguish carefully**:

- **OpenCode upstream bug** → log as `_OPENCODE-BUG_` in painpoints, work around in IMPLEMENTATION.md, continue
- **CONTRACT v1 inadequacy** → log as `_CONTRACT-BUG_`, STOP per protocol, return to Advisor

This distinction is **the most important output of round 2**.

## Version lock

- **CONTRACT v1 is frozen** at commit `8293544`: `adapters/CONTRACT.md` + `adapters/conformance/*.spec.md` (specs unchanged from v0.1).
- DO NOT edit CONTRACT or specs during this round. Found a CONTRACT bug → log painpoint, STOP, return.

## Strict working order — sequential, not parallel

**Phase 1 — write `adapters/opencode/IMPLEMENTATION.md` ONLY. All 7 sections (CONTRACT Appendix B). NO adapter code yet.**

Cheat sheet — recommended decisions (validate while writing; expect more friction than Claude Code):

1. Transport: subprocess + `opencode run --format json` (raw events; less structured than Claude's stream-json — adapter parser does more lifting)
2. Config isolation: `--pure` (skips external plugins) + `XDG_CONFIG_HOME=<isolated-dir>` env override (per CONTRACT §6.1 ALLOWED env-var channel; CONFIRM mechanism during impl — fact-finding labelled it "likely")
3. cwd: `opencode run [project]` accepts the positional `[project]` path; pass absolute `worktree_path` there
4. Tool whitelist: **likely painpoint** — OpenCode only has `--dangerously-skip-permissions` (coarse on/off). Translating `allowed_tools[]` to per-tool whitelist may not be expressible; document the gap honestly, do NOT fake compliance
5. `usage_tokens`: per-call usage may be null (`opencode stats` is post-hoc); per CONTRACT §3 + IMPLEMENTATION.md §5, returning `null` is allowed — emit normally
6. Session continuation: `-s, --session <id>` flag for resuming; mapping per CONTRACT §4 + §4.3 (use CEO UUIDv7 as `runtime_session_id` if OpenCode is opaque)
7. Trace: same pattern as Claude Code adapter — project-local `.aiplus/agent-team/adapter-trace/<session_id>.jsonl` with redaction

**Phase 2 — implement adapter binary/script. IMPLEMENTATION.md MUST be complete before this phase starts.**

**Phase 3 — run conformance + capture painpoints (with bug-class distinction).**

## Conformance sequence + STOP rule

Run in this order (same as round 1):

`#01 → #02 → #03 → #06 → #04 → #05 → #07`

**Any spec fail → first classify, THEN act**:

- If failure is **OpenCode upstream bug** (e.g., `opencode run` hangs in known-broken non-interactive scenario): append to `adapters/CONTRACT-v1-painpoints.md` with `_OPENCODE-BUG_` tag, document workaround in IMPLEMENTATION.md, continue conformance with workaround
- If failure is **CONTRACT v1 cannot express what OpenCode needs** (or v1's assumption breaks): STOP IMMEDIATELY, append `_CONTRACT-BUG_` painpoint, return to Advisor

Painpoint format (extended with class tag):

```
## [conformance #NN] [_CONTRACT-BUG_ | _OPENCODE-BUG_] <one-line symptom>
**Where stuck**: <CONTRACT §X / spec line / code site>
**Why blocked**: <what doesn't work>
**Workaround applied (if OPENCODE-BUG)** OR **Workaround wanted, NOT applied (if CONTRACT-BUG)**: <description>
```

If unsure whether bug is OpenCode-class or CONTRACT-class → default to CONTRACT-BUG and STOP. Better to ping Advisor than silently fork.

## Scope fence

- **IN**: `adapters/opencode/*` only
- **OUT**: other adapters (Claude Code is round 1's frozen baseline; do not modify), CONTRACT.md, conformance specs, new specs
- **GRAY (allowed)**: append additional entries to `adapters/conformance/_harness/global_config_markers.txt` if OpenCode global config lives in non-standard paths; verify Appendix A items still labelled "likely" / "verify during impl" for OpenCode

## Deliverables (4 — all required)

1. `adapters/opencode/IMPLEMENTATION.md` — 7 sections per Appendix B, complete BEFORE Phase 2
2. `adapters/opencode/<binary or script>` — the adapter itself
3. Conformance 1-7 evidence → paste at IMPLEMENTATION.md trailer. Marked PASS / SKIP_OPENCODE_BUG / FAIL where applicable
4. `adapters/CONTRACT-v1-painpoints.md` — every painpoint, tagged `_CONTRACT-BUG_` or `_OPENCODE-BUG_`. **Create even if zero entries** (`## Painpoints (none)`) so Advisor knows the pass completed

## Time ceiling

**Hard ceiling: T+3d.** AI runtime; work likely completes in a fraction. No mid-task reports. STOP-on-CONTRACT-BUG overrides the ceiling.

## Handoff endpoint

After all 4 deliverables exist → **report back to Advisor, not Owner**. Advisor drafts CONTRACT v2 from round 2 painpoints; Owner reviews v2.

Reaching this endpoint with **zero `_CONTRACT-BUG_` painpoints** is the success signal that CONTRACT can be considered **Frozen** (per CONTRACT.md Revisions section). One or more `_CONTRACT-BUG_` painpoints → another spiral round is required.

## Owner gates (always active)

No push, no publish, no global config edits, no force operations, no editing files outside IN scope.

---

— Advisor (CONTRACT v1 author), 2026-05-18, against commit `8293544`
