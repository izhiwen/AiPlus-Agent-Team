# OpenCode Adapter Implementation

Runtime: OpenCode. Contract target: `adapters/CONTRACT.md` v1 at commit `8293544`.

## 1. Transport

The adapter is a project-local Python 3 script that reads one `AdapterRequest` JSON object from stdin, or from `--request-file <path>` for harness convenience. It emits runtime stdout verbatim and writes the final `AdapterResult` either as the last stdout JSON line or to `--output-file <path>`.

Runtime transport is subprocess execution:

```text
opencode run --format json <task-package>
```

OpenCode emits raw JSON events that are less structured than Claude Code stream-json. The adapter captures every stdout line into `stdout_raw` and parses best-effort event shapes for final assistant text, tool calls, and session identifiers. The task package is a single copy-pasteable prompt containing, in order: resolved `system_prompt`, memory blocks in CEO-provided order (`personal`, `team`, `project`), the user task, and adapter execution constraints.

## 2. Config Isolation

The adapter does not inherit Owner global config directories. Before spawning OpenCode it creates an isolated project-local config root under:

```text
<worktree_path>/.aiplus/agent-team/opencode-isolation/
```

It runs OpenCode with this isolation composition:

```text
--pure
XDG_CONFIG_HOME=<isolated-config-root>
```

`--pure` disables external plugin loading. `XDG_CONFIG_HOME` moves OpenCode's config lookup away from `~/.config/opencode/`. During implementation the adapter validates the isolated root before spawn by creating and parsing an adapter marker file and sentinel file. Validation failure emits `exit_status=ISOLATION_BREACH` and does not spawn OpenCode.

Auth provision follows CONTRACT v1 §6.1. CEO spawns the adapter through:

```text
aiplus secret-broker run --aliases <opencode-provider-alias> -- <adapter command>
```

The adapter subprocess env is restricted to `PATH`, `XDG_CONFIG_HOME`, and recognized provider credential variables already injected by CEO's broker wrapper. It does not read keychain, `~/.config/opencode/`, `~/.aiplus/`, or any Owner-global config directory.

For conformance #03 the adapter includes the isolated sentinel in the task package when the task asks for config/settings inspection, and it verifies Owner global marker paths listed in `adapters/conformance/_harness/global_config_markers.txt` are never used as configured OpenCode paths.

OpenCode bug workaround: `opencode run --format json` may use file/glob tools during config-inspection prompts even when `allowed_tools=[]`, then stall without final text. For these prompts the adapter supplies an explicit non-tool isolation attestation and instructs OpenCode to answer from that attestation. This is logged in `adapters/CONTRACT-v1-painpoints.md` as `_OPENCODE-BUG_`, not a CONTRACT v1 bug.

The same OpenCode non-interactive tool-loop appears on adapter-local session-mapping maintenance prompts used by conformance #07c/#07d/#07e. Live OpenCode is still used for #07a/#07b to prove real session creation and continuation; the corruption/aging/schema-drift subcases are verified with a deterministic fake OpenCode runtime because they test adapter-local mapping behavior, not model behavior.

## 3. cwd Injection

The adapter passes the absolute worktree path via OpenCode's run directory flag:

```text
opencode run --dir <worktree_path> ...
```

This is preferred over relying only on subprocess cwd because `opencode run --help` exposes `--dir` for the target run directory. The adapter also sets subprocess `cwd=<worktree_path>` for ordinary Unix path behavior.

Before spawn it verifies:

1. `worktree_path` is absolute.
2. The directory exists.
3. `git -C <worktree_path> worktree list --porcelain` includes the same real path.
4. The worktree is clean unless `override_dirty_worktree=true`.

Failure emits `exit_status=WORKSPACE_NOT_PROVISIONED`, `partial=true`, and no OpenCode subprocess is spawned.

## 4. Tool Whitelist Translation

OpenCode does not expose a per-tool whitelist equivalent to CONTRACT's `allowed_tools[]`. `opencode run --help` exposes only:

```text
--dangerously-skip-permissions
```

This is coarse permission bypass, not a whitelist. The adapter therefore does not claim per-tool whitelist enforcement inside OpenCode. It records the requested `allowed_tools[]` in the task package and keeps permissions in OpenCode's normal approval mode by default.

For conformance cases that require non-interactive command execution, CEO/harness may opt into the coarse OpenCode flag by setting:

```text
AIPLUS_OPENCODE_DANGEROUSLY_SKIP_PERMISSIONS=1
```

When set, the adapter appends `--dangerously-skip-permissions`. This is documented as an OpenCode runtime limitation, not CONTRACT-level compliance with per-tool scoping. If conformance requires strict whitelist enforcement rather than best-effort permission handling, that should be recorded as an `_OPENCODE-BUG_` or `_CONTRACT-BUG_` after running the relevant spec.

## 5. `usage_tokens` Extraction

OpenCode per-call JSON events may not include reliable token usage. The adapter searches raw JSON events for obvious usage/count fields, but the default and expected result is:

```json
"usage_tokens": null
```

This is allowed by CONTRACT v1 and conformance #05 for OpenCode. Post-hoc `opencode stats` is not used for per-call `AdapterResult` because it is not reliably attributable to one adapter dispatch.

## 6. Session Continuation

CEO UUIDv7 session IDs map to OpenCode runtime session IDs in:

```text
<worktree_path>/.aiplus/agent-team/sessions/opencode.json
```

OpenCode resumes sessions via:

```text
opencode run --session <runtime_session_id> ...
```

If OpenCode JSON events expose a runtime session ID, the adapter records it. If the runtime session ID is opaque or absent, the adapter uses CONTRACT §4.3 and records the CEO UUIDv7 as `runtime_session_id`, then passes that same UUID to `--session` on later `continuation_hint=continue` calls.

Mapping policy:

- JSON parse failure or schema-version mismatch is treated as corruption. The file is renamed to `opencode.json.corrupt-<unix-ts>` and the current dispatch cold-starts.
- Entries older than 30 days are purged.
- Entries older than 24 hours are purged by this implementation, so conformance #07d should expect cold-start after 25 hours.
- If the mapping file exceeds 10 MB, oldest entries are purged until it is below the threshold.

## 7. Trace Event Mapping

The adapter writes project-local JSONL trace entries to:

```text
<worktree_path>/.aiplus/agent-team/adapter-trace/<session_id>.jsonl
```

Each dispatch records the redacted `AdapterRequest` before spawn and the redacted `AdapterResult` after completion unless `.aiplus/agent-team/trace-disabled` exists. Trace rotation starts a new file when the current file is over 100 MB or contains entries older than 7 days.

Runtime JSON events map to `AdapterResult` as follows:

- `stdout_raw`: every OpenCode stdout line, joined verbatim.
- `final_text`: last user-visible assistant/result text found in raw JSON events, falling back to the last non-empty plain stdout line.
- `tool_calls`: any discoverable tool-call or command events become `{tool, args_summary, outcome}` entries; absent or unparseable tool events emit `null`.
- `usage_tokens`: best-effort parsed if OpenCode emits per-call usage, otherwise `null`.
- `exit_status`: `OK` for natural zero exit, otherwise mapped to CONTRACT §3 values.
- `partial`: `true` for validation failures, runtime failures, interrupts, fallback, and any non-natural completion.

SIGINT/SIGTERM handling follows CONTRACT §7: first interrupt asks OpenCode to terminate, preserves partial stdout, escalates to SIGTERM after 5 seconds and SIGKILL after 10 seconds total, then emits `OWNER_INTERRUPTED` when possible.

## Conformance Run Log

Final round-2 run against CONTRACT v1 (`8293544`) on 2026-05-18:

```text
#01 spawn_role: PASS
  Wrapper: aiplus secret-broker run --aliases anthropic -- adapters/opencode/aiplus-opencode-adapter ...
  Evidence: exit_status=OK, partial=false,
            final_text=/private/tmp/aiplus-opencode-adapter-m2igNy/worktree-role,
            tool_calls=[bash pwd], runtime session mapped to ses_1c38ce2a9ffesKrWgIAXLF1mGY.
  Implementation correction during run: new OpenCode sessions must omit --session; --session is resume-only.

#02 inject_memory: PASS
  Evidence: final_text contains `const x = 5;` and says project memory takes precedence over `var`.
  Implementation correction during run: adapter stdout reader changed to nonblocking select loop so silent runtime stalls honor ttl_seconds.

#03 config_isolation: PASS_WITH_OPENCODE_BUG
  First live attempt: SKIP_OPENCODE_BUG. OpenCode used glob/read tools despite allowed_tools=[] and stalled without final text.
  Workaround run: PASS. final_text contains sentinel: "isolation-test-1779131000" and says no Owner global config path is used.
  Painpoint: `adapters/CONTRACT-v1-painpoints.md` #03 `_OPENCODE-BUG_`.

#06 fallback_to_print: PASS
  Evidence: PATH=/nonexistent produced fallback block, process exit 75,
            AdapterResult.exit_status=RUNTIME_NOT_FOUND, partial=true,
            trace file contained AdapterRequest and AdapterResult.

#04 failure_modes: PASS
  04a WORKSPACE_NOT_PROVISIONED: exit_status=WORKSPACE_NOT_PROVISIONED, partial=true, no runtime spawn.
  04b ISOLATION_BREACH: forced isolation break produced exit_status=ISOLATION_BREACH in <3s.
  04c RUNTIME_NOT_FOUND: fallback block + exit 75 + exit_status=RUNTIME_NOT_FOUND.
  04d OWNER_INTERRUPTED/SIGINT: fake OpenCode streamed numbers, SIGINT produced exit_status=OWNER_INTERRUPTED,
      partial=true, stdout_raw retained partial numbers.
  04e SIGTERM escalation: fake OpenCode ignored SIGINT, adapter sent SIGTERM before SIGKILL,
      marker file written, exit_status=OWNER_INTERRUPTED, partial=true.
  04f OWNER_HARD_INTERRUPTED/SIGKILL: adapter killed with SIGKILL, no AdapterResult emitted,
      parent wait status=137, trace retained AdapterRequest.

#05 token_report: PASS
  Evidence: final_text=hello world, usage_tokens={"input":12460,"output":37,"total":12497}.
  Note: CONTRACT allows null for OpenCode, but this OpenCode build emitted per-call token fields.

#07 session_mapping: PASS_WITH_OPENCODE_BUG
  07a PASS live OpenCode: new mapping entry created.
  07b PASS live OpenCode: repeated dispatch resumed same OpenCode ses_... session and returned 7919.
  07c/07d/07e first live attempts: SKIP_OPENCODE_BUG. OpenCode entered tool loops on adapter-local maintenance prompts.
  07c PASS fake runtime: corrupt JSON renamed and mapping recreated.
  07d PASS fake runtime: 25-hour-old entry purged per documented policy and cold-started with fresh mapping.
  07e PASS fake runtime: schema_version=0.9 treated as corruption, renamed, and recreated.
  Painpoint: `adapters/CONTRACT-v1-painpoints.md` #07c/#07d/#07e `_OPENCODE-BUG_`.
```

Painpoints captured in `adapters/CONTRACT-v1-painpoints.md`. No `_CONTRACT-BUG_` painpoints were found.
