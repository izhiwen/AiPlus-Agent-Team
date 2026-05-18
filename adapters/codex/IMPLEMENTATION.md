# Codex Adapter Implementation

Runtime: Codex CLI. Contract target: `adapters/CONTRACT.md` v1 FROZEN at commit `2affaa9`.

## 1. Transport

The adapter is a project-local Python 3 script that reads one `AdapterRequest` JSON object from stdin, or from `--request-file <path>` for harness convenience. It emits runtime stdout verbatim and writes the final `AdapterResult` either as the last stdout JSON line or to `--output-file <path>`.

Runtime transport is subprocess execution:

```text
codex exec --json --output-schema <schema.json> --output-last-message <last-message.txt> ...
```

The adapter passes the full task package on stdin (`-`). The task package is copy-pasteable and contains, in order: resolved `system_prompt`, memory blocks in CEO-provided order (`personal`, `team`, `project`), the user task, and adapter execution constraints. Codex JSONL events are captured line-by-line into `stdout_raw`; the `--output-last-message` file is used as the primary final-text source when present.

## 2. Config Isolation

The adapter never inherits Owner global Codex config. Before spawning Codex it creates project-local isolation under:

```text
<worktree_path>/.aiplus/agent-team/codex-isolation/
```

It runs Codex with this isolation composition:

```text
CODEX_HOME=<isolated-codex-home>
codex exec --ignore-user-config --ignore-rules -c aiplus_adapter.marker="codex" ...
```

The isolated `CODEX_HOME` contains a generated `config.toml` with an adapter marker and no inherited Owner settings. The adapter validates before spawn that the isolated home exists, the marker file parses, the sentinel file exists, and the directory is writable. Validation failure emits `exit_status=ISOLATION_BREACH` and does not spawn Codex.

Auth provision follows CONTRACT v1 §6.1. CEO should spawn the adapter through:

```text
aiplus secret-broker run --aliases openai -- <adapter command>
```

The adapter subprocess env is restricted to `PATH`, `CODEX_HOME`, and recognized OpenAI credential variables already injected by CEO's broker wrapper (`OPENAI_API_KEY`, plus optional OpenAI endpoint/project/org variables if present). It does not read keychain, `~/.codex/`, `~/.aiplus/`, or any Owner-global config directory.

Codex CLI v0.130.0 does not treat `OPENAI_API_KEY` alone as an authenticated login state when `CODEX_HOME` is newly isolated. When `OPENAI_API_KEY` is present, the adapter first runs:

```text
CODEX_HOME=<isolated-codex-home> codex login --with-api-key
```

with the key supplied on stdin. This writes auth material only inside the isolated project-local `CODEX_HOME`; it does not read or write Owner `~/.codex`.
The adapter removes the isolated auth file after the Codex subprocess exits. If the adapter itself is killed with SIGKILL, the file may remain inside the project-local isolated directory; CEO/harness cleanup removes the temporary worktree after conformance.

The isolated config and CLI overrides also set:

```text
web_search = "disabled"
```

because the Codex CLI may otherwise include the hosted `web_search_preview` tool in Responses requests, and that hosted tool can be organization-disabled. The adapter contract does not require web search for any conformance case.

For conformance #03 the adapter includes the isolated sentinel in the task package when the task asks for config/settings inspection, and it verifies Owner global marker paths listed in `adapters/conformance/_harness/global_config_markers.txt` are never used as configured Codex paths. The positive proof is the isolated sentinel, not a read of Owner global config.

## 3. cwd Injection

The adapter passes the absolute worktree path through Codex's native cwd flag:

```text
codex exec -C <worktree_path> ...
```

It also sets subprocess `cwd=<worktree_path>` for ordinary Unix path behavior.

Before spawn it verifies:

1. `worktree_path` is absolute.
2. The directory exists.
3. `git -C <worktree_path> worktree list --porcelain` includes the same real path.
4. The worktree is clean unless `override_dirty_worktree=true`.

Failure emits `exit_status=WORKSPACE_NOT_PROVISIONED`, `partial=true`, and no Codex subprocess is spawned.

## 4. Tool Whitelist Translation

Codex does not expose a CONTRACT-style per-tool whitelist. The adapter maps `allowed_tools[]` to the nearest native sandbox policy:

```text
no write/exec tools requested      -> --sandbox read-only
Bash/Edit/Write/MultiEdit allowed  -> --sandbox workspace-write
```

The adapter never uses Codex's `--dangerously-bypass-approvals-and-sandbox` by default. Harnesses may opt into `danger-full-access` by setting:

```text
AIPLUS_CODEX_SANDBOX=danger-full-access
```

This remains a coarse runtime-native sandbox mapping, not per-tool enforcement. The requested `allowed_tools[]` is still included in the task package for model-side instruction and traceability. This granularity gap is an adapter-level Codex detail and not a `_CONTRACT-BUG_`.

## 5. `usage_tokens` Extraction

Codex `--json` JSONL events are parsed line-by-line. The adapter searches every JSON event for common Codex/OpenAI usage shapes:

```text
usage.input_tokens
usage.output_tokens
usage.total_tokens
token_usage.input_tokens
token_usage.output_tokens
tokens.input
tokens.output
```

The populated `AdapterResult.usage_tokens` object is:

```json
{"input": <input>, "output": <output>, "total": <sum or reported total>}
```

If no token usage is present, the adapter emits `usage_tokens=null` rather than failing. Conformance #05 expects Codex to report non-null positive usage; absence is recorded as a conformance failure, not an adapter exception.

## 6. Session Continuation

CEO UUIDv7 session IDs map to Codex runtime session IDs in:

```text
<worktree_path>/.aiplus/agent-team/sessions/codex.json
```

For a new CEO session, the adapter starts Codex with `codex exec ...`. If Codex JSONL exposes a runtime session ID, the adapter records it. If the runtime session ID is opaque or absent, the adapter uses CONTRACT §4.3 and records the CEO UUIDv7 as the runtime session ID.

For `continuation_hint=continue`, the adapter loads the mapping and passes:

```text
codex exec resume <runtime_session_id> ... -
```

Codex CLI v0.130.0 exposes a narrower `exec resume` option surface than `exec`: resume accepts JSONL and last-message capture, but not `-C`, `--sandbox`, or `--output-schema`. For resumed calls, the adapter sets subprocess `cwd=<worktree_path>` and keeps the same task-package instructions; first dispatches still use `-C <worktree_path>` and `--sandbox <policy>`.

Mapping policy:

- JSON parse failure or schema-version mismatch is treated as corruption. The file is renamed to `codex.json.corrupt-<unix-ts>` and the current dispatch cold-starts.
- Entries older than 30 days are purged.
- Entries older than 24 hours are purged by this implementation, so conformance #07d should expect cold-start after 25 hours.
- If the mapping file exceeds 10 MB, oldest entries are purged until it is below the threshold.

## 7. Trace Event Mapping

The adapter writes project-local JSONL trace entries to:

```text
<worktree_path>/.aiplus/agent-team/adapter-trace/<session_id>.jsonl
```

Each dispatch records the redacted `AdapterRequest` before spawn and the redacted `AdapterResult` after completion unless `.aiplus/agent-team/trace-disabled` exists. Trace rotation starts a new file when the current file is over 100 MB or contains entries older than 7 days.

Codex JSONL events map to `AdapterResult` as follows:

- `stdout_raw`: every Codex stdout line, joined verbatim.
- `final_text`: `--output-last-message` file when populated, otherwise the last user-visible assistant/result text found in JSONL events.
- `tool_calls`: discoverable tool/command events become `{tool, args_summary, outcome}` entries; absent or unparseable tool events emit `null`.
- `usage_tokens`: parsed as described in section 5.
- `exit_status`: `OK` for natural zero exit, otherwise mapped to CONTRACT §3 values.
- `partial`: `true` for validation failures, runtime failures, interrupts, fallback, and any non-natural completion.

SIGINT/SIGTERM handling follows CONTRACT §7: first interrupt asks Codex to terminate, preserves partial stdout, escalates to SIGTERM after 5 seconds and SIGKILL after 10 seconds total, then emits `OWNER_INTERRUPTED` when possible.

## Conformance Run Log

Final round-3 run against CONTRACT v1 FROZEN (`2affaa9`) on 2026-05-18:

```text
#01 spawn_role: PASS
  Wrapper: aiplus secret-broker run --aliases openai -- adapters/codex/aiplus-codex-adapter ...
  Evidence: exit_status=OK, partial=false,
            final_text="I am in /private/tmp/aiplus-codex-conf01-8iD6ce/worktree-role",
            session mapping created with runtime_session_id=019e3cc4-4d5c-73b1-bc7f-14e868979014,
            usage_tokens={"input":46938,"output":198,"total":47136}.
  Implementation correction during run: isolated CODEX_HOME needs `codex login --with-api-key`
            from CEO-provided OPENAI_API_KEY before `codex exec`.
  Implementation correction during run: set `web_search="disabled"` because hosted
            `web_search_preview` is organization-disabled and not needed for conformance.

#02 inject_memory: PASS
  Evidence: final_text contains `const x = 5` and says project memory overrides the
            personal `var` preference.

#03 config_isolation: PASS
  Evidence: final_text contains sentinel: "isolation-test-1779136003";
            no Owner global config path (`~/.codex`, `~/.claude`, `~/.config/opencode`) appeared.

#06 fallback_to_print: PASS
  Evidence: PATH=/nonexistent produced fallback block, process exit 75,
            AdapterResult.exit_status=RUNTIME_NOT_FOUND, partial=true,
            trace file contained AdapterRequest.

#04 failure_modes: PASS
  04a WORKSPACE_NOT_PROVISIONED: exit_status=WORKSPACE_NOT_PROVISIONED, partial=true, no runtime spawn.
  04b ISOLATION_BREACH: forced isolation break produced exit_status=ISOLATION_BREACH in <3s.
  04c RUNTIME_NOT_FOUND: fallback block + exit 75 + exit_status=RUNTIME_NOT_FOUND.
  04d OWNER_INTERRUPTED/SIGINT: fake Codex streamed numbers, SIGINT produced exit_status=OWNER_INTERRUPTED,
      partial=true, stdout_raw retained partial numbers.
  04e SIGTERM escalation: fake Codex ignored SIGINT, adapter sent SIGTERM before SIGKILL,
      marker file written, exit_status=OWNER_INTERRUPTED, partial=true.
  04f OWNER_HARD_INTERRUPTED/SIGKILL: adapter killed with SIGKILL, no AdapterResult emitted,
      parent wait status=137, trace retained AdapterRequest.

#05 token_report: PASS
  Evidence: final_text=hello world, usage_tokens={"input":18532,"output":34,"total":18566}.

#07 session_mapping: PASS
  07a PASS live Codex: new mapping entry created.
  07b PASS live Codex: repeated dispatch resumed same Codex thread and returned 7919.
  07c PASS fake runtime: corrupt JSON renamed and mapping recreated.
  07d PASS fake runtime: 25-hour-old entry purged per documented policy and cold-started with fresh mapping.
  07e PASS fake runtime: schema_version=0.9 treated as corruption, renamed, and recreated.
```

Painpoints captured in `adapters/CONTRACT-v1-painpoints.md`. No `_CONTRACT-BUG_` or `_CODEX-BUG_` painpoints were found.
