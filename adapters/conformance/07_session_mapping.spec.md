# Conformance #07 — Session ID mapping

**What it proves**: `sessions/<adapter>.json` mapping correctly bridges CEO's logical UUIDv7 to runtime-internal session IDs, survives normal usage, and fails closed (cold-start) on corruption.

**Sub-cases**:

**07a — New session creates mapping entry**: fresh UUIDv7, no prior entry. Pass: after dispatch, `sessions/<adapter>.json` contains entry with `runtime_session_id` non-empty, `created_at` ≈ now, `last_used_at` ≈ now, `role` matches request.

**07b — Repeat session resumes runtime**: same UUIDv7 dispatched twice; first task `task="remember the number 7919"`, second task `task="what number did I just tell you?"`. Pass: second `final_text` contains `7919` (proves runtime session continued, not cold-started). `last_used_at` of the entry advanced.

**07c — Corruption recovery**: manually corrupt `sessions/<adapter>.json` (e.g., truncate to invalid JSON). Dispatch with any UUIDv7 (new or previously-mapped). Pass: adapter drops the corrupted file (renames to `sessions/<adapter>.json.corrupt-<ts>` or deletes), recreates with the new entry, treats current dispatch as cold start. `aiplus agent doctor` surfaces a warning for the corruption event. NEVER silently proceeds with broken mapping.

**07d — Aging policy**: pre-seed `sessions/<adapter>.json` with an entry whose `last_used_at` is 25 hours ago. Dispatch with that UUIDv7. Adapter MAY (not MUST) purge and cold-start; whichever the adapter chose, its `adapters/<runtime>/IMPLEMENTATION.md` §6 documents the policy, and the conformance assertion follows that documentation. Goal: behavior is consistent and documented, not undefined.

**07e — Schema version drift**: mapping file has `schema_version="0.9"` (pre-1.0). Pass: adapter rejects load, treats as corruption (07c path).

**Fail modes covered**: silent bad-mapping reuse, schema downgrade compatibility creep, runtime session ID leak across UUIDv7s.
