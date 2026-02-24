# CLAUDE-MCP.md — Agent Operating Instructions (Phase 1+)

> This file tells the coding agent how to work in this repository.
> It is read automatically by Claude Code, Cursor, and similar tools.
> Do NOT modify this file unless instructed by the human operator.
>
> **This is Phase 1+** — the nlspec MCP server is running. You access specs
> via MCP tools (nlspec_get, nlspec_search, nlspec_slice, etc.), not by
> reading raw markdown files.
>
> If the MCP server is not running, fall back to CLAUDE.md (Phase 0).

---

## Identity

You are implementing a system defined by natural language specifications (nlspecs).
The specs are the source of truth. Code is a derived artifact. When the spec and
code disagree, the spec is correct and the code must change.

You have access to the nlspec MCP server. Use its tools to read, query, and
modify specs. Do NOT read spec files directly from the filesystem — the MCP server
provides structured, indexed access.

---

## Available MCP Tools

### Bootstrap Tools (Phase 1+)

```
nlspec_get          Read a specific element or section
nlspec_list         List specs, sections, or elements with filters
nlspec_search       Text and structural search across specs
nlspec_create       Add a new element to a spec
nlspec_update       Modify an existing element
nlspec_delete       Remove an element from a spec
nlspec_init         Initialize a new spec from template
```

### Extended Tools (Phase 2+)

```
nlspec_import       Import a spec file into the system under a namespace
nlspec_namespaces   List all namespaces and their specs
nlspec_slice        Extract minimal context for a bug fix
nlspec_validate     Check a spec's structural integrity
nlspec_graph        Get the dependency graph for a spec or element
nlspec_split        Analyze and decompose a monolith spec
nlspec_patch_create Create a tracked patch for a spec
nlspec_patch_list   List patches with filters
nlspec_patch_absorb Merge a patch back into the main spec
```

All tools accept an optional `namespace` parameter. When omitted, queries
search across all namespaces.

---

## Operating Modes

You operate in one of four modes. The human tells you which mode, or you infer it
from the instruction. If ambiguous, ask.

### MODE: IMPLEMENT

```
Trigger:  "implement", "build", "create", "start"
Input:    A spec identified by namespace and spec_id
Output:   Complete codebase matching the spec
Process:
  1. Call nlspec_get to read the full spec, section by section
  2. Start with Section 12 (File Structure) — create the directory layout
  3. Implement Section 4 (Data Model) — call nlspec_list({element_type: "RECORD"})
  4. Implement Section 7 (Error Model) — call nlspec_get({section: "7"})
  5. Implement Section 5 (Core Functions) — call nlspec_list({element_type: "FUNCTION"})
  6. Implement Section 6 (API Surface) — call nlspec_list({element_type: "ENDPOINT"})
  7. Implement Section 8 (Configuration) — call nlspec_list({element_type: "CONFIG"})
  8. Write tests for ALL Section 10 scenarios — call nlspec_list({element_type: "SCENARIO"})
  9. Run all tests. Fix until ALL pass.
  10. Implement Section 14 (Build and Run) — verify build commands work
Validation: ALL scenarios pass. No exceptions.
```

### MODE: FIX

```
Trigger:  "fix", "bug", "broken", "failing", "patch"
Input:    A bug description or failing scenario number.
Output:   Targeted code changes (specific files, not full rewrite)
Process:
  1. USE nlspec_slice TO GET YOUR CONTEXT (see Context Slicing below)
     - If given a scenario number:
       nlspec_slice({scenario: 7})
     - If given a section/function:
       nlspec_slice({section: "5.3"})
     - The slice contains ONLY the elements you need.
  2. If a PATCH exists, call nlspec_patch_list to find it
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
Input:    Nothing new
Output:   Test results report + spec integrity check
Process:
  1. Call nlspec_validate to check spec structural integrity
  2. Run ALL scenario tests (full suite)
  3. Report: which pass, which fail, performance numbers
  4. Report: any validation warnings (orphaned records, untested sections)
  5. Do NOT fix anything — just report
Validation: Report only. Human decides next action.
```

### MODE: CONSOLIDATE

```
Trigger:  "consolidate", "absorb patches", "version bump", "clean up"
Input:    Spec with patches to absorb
Output:   Refactored codebase aligned with consolidated spec
Process:
  1. Call nlspec_patch_list({status: "ACTIVE"}) to find all active patches
  2. For each patch, call nlspec_patch_absorb to merge it into the spec
  3. Compare current code against updated spec
  4. Refactor where spec has changed
  5. Remove any workarounds that patches introduced
  6. Run ALL scenario tests
  7. Call nlspec_validate to verify spec integrity
  8. Clean up: remove dead code, align naming, update comments
Validation: ALL scenarios pass. Spec validates clean. Code aligned.
```

---

## Context Slicing — How You Get Your Context

In FIX mode, the MCP server assembles your context for you.
You do NOT read the whole spec. You do NOT navigate manually.

### Automatic context slicing:

```
STEP 1: Call nlspec_slice.
  If you have a scenario number:
    nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  If you have a section number:
    nlspec_slice({namespace: "myproject", spec_id: "auth", section: "5.3"})

STEP 2: Read the slice.
  The slice contains:
  - The triggering scenario or section
  - All FUNCTIONs in tagged sections
  - All RECORDs referenced by those FUNCTIONs (via USES)
  - All error types referenced (via THROWS)
  - All related SCENARIOs
  - Cross-spec elements reached via IMPORTs

STEP 3: That's it. You have your context.
  Total: typically 200-500 lines (10-30 elements).
  The server did the graph traversal for you.
```

### When you need more context:

```
RULE: If the slice is insufficient, use nlspec_search to find related elements.
  nlspec_search({query: "token expiry", element_type: "FUNCTION"})

RULE: If you discover the bug spans multiple sections, call nlspec_slice on
  each section and merge the results. Report this to the human:
  "Fix requires changes across Sections 5.1 and 5.3. Expanded context."

RULE: Use nlspec_graph to understand impact before making changes:
  nlspec_graph({element_id: "myproject/auth:4.1:record:Token", direction: "incoming"})
  → Shows you everything that depends on Token
```

---

## Scenario Execution Rules

```
SMOKE scenarios [SMOKE]:
  - Run on EVERY change, in every mode except VALIDATE
  - Must pass before any other validation
  - If SMOKE fails, stop immediately and report
  - Target: < 30 seconds total
  - Find them: nlspec_list({element_type: "SCENARIO", tags: ["SMOKE"]})

AFFECTED scenarios [SEC:x.x]:
  - Run when the corresponding section is modified
  - Determined by [SEC:] tags in scenario headers
  - In FIX mode, run all scenarios matching the changed section(s)
  - Find them: nlspec_search({tags: ["SEC:5.3"], element_type: "SCENARIO"})

FULL suite:
  - Run in VALIDATE and CONSOLIDATE modes
  - Run nightly if automation is set up
  - Includes performance scenarios — these may be slow
  - Find them: nlspec_list({element_type: "SCENARIO"})
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
  - If approved, create a patch: nlspec_patch_create({category: "SPEC_DEFICIENCY", ...})

WHEN a scenario fails and you can't fix it:
  - State which scenario fails
  - Show the expected vs actual behavior
  - State what you tried
  - Use nlspec_graph to show what else might be affected
  - Ask for guidance
```

---

## File Naming Conventions

```
Spec files:       {project}-spec.md
Patch files:      Managed by nlspec_patch_create (auto-named PATCH-{NNN})
Scenario tests:   scenario_{NNN}_{short_name}.{ext}
Smoke tests:      smoke_{NNN}_{short_name}.{ext}
```

---

## Working with Multiple Specs (Phase 2+)

When the project has multiple specs in namespaces:

```
RULE: Use namespace-qualified queries:
  nlspec_get({namespace: "myproject", spec_id: "auth", section: "5.1"})

RULE: When fixing a bug, nlspec_slice follows cross-spec imports automatically.
  The slice may include elements from multiple specs. This is expected.

RULE: Never modify another spec's elements directly. If a bug requires changes
  to an imported spec, create a patch against THAT spec:
  nlspec_patch_create({namespace: "myproject", spec_id: "storage", ...})

RULE: Before making changes to a shared RECORD, check impact:
  nlspec_graph({element_id: "...", direction: "incoming"})
  This shows every function and spec that depends on it.

RULE: After any structural change, validate:
  nlspec_validate({namespace: "myproject", spec_id: "auth"})
```

---

## Anti-Patterns — Do NOT Do These

```
❌ Do NOT read spec files directly from the filesystem (use MCP tools)
❌ Do NOT read the full spec when in FIX mode (use nlspec_slice)
❌ Do NOT rewrite files unrelated to the current task
❌ Do NOT add dependencies not listed in Section 11
❌ Do NOT modify scenario tests to match your code (scenarios are immutable)
❌ Do NOT skip SMOKE tests (they catch cascading breakage)
❌ Do NOT "improve" code style when fixing a bug (separate concerns)
❌ Do NOT create new public API not defined in Section 6
❌ Do NOT modify imported elements from other specs (patch the source spec)
❌ Do NOT assume — use nlspec_search and nlspec_graph when context is insufficient
```
