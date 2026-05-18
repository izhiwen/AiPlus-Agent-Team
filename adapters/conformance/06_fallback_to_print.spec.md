# Conformance #06 — fallback_to_print

**Validates**: CONTRACT §8 print-prompt fallback. The v0.2 trust anchor — installing the adapter is never worse than v0.1.

**Pre-conditions**:
- Strip the runtime binary from PATH (`PATH=/nonexistent`) or rename it (`mv claude claude.disabled`).
- Otherwise normal AdapterRequest.

**Action**: dispatch a representative AdapterRequest as if everything were normal.

**Pass criteria** (all required):
- Adapter exits with code `75` (POSIX `EX_TEMPFAIL`).
- Stdout starts with the literal line `AIPLUS_ADAPTER_FALLBACK=print-prompt`.
- Stdout includes `session_id`, `role`, `worktree` lines.
- Stdout includes the complete `system_prompt + memory + task` in a copy-pasteable block delimited by `---`.
- Stdout ends with `Action: paste above into <runtime> manually; report result via aiplus agent report-manual <session_id>`.
- Trace file (`.aiplus/agent-team/adapter-trace/<session_id>.jsonl`) contains an AdapterRequest entry.
- AdapterResult is emitted (either to stdout after the fallback block, or to `--output-file`) with `exit_status=RUNTIME_NOT_FOUND`, `partial=true`.

**Fail conditions**:
- Adapter crashes / panics / segfaults instead of fallback.
- Fallback block missing any required field (Owner cannot reconstruct manually).
- Trace file absent (Owner cannot audit what would have been dispatched).

**Notes**: this test is non-negotiable. Owner must trust "install adapter → no worse than v0.1." If #06 fails, the adapter is not shippable regardless of other test results.
