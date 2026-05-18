# Conformance #05 — token_report

**Validates**: CONTRACT §2 `usage_tokens` field — day-1 in schema, best-effort population, null OK, never throws.

**Pre-conditions**: any runtime invocation completing successfully (re-use #01 fixture).

**Action**: dispatch a simple task ("say 'hello world' and stop"), capture AdapterResult.

**Pass criteria** (per-runtime):
- **Claude Code adapter**: `usage_tokens.{input, output, total}` MUST all be present and `> 0` (runtime reports per-turn).
- **Codex adapter**: same expectation (JSONL events include token counts).
- **OpenCode adapter**: `usage_tokens=null` is ACCEPTABLE — per-call token reporting may not exist (`opencode stats` is post-hoc). The velocity consumer records `cost_unknown` and proceeds. Schema must still validate (5 fields present, `usage_tokens` explicitly null).

**Fail conditions**:
- Adapter THROWS or refuses to emit AdapterResult because token data was unavailable.
- Schema invalid (e.g., `usage_tokens` field missing rather than null).
- Reported `usage_tokens` is a non-null object missing required sub-fields when populated.

**Notes**: this spec encodes the explicit lock: token usage is best-effort, never blocking. Adapter author writes "always null" path in IMPLEMENTATION.md if runtime cannot report; that is conformant.
