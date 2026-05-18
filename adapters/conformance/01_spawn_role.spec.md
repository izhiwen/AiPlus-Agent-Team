# Conformance #01 — spawn_role

**Validates**: CONTRACT §1, §2, §5. The minimum viable path: adapter receives
AdapterRequest, runs runtime in the right worktree, emits valid AdapterResult.

**Pre-conditions**:
- A clean git worktree exists at `<harness_tmp>/worktree-role/`
- No prior session mapping entry for the test UUIDv7
- Runtime binary is on PATH

**Action**: send `AdapterRequest{session_id=<fresh UUIDv7>, role=engineer-a, task="run 'pwd' and report the directory you are in", worktree_path=<abs path>, allowed_tools=["Bash"], continuation_hint="new"}`.

**Pass criteria**:
- `AdapterResult.exit_status == "OK"`
- `AdapterResult.partial == false`
- `AdapterResult.final_text` contains the worktree path
- `AdapterResult.session_id` echoes the request UUID
- Schema validation: all 5 mandatory fields present, types match
- A new entry exists in `.aiplus/agent-team/sessions/<adapter>.json`

**Fail conditions**: spawn timeout > 60s, schema invalid, wrong cwd, missing mandatory field.
