# CLAUDE.md — Agent Operating Instructions (Phase 0)

> This file tells the coding agent how to work in this repository.
> It is read automatically by Claude Code, Cursor, and similar tools.
> Do NOT modify this file unless instructed by the human operator.
>
> **This is Phase 0** — you read spec files directly from the filesystem.
> No MCP server, no structured queries. Just you, the spec, and the code.
>
> When the project transitions to Phase 1+ (MCP server running), replace
> this file with specs/CLAUDE-MCP.md for tool-based spec access.

---

## Identity

You are implementing a system defined by natural language specifications (nlspecs).
The specs are the source of truth. Code is a derived artifact. When the spec and
code disagree, the spec is correct and the code must change.

---

## Repository Structure

```
/
├── CLAUDE.md                  # THIS FILE — your operating instructions
├── specs/
│   ├── TEMPLATE.md            # The nlspec template (reference only, do not implement)
│   ├── {project}-spec.md      # Main spec for each project
│   └── patches/
│       └── PATCH-{nnn}.md     # Active patches (temporary, absorbed at version bumps)
├── src/                       # Implementation (you write this)
├── tests/
│   ├── scenarios/             # Scenario tests (derived from spec Section 10)
│   └── smoke/                 # Smoke tests tagged [SMOKE]
└── config/                    # Configuration files
```

---

## Operating Modes

You operate in one of six modes. The human tells you which mode, or you infer it
from the instruction. If ambiguous, ask.

### MODE: SPEC

```
Trigger:  "spec", "refine", "review spec", "write spec", "help me spec", "add scenarios"
Input:    A spec file (new or existing) and human guidance on what to improve
Output:   An improved spec file
Process:
  1. Read the spec (or start from NLSPEC-TEMPLATE.md if new)
  2. Analyze for completeness and consistency:
     a. Are all 15 sections present?
     b. Does every FUNCTION have USES and THROWS declarations?
     c. Does every RECORD have USED BY references?
     d. Does every SCENARIO have [SEC:] tags?
     e. Are there sections with no SCENARIOs (untested)?
     f. Are there RECORDs with no USED BY (orphaned)?
     g. Do all cross-references resolve (no dangling USES/THROWS)?
  3. If the human asked for specific help (e.g., "add scenarios for Section 5"),
     focus there. Otherwise, report findings and suggest improvements.
  4. Propose changes — do NOT modify the spec without human approval
  5. When approved, make the changes to the spec file
Validation: Spec is internally consistent. Human approves all changes.

IMPORTANT:
  - In SPEC mode you work on the SPEC, not the code.
  - You may create, modify, or reorganize spec content.
  - You do NOT write implementation code in this mode.
  - You do NOT modify scenarios without human approval (scenarios are the contract).
  - When adding FUNCTIONs, always add USES/THROWS declarations.
  - When adding RECORDs, always add USED BY references.
  - When adding SCENARIOs, always add [SEC:] tags and a tier ([SMOKE], [AFFECTED], [FULL]).
```

### MODE: DESCRIBE

```
Trigger:  "describe", "explain", "walk me through", "what does", "how does",
          "summarize", "overview", "tell me about", "onboard me"
Input:    A spec file (or specific section/element)
Output:   Human-readable explanation. No modifications to anything.
Process:
  1. Read the requested spec, section, or element
  2. Explain it in plain language:
     - What does this system/component/function do?
     - How do the parts relate to each other?
     - What are the key data flows?
     - What does the architecture look like?
  3. Adapt depth to the question:
     - "What does this system do?" → high-level summary (abstract + architecture)
     - "Explain Section 5.3" → detailed walkthrough of that function
     - "How does auth work?" → trace the auth flow through records, functions, endpoints
     - "What scenarios cover storage?" → list scenarios tagged with storage sections
  4. If asked, produce visual aids:
     - Component diagrams (mermaid)
     - Data flow descriptions
     - Element inventories ("47 records, 23 functions, 15 scenarios")
     - Dependency maps ("Entry is used by: storage_get, storage_set, storage_delete")

IMPORTANT:
  - In DESCRIBE mode you are READ-ONLY. Do not modify any files.
  - Do not suggest improvements unless explicitly asked.
  - Do not critique the spec — just explain it.
  - If you spot something clearly broken (e.g., dangling reference), you may
    mention it briefly, but do not derail into SPEC mode.
  - This mode is for understanding, not editing.
```

### MODE: IMPLEMENT

```
Trigger:  "implement", "build", "create", "start"
Input:    Full spec (specs/{project}-spec.md)
Output:   Complete codebase matching the spec
Process:
  1. Read the FULL spec
  2. Start with Section 12 (File Structure) — create the directory layout
  3. Implement Section 4 (Data Model) — all RECORDs, ENUMs, type aliases
  4. Implement Section 7 (Error Model) — all error types
  5. Implement Section 5 (Core Functions) — in dependency order
  6. Implement Section 6 (API Surface) — endpoints wired to core functions
  7. Implement Section 8 (Configuration) — config loading
  8. Write tests for ALL Section 10 scenarios
  9. Run all tests. Fix until ALL pass.
  10. Implement Section 14 (Build and Run) — verify build commands work
Validation: ALL scenarios pass. No exceptions.
```

### MODE: FIX

```
Trigger:  "fix", "bug", "broken", "failing", "patch"
Input:    A bug description or failing scenario number.
          You may also receive a PATCH spec.
Output:   Targeted code changes (specific files, not full rewrite)
Process:
  1. ASSEMBLE YOUR OWN CONTEXT SLICE (see Section: Context Assembly below)
     a. Read the spec's Table of Contents ONLY (Section headers, ~50 lines)
     b. From the bug description, identify which FUNCTION or ENDPOINT is affected
     c. Read ONLY that spec section (e.g., Section 5.3)
     d. Note which RECORDs the function references — read ONLY those from Section 4
     e. Note which errors it can throw — read ONLY those from Section 7
     f. Find scenarios tagged [SEC:5.3] — read ONLY those from Section 10
     g. STOP READING. You now have your context slice.
  2. If a PATCH spec was provided, read that too
  3. Identify the root cause in the existing code
  4. Fix ONLY the affected code
  5. Do NOT refactor unrelated code
  6. Do NOT modify files unrelated to the fix
  7. Run SMOKE scenarios + AFFECTED scenarios (by [SEC:x.x] tags)
  8. If all pass, you're done
  9. If SMOKE fails, something fundamental broke — stop and report
Validation: Failing scenario now passes. SMOKE scenarios still pass.
```

### MODE: VALIDATE

```
Trigger:  "validate", "test", "verify", "check", "nightly"
Input:    Nothing new (or full spec for full validation)
Output:   Test results report
Process:
  1. Run ALL scenario tests (full suite)
  2. Report: which pass, which fail, performance numbers
  3. Do NOT fix anything — just report
  4. If failures found, list the failing scenarios with section tags
Validation: Report only. Human decides next action.
```

### MODE: CONSOLIDATE

```
Trigger:  "consolidate", "absorb patches", "version bump", "clean up"
Input:    Full updated spec (patches absorbed into main spec)
Output:   Refactored codebase aligned with consolidated spec
Process:
  1. Read the FULL updated spec
  2. Compare current code against updated spec
  3. Refactor where spec has changed (absorbed patches may have clarified behavior)
  4. Remove any workarounds that patches introduced
  5. Run ALL scenario tests
  6. Clean up: remove dead code, align naming, update comments
Validation: ALL scenarios pass. Code is clean and aligned with consolidated spec.
```

---

## Context Assembly — How You Build Your Own Slice

In FIX mode, you assemble your own context from the spec file on disk.
You do NOT read the whole spec. You navigate it surgically.

### Step-by-step context assembly:

```
STEP 1: Read the Table of Contents only.
  Command: Read the first ~60 lines of specs/{project}-spec.md
  Purpose: Get the section map. Know where everything lives.
  Cost: ~60 lines, minimal tokens.

STEP 2: Identify the affected section.
  From the bug description or failing scenario, determine which
  FUNCTION (Section 5.x) or ENDPOINT (Section 6.x) is broken.
  If given a scenario number, read that scenario's [SEC:] tags.

STEP 3: Read the affected section.
  Command: Read Section 5.x (or 6.x) from the spec.
  This tells you the function's inputs, outputs, behavior, errors.
  Cost: ~50-150 lines.

STEP 4: Read referenced RECORDs.
  The function definition references RECORDs by name (e.g., "takes a QueryRequest,
  returns a QueryResponse"). Read ONLY those RECORD definitions from Section 4.
  Do NOT read all of Section 4.
  Cost: ~20-50 lines per RECORD.

STEP 5: Read referenced errors.
  The function lists THROWS errors. Read ONLY those error definitions from Section 7.
  Cost: ~10-30 lines.

STEP 6: Read the affected scenarios.
  Find scenarios tagged with the affected section's number.
  Read ONLY those.
  Cost: ~20-50 lines per scenario.

STEP 7: Stop reading. You have your slice.
  Total context: typically 200-500 lines.
```

### Navigation helpers in the spec:

The spec is designed to support this workflow. Look for these markers:

```
In FUNCTION definitions:
  USES: RecordA, RecordB           -- tells you which RECORDs to read
  THROWS: ErrorX, ErrorY           -- tells you which errors to read

In SCENARIO definitions:
  [SEC:5.3] [SEC:4.1] [SMOKE]     -- tells you which sections this validates

In RECORD definitions:
  USED BY: function_a, function_b  -- tells you which functions reference this
```

### When context is insufficient:

```
RULE: If you need a RECORD definition you don't have, read it from the spec.
  Do NOT ask the human. Navigate the spec file yourself.

RULE: If you discover the bug spans multiple sections (e.g., Section 5.3 AND 5.1),
  read the additional section. But REPORT this to the human:
  "Fix requires changes across Sections 5.1 and 5.3. Expanded context."

RULE: If you genuinely cannot determine which section is affected from the bug
  description, ask the human: "Which spec section does this relate to?"
  Do NOT read the full spec to figure it out.
```

---

## Scenario Execution Rules

```
SMOKE scenarios [SMOKE]:
  - Run on EVERY change, in every mode except VALIDATE
  - Must pass before any other validation
  - If SMOKE fails, stop immediately and report
  - Target: < 30 seconds total

AFFECTED scenarios [SEC:x.x]:
  - Run when the corresponding section is modified
  - Determined by [SEC:] tags in scenario headers
  - In FIX mode, run all scenarios matching the changed section(s)

FULL suite:
  - Run in VALIDATE and CONSOLIDATE modes
  - Run nightly if automation is set up
  - Includes performance scenarios — these may be slow
```

---

## Code Style Rules

```
RULE: Match the language idioms specified in Section 1 of the spec.
RULE: Error handling must match Section 7 exactly — no swallowing errors.
RULE: Every public function must have a doc comment referencing the spec section.
      Example: /// Implements Section 5.3: query_execute
RULE: Every test file must reference the scenario number.
      Example: // SCENARIO 7: Query returns results within latency budget
RULE: Do not add features not in the spec. No "helpful" extras.
RULE: Do not optimize unless the spec's PERFORMANCE targets require it.
RULE: Prefer clarity over cleverness. The next agent (or human) must read this.
```

---

## Communication Rules

```
WHEN you complete a task:
  - State which scenarios you ran and their results
  - List files modified
  - State the mode you were in

WHEN you encounter an ambiguity:
  - Quote the exact spec text that's ambiguous
  - Propose two interpretations
  - Ask which is correct
  - Do NOT pick one and proceed silently

WHEN you think the spec is wrong:
  - State: "Possible spec deficiency in Section X.X: {description}"
  - Explain why you think it's wrong
  - Propose a fix
  - Wait for human decision — do NOT modify the spec yourself

WHEN a scenario fails and you can't fix it:
  - State which scenario fails
  - Show the expected vs actual behavior
  - State what you tried
  - Ask for expanded context or guidance
```

---

## File Naming Conventions

```
Spec files:       {project}-spec.md
Patch files:      PATCH-{NNN}-{short-description}.md (e.g., PATCH-001-fix-query-timeout.md)
Scenario tests:   scenario_{NNN}_{short_name}.{ext}
Smoke tests:      smoke_{NNN}_{short_name}.{ext}
```

---

## Cross-Spec References

Some specs IMPORT definitions from other specs using:
```
IMPORT {Name} FROM {spec-file} Section {x.x}
```

When you encounter an IMPORT:
1. Check if the referenced spec file exists in specs/
2. If yes, read ONLY the referenced section (not the whole file)
3. If no, ask: "I need {spec-file} Section {x.x} for the definition of {Name}."
4. Treat imported definitions as read-only — never modify another spec's RECORDs

---

## Anti-Patterns — Do NOT Do These

```
❌ Do NOT read the full spec when in FIX mode (wasteful, loses focus)
❌ Do NOT rewrite files unrelated to the current task
❌ Do NOT add dependencies not listed in Section 11
❌ Do NOT modify scenario tests to match your code (scenarios are immutable)
❌ Do NOT skip SMOKE tests (they catch cascading breakage)
❌ Do NOT "improve" code style when fixing a bug (separate concerns)
❌ Do NOT create new public API not defined in Section 6
❌ Do NOT assume — ask when the context slice is insufficient
```

---

## Transition to Phase 1

When the project transitions to MCP-based spec management:
1. Replace this file with `specs/CLAUDE-MCP.md`
2. The agent will use MCP tools instead of reading spec files directly
3. Context slicing becomes automatic (nlspec_slice) instead of manual
4. All operating modes remain the same — only the access method changes
