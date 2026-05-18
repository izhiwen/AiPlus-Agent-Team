# Claude Code Adapter Reference Implementation — Briefing for CEO Codex

**Drafted**: 2026-05-18 by Advisor (CONTRACT v0.1 author)
**Target executor**: CEO codex session
**Authoritative source**: CONTRACT v0.1 at commit `5bb0a20`

---

You are CEO of AiPlus Agent Team. Implement `adapters/claude-code/` as the spiral-round-1 reference adapter against CONTRACT v0.1.

## Version lock

- **CONTRACT v0.1 is frozen** at commit `5bb0a20`: `adapters/CONTRACT.md` + `adapters/conformance/*.spec.md`.
- DO NOT edit CONTRACT or specs during this round. Found a bug → log a painpoint, keep going (or STOP per below). Advisor converts painpoints into v1.

## Strict working order — sequential, not parallel

**Phase 1 — write `adapters/claude-code/IMPLEMENTATION.md` ONLY. All 7 sections (CONTRACT Appendix B). NO adapter code yet.**

Cheat sheet — recommended decisions per section (validate while writing):

1. Transport: subprocess + `claude --print --output-format stream-json`
2. Config isolation: `--bare --settings <iso> --mcp-config <iso> --agents '{}' --plugin-dir <empty>`
3. cwd: subprocess `cwd=<worktree_path>` (standard Unix)
4. Tool whitelist: `allowed_tools[]` → `--allowedTools "Tool1 Tool2 ..."` join
5. `usage_tokens`: parse stream-json `usage` event
6. Session continuation: CEO UUIDv7 ↔ Claude internal session ID via `--resume <claude_session_id>`
7. Trace event mapping: stream-json events → AdapterResult fields

**Phase 2 — implement adapter binary/script. IMPLEMENTATION.md MUST be complete before this phase starts.**

**Phase 3 — run conformance + capture painpoints.**

## Conformance sequence + STOP rule

Run in this order (deviation from numeric is intentional — #06 is the v0.2 trust anchor, verify early):

`#01 → #02 → #03 → #06 → #04 → #05 → #07`

**Any spec fail → STOP IMMEDIATELY.** Append to `adapters/CONTRACT-v0.1-painpoints.md`:

```
## [conformance #NN] <one-line symptom>
**Where stuck**: <CONTRACT §X / spec line / code site>
**Why blocked**: <what contract doesn't let me express>
**Workaround wanted (NOT applied)**: <patch I'd want in CONTRACT>
```

Then return to Advisor. No fork, no skip, no proceed past failure.

## Scope fence

- **IN**: `adapters/claude-code/*` only
- **OUT**: other adapters, CONTRACT.md, conformance specs, new specs
- **GRAY (allowed)**: create `adapters/conformance/_harness/global_config_markers.txt` (seed: `~/.claude/CLAUDE.md`, `~/.claude/settings.json`, `~/.codex/config.toml`, `~/.config/opencode/config.json`, `~/.aiplus/`); verify Appendix A items still labelled "likely" / "verify during impl"

## Deliverables (4 — all required)

1. `adapters/claude-code/IMPLEMENTATION.md` — 7 sections per Appendix B, complete BEFORE Phase 2
2. `adapters/claude-code/<binary or script>` — the adapter itself
3. Conformance 1-7 all-green run-log → paste at IMPLEMENTATION.md trailer
4. `adapters/CONTRACT-v0.1-painpoints.md` — every "contract didn't let me express X". **Create even if zero entries** (`## Painpoints (none)`) so Advisor knows the pass completed

## Time ceiling

**Hard ceiling: T+3d.** Not a target — work likely completes in a fraction of this. No mid-task reports. STOP-on-fail overrides the ceiling.

## Handoff endpoint

After all 4 deliverables exist → **report back to Advisor, not Owner**. Advisor drafts CONTRACT v1 from painpoints; Owner reviews v1, not raw painpoints.

## Owner gates (always active)

No push, no publish, no global config edits, no force operations, no editing files outside IN scope.

---

— Advisor (CONTRACT v0.1 author), 2026-05-18, against commit `5bb0a20`
