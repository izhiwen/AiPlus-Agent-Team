# Conformance #02 — inject_memory

**Validates**: CONTRACT §1 (memory field) + DESIGN.md §8 read order (personal < team < project).

**Pre-conditions**: 3 fixture memory blobs with deliberate conflict:
- personal: "I am Engineer A. I prefer `var` for variable declarations."
- team: "Be terse in code reviews. Always cite line numbers."
- project: "**Always use `const`, never `var`**, project-wide rule."

**Action**: send AdapterRequest carrying all three memory blobs + task: "Write a one-line JS variable declaration of `x = 5` and briefly justify your choice."

**Pass criteria**:
- `AdapterResult.final_text` contains `const x = 5` (project layer wins)
- `final_text` references the project rule, NOT the personal preference
- Schema valid, exit_status=OK

**Fail conditions**: agent uses `var` (personal won — wrong precedence), agent ignores all 3 (memory not injected), schema invalid.

**Notes**: this test is the cleanest validation that authority order is honored. Run it on every adapter; if it fails, the adapter's memory-merging or system-prompt construction is broken.
