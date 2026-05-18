# Claude Code Adapter Implementation

Runtime: Claude Code. Contract target: `adapters/CONTRACT.md` v0.1 at commit `5bb0a20`.

## 1. Transport

The adapter is a project-local Python 3 script that reads one `AdapterRequest` JSON object from stdin, or from `--request-file <path>` for harness convenience. It emits runtime stdout verbatim and writes the final `AdapterResult` either as the last stdout JSON line or to `--output-file <path>`.

Runtime transport is subprocess execution:

```text
claude --print --output-format stream-json ...
```

The adapter passes the full task package as the single print prompt. The prompt is copy-pasteable and contains, in order: resolved `system_prompt`, memory blocks in CEO-provided order (`personal`, `team`, `project`), task, and adapter execution constraints. Claude stream-json output is captured line-by-line into `stdout_raw`.

## 2. Config Isolation

The adapter never relies on Owner global config. Before spawning Claude it creates project-local isolated files under:

```text
<worktree_path>/.aiplus/agent-team/claude-code-isolation/
```

It passes Claude this exact flag composition:

```text
--bare
--settings <isolated-settings.json>
--mcp-config <isolated-mcp.json>
--strict-mcp-config
--agents {}
--plugin-dir <empty-plugin-dir>
--setting-sources project
--no-chrome
```

The settings file is valid JSON and includes an adapter marker. The MCP file contains an empty MCP server map. The plugin directory is empty and project-local. If these files/directories cannot be created or validated before spawn, the adapter emits `exit_status=ISOLATION_BREACH` and does not spawn Claude.

For conformance #03 the adapter also reads optional marker lines from `adapters/conformance/_harness/global_config_markers.txt` when present. Those paths are used only as negative markers for traceable local validation; the adapter does not read Owner global config contents.

Claude Code `--print` may silently ignore invalid settings files, so this implementation performs adapter-side validation before spawn and includes the isolated marker path/value in the task package as an isolation attestation. If the contract later requires proving runtime-internal settings load without prompt attestation, that should become a v1 painpoint.

Auth provision (CONTRACT §6 letter, intent-gap noted in `adapters/CONTRACT-v0.1-painpoints.md`):

- Adapter subprocess env is restricted to `PATH` and `ANTHROPIC_API_KEY`; all other env is stripped before spawning Claude Code.
- CEO spawns the adapter through `aiplus secret-broker run --aliases anthropic -- <adapter command>`, which injects `ANTHROPIC_API_KEY` for the child process and clears it when the wrapped command exits.
- Adapter does not read keychain, `~/.claude/`, or any global config dir. CONTRACT §6 remains unchanged for this spiral round.

## 3. cwd Injection

The adapter sets subprocess `cwd=<worktree_path>`.

Before spawn it verifies:

1. `worktree_path` is absolute.
2. The directory exists.
3. `git -C <worktree_path> worktree list --porcelain` includes the same real path.
4. The worktree is clean unless `override_dirty_worktree=true`.

Failure emits `exit_status=WORKSPACE_NOT_PROVISIONED`, `partial=true`, and no Claude subprocess is spawned.

## 4. Tool Whitelist Translation

`allowed_tools[]` maps to Claude Code's native tool flags:

```text
--allowedTools "<space-separated allowed_tools>"
--tools "<space-separated allowed_tools>"
```

The adapter preserves runtime-native tool names such as `Bash`, `Edit`, and `Read`. Empty `allowed_tools[]` is treated as no tools:

```text
--tools ""
```

Per-argument restrictions such as `Bash(git *)` are passed through unchanged when CEO includes them, but CONTRACT v0.1 does not require the adapter to derive per-argument patterns.

## 5. `usage_tokens` Extraction

Claude Code stream-json is parsed line-by-line. The adapter searches every JSON event for usage fields in common Claude shapes:

```text
usage.input_tokens
usage.output_tokens
usage.cache_creation_input_tokens
usage.cache_read_input_tokens
message.usage.*
result.usage.*
```

The populated `AdapterResult.usage_tokens` object is:

```json
{"input": <input + cache input tokens>, "output": <output>, "total": <sum>}
```

If no token usage is present, the adapter emits `usage_tokens=null` rather than failing. Conformance #05 requires Claude Code to report non-null positive usage; absence is recorded as a conformance failure, not a runtime exception.

## 6. Session Continuation

CEO UUIDv7 session IDs map to Claude internal session IDs in:

```text
<worktree_path>/.aiplus/agent-team/sessions/claude-code.json
```

For a new CEO session, the adapter starts Claude without `--resume` and records the Claude session ID found in stream-json events. If no runtime session ID is discoverable, it records the CEO session ID as the runtime session ID and also passes it to later calls with Claude's native `--resume <runtime_session_id>` flag.

For `continuation_hint=continue`, the adapter loads the mapping and passes:

```text
--resume <claude_session_id>
```

Mapping policy:

- JSON parse failure or schema version mismatch is treated as corruption. The file is renamed to `claude-code.json.corrupt-<unix-ts>` and the current dispatch cold-starts.
- Entries older than 30 days are purged.
- Entries older than 24 hours are purged by this implementation, so conformance #07d should expect cold-start after 25 hours.
- If the mapping file exceeds 10 MB, oldest entries are purged until it is below the threshold.

## 7. Trace Event Mapping

The adapter writes project-local JSONL trace entries to:

```text
<worktree_path>/.aiplus/agent-team/adapter-trace/<session_id>.jsonl
```

Each dispatch records the redacted `AdapterRequest` before spawn and the redacted `AdapterResult` after completion unless `.aiplus/agent-team/trace-disabled` exists. Trace rotation starts a new file when the current file is over 100 MB or contains entries older than 7 days.

Runtime stream-json events map to `AdapterResult` as follows:

- `stdout_raw`: every Claude stdout line, joined verbatim.
- `final_text`: last user-visible assistant text found in stream-json events, falling back to plain stdout text if needed.
- `tool_calls`: Claude tool-use events become `{tool, args_summary, outcome}` entries; absent tool-use emits `null`.
- `usage_tokens`: parsed as described in section 5.
- `exit_status`: `OK` for natural zero exit, otherwise mapped to CONTRACT §3 values.
- `partial`: `true` for validation failures, runtime failures, interrupts, fallback, and any non-natural completion.

SIGINT/SIGTERM handling follows CONTRACT §7: first interrupt asks Claude to terminate, preserves partial stdout, escalates to SIGTERM after 5 seconds and SIGKILL after 10 seconds total, then emits `OWNER_INTERRUPTED` when possible.

## Conformance Run Log

Final resumed run against CONTRACT v0.1 (`5bb0a20`) on 2026-05-18:

```text
#01 spawn_role: PASS
  Wrapper: aiplus secret-broker run --aliases anthropic -- adapters/claude-code/aiplus-claude-code-adapter ...
  Evidence: exit_status=OK, partial=false, final_text=/private/tmp/aiplus-claude-adapter-Wb99og/worktree-role,
            session mapping created for 019e44e9-0000-7000-8000-000000000101.

#02 inject_memory: PASS
  Evidence: final_text contains `const x = 5;` and cites project rule overriding personal `var`.

#03 config_isolation: PASS
  Evidence: stdout contains sentinel: "isolation-test-1779127600"; no Owner global config path appeared.

#06 fallback_to_print: PASS
  Evidence: PATH=/nonexistent produced fallback block, process exit 75,
            AdapterResult.exit_status=RUNTIME_NOT_FOUND, partial=true,
            trace file contained AdapterRequest.

#04 failure_modes: PASS
  04a WORKSPACE_NOT_PROVISIONED: exit_status=WORKSPACE_NOT_PROVISIONED, partial=true, no runtime spawn.
  04b ISOLATION_BREACH: forced isolation break produced exit_status=ISOLATION_BREACH in <3s.
  04c RUNTIME_NOT_FOUND: fallback block + exit 75 + exit_status=RUNTIME_NOT_FOUND.
  04d OWNER_INTERRUPTED/SIGINT: harness runtime streamed numbers, SIGINT produced exit_status=OWNER_INTERRUPTED,
      partial=true, stdout_raw retained partial numbers.
  04e SIGTERM escalation: harness runtime ignored SIGINT, adapter sent SIGTERM before SIGKILL,
      marker file written, exit_status=OWNER_INTERRUPTED, partial=true.
  04f OWNER_HARD_INTERRUPTED/SIGKILL: adapter killed with SIGKILL, no AdapterResult emitted,
      parent wait status=137, trace retained AdapterRequest.

#05 token_report: PASS
  Evidence: final_text=hello world, usage_tokens={"input":1240,"output":16,"total":1256}.

#07 session_mapping: PASS
  07a new session created mapping entry.
  07b repeat session resumed Claude session and returned 7919.
  07c corrupt mapping renamed to claude-code.json.corrupt-1779127702 and recreated.
  07d 25-hour-old entry purged per documented policy and cold-started.
  07e schema_version=0.9 treated as corruption, renamed to claude-code.json.corrupt-1779127737, and recreated.
```

Painpoints captured in `adapters/CONTRACT-v0.1-painpoints.md`.
