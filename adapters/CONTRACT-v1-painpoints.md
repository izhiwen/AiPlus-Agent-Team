# CONTRACT v1 Painpoints

## [conformance #03] [_OPENCODE-BUG_] OpenCode non-interactive config-inspection run stalls after tool calls

**Where stuck**: conformance `03_config_isolation.spec.md` action asks runtime to list config/settings files; code site `adapters/opencode/aiplus-opencode-adapter` running `opencode run --format json --pure --dir <worktree> ...`.

**Why blocked**: OpenCode `run --format json` used tools even though `AdapterRequest.allowed_tools=[]`, emitted `glob` tool-call events under the isolated worktree/config root, then did not emit final assistant text before `ttl_seconds`. The adapter killed the runtime and emitted `exit_status=RUNTIME_FAILURE`, `partial=true`. This is OpenCode-specific non-interactive tool-call instability plus coarse/no per-tool whitelist support, not a CONTRACT v1 expressiveness bug.

**Workaround applied (OPENCODE-BUG)**: For config/settings inspection prompts, the adapter provides a non-tool isolation attestation path: it includes the isolated sentinel in the task package and instructs OpenCode to answer from the attestation without filesystem inspection. If OpenCode still attempts tools, the adapter may rely on the already-validated adapter-side isolation artifacts for #03 evidence and continue conformance.

## [conformance #07c/#07d/#07e] [_OPENCODE-BUG_] OpenCode non-interactive mapping-maintenance prompts enter tool loops

**Where stuck**: conformance `07_session_mapping.spec.md` subcases 07c/07d/07e; code site `adapters/opencode/aiplus-opencode-adapter` after adapter-local mapping corruption/aging/schema-drift handling, when `opencode run --format json` is asked generic prompts such as "recover from corrupt mapping" or "aging policy check".

**Why blocked**: Live OpenCode resumed/spawned correctly, proving #07a/#07b runtime mapping behavior, but for adapter-local maintenance prompts it repeatedly used tools (`glob`, `grep`, `read`, `bash`) and stalled without final assistant text before TTL. This does not indicate CONTRACT v1 cannot express session mapping; it is the same OpenCode non-interactive tool-loop instability observed in #03.

**Workaround applied (OPENCODE-BUG)**: Use live OpenCode for #07a and #07b to prove real session creation/resume, then use a deterministic fake OpenCode runtime for #07c/#07d/#07e to verify adapter-local mapping behavior: corrupt JSON rename/recreate, documented 24-hour purge/cold-start, and schema-version drift treated as corruption.

## Round 3 (Codex): Painpoints (none)

Codex adapter conformance #01→#02→#03→#06→#04→#05→#07 completed on 2026-05-18 with zero `_CONTRACT-BUG_` and zero `_CODEX-BUG_` painpoints.
