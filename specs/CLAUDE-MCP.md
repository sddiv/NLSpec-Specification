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

Specs have a TYPE: SYSTEM (running code), PATTERN (reusable blueprint), or ASSET
(static resources). Check the spec header to determine the type. PATTERN and ASSET
specs are consumed by SYSTEM specs via "USES PATTERN:" and "USES ASSET:" in Architecture.3.

---

## Spec Tolerance

Specs come in many shapes. Your job is to work with what's there, not demand perfection.

**Strict specs** have a SECTIONS declaration block, typed elements (RECORD, FUNCTION,
SCENARIO), cross-references (USES, THROWS, [SEC:] tags), and a declared TYPE. Full
tooling works. Gap detection is precise.

**Loose specs** have `## ` section headers but no SECTIONS block. Maybe they use
plain prose instead of formal RECORD/FUNCTION blocks. Maybe sections have unusual names.
The parser discovers structure from headers and pattern-matches elements. Everything
still works — slicing, implementation, validation — just with less precision.

**Minimal specs** might be a single markdown file with a description, some function
signatures, and a few behavioral notes. The agent does its best: extract what structure
exists, infer what's missing, and report gaps.

**Rules for tolerance:**
- Never refuse to parse or implement a spec because it's missing structure.
- Never insist a user add a SECTIONS block, formal elements, or specific sections.
- When structure is missing, INFER what you can and REPORT what you can't.
- Gap detection is always advisory. The human decides whether to fix gaps.
- The NLSPEC-TEMPLATE is a recommendation, not a requirement. Specs that diverge
  from it are first-class citizens.

**Gap detection via nlspec_validate:**
- The validate tool returns a GapReport with parse mode (strict/loose) and gaps
- CRITICAL gaps: missing definitions referenced by other elements, broken cross-refs
- WARNING gaps: FUNCTIONs without USES/THROWS, SCENARIOs without [SEC:] tags
- INFO gaps: missing common sections for the TYPE, large unstructured prose blocks
- In loose mode, "missing declared sections" gaps don't apply (nothing was declared)
- completeness score (0.0-1.0) gives a rough measure of spec maturity

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
nlspec_seed_resolve Walk dependency graph, resolve contracts, produce seed manifest
nlspec_seed_audit   Trace provenance of a specific seed rule
```

All tools accept an optional `namespace` parameter. When omitted, queries
search across all namespaces.

---

## Operating Modes

You operate in one of seven modes. The human tells you which mode, or you infer it
from the instruction. If ambiguous, ask.

**Pipeline: DESIGN → SPEC → IMPLEMENT → VALIDATE**

### MODE: SPEC

```
Trigger:  "spec", "refine", "review spec", "write spec", "help me spec", "add scenarios"
Input:    A spec (identified by namespace/spec_id) and human guidance
Output:   An improved spec
Process:
  1. Call nlspec_validate to get current integrity status and gap report
     - The validator reports parse mode (strict vs loose) and gaps by severity
  2. Call nlspec_list to survey all elements
  3. Review the gap report:
     - CRITICAL gaps: report immediately (missing definitions, broken refs)
     - WARNING gaps: include in findings (missing USES/THROWS, untagged scenarios)
     - INFO gaps: mention as suggestions (missing common sections, unstructured prose)
  4. Additional analysis:
     a. Call nlspec_search({element_type: "FUNCTION"}) — do all have USES and THROWS?
     b. Call nlspec_search({element_type: "RECORD"}) — do all have USED BY?
     c. Call nlspec_search({element_type: "SCENARIO"}) — do all have [SEC:] tags?
     d. Use nlspec_graph to find orphaned elements (no incoming references)
     e. Find sections with no SCENARIOs
  5. If the human asked for specific help (e.g., "add scenarios for Functions section"),
     focus there. Otherwise, report findings and suggest improvements.
  6. Propose changes — do NOT modify the spec without human approval
  7. When approved, use nlspec_create/nlspec_update to make changes
  8. Call nlspec_validate again to confirm improvements
Validation: Spec is internally consistent. Human approves all changes.

IMPORTANT:
  - In SPEC mode you work on the SPEC, not the code.
  - You may create, modify, or reorganize spec content via MCP tools.
  - You do NOT write implementation code in this mode.
  - You do NOT modify scenarios without human approval (scenarios are the contract).
  - When adding FUNCTIONs, always add USES/THROWS declarations.
  - When adding RECORDs, always add USED BY references.
  - When adding SCENARIOs, always add [SEC:] tags and a tier ([SMOKE], [AFFECTED], [FULL]).
  - Use nlspec_graph to understand impact before suggesting changes to shared elements.
  - For loose specs, SUGGEST adding a SECTIONS block but don't require it.
  - Never refuse to work with a spec because it lacks a SECTIONS declaration.

DEPENDENCY CONTRACT RULES (for specs with IMPORTS or EXPORTS):
  - When modifying Contracts section EXPORTS, call nlspec_graph({direction: "incoming"})
    to find all downstream consumers. Report the blast radius before proposing changes.
  - Call nlspec_seed_resolve with dry_run=true to verify the full contract chain.
  - For PATTERN specs: ensure all architectural constraints are exported.
  - For ASSET specs: ensure all design constraints are exported.
```

### MODE: DESIGN

```
Trigger:  "design", "architect", "propose architecture", "plan the system",
          "what patterns should", "design this", "lay out the architecture"
Input:    A problem statement (Problem section) or partial spec, plus human guidance
Output:   A partial spec: Abstract, Problem, Architecture, 15, 16 filled in. All other
          sections left as template placeholders.
Process:
  1. Read the problem statement via nlspec_get or from human's description
  2. Propose an architecture:
     a. Draw the ASCII diagram (Architecture section)
     b. Write the component inventory (Architecture section.1)
     c. Map the primary data flows (Architecture section.2)
  3. Select patterns:
     a. Call nlspec_get on pattern-catalog.md for prior art patterns.
        Write USES PATTERN declarations in Architecture.3.
     b. If novel patterns are needed, note them as future TYPE: PATTERN specs.
  4. Identify assets needed:
     a. Check existing ASSET specs via nlspec_list({type: "ASSET"}).
     b. Check workspace for asset files (styles/, assets/ directories).
     c. If asset files exist but no ASSET spec catalogs them, note that an
        ASSET spec needs to be created with FileStructure section (catalog) and Functions section
        (style guide with RULEs and TOKENs).
     d. Write USES ASSET declarations for existing or planned ASSET specs.
  5. Map dependencies:
     a. Call nlspec_graph to see the existing dependency landscape.
     b. Write IMPORTS for specs this system depends on (Appendix B).
     c. Write Contracts section EXPORTS and EXPECTS.
     d. Call nlspec_seed_resolve with dry_run=true to validate contracts.
  6. Define boundaries (Boundaries section)
  7. Write the abstract (Abstract section)
  8. Save partial spec via nlspec_patch and present for human review

OUTPUT FORMAT:
  - Abstract, Problem, Architecture (with 3.1, 3.2, 3.3), 15, 16: fully written
  - All other sections: template headers with one-line notes
  - A "Design Decisions" appendix listing key choices and alternatives

IMPORTANT:
  - DESIGN mode produces ARCHITECTURE, not code.
  - The output is a starting point for SPEC mode, which fills in the details.
  - Always explain WHY a pattern was chosen, not just which one.
  - For multi-spec systems, also produce a dependency diagram showing which
    specs import from which, and the build order.
```

### MODE: DESCRIBE

```
Trigger:  "describe", "explain", "walk me through", "what does", "how does",
          "summarize", "overview", "tell me about", "onboard me"
Input:    A spec (or specific section/element) identified by namespace/spec_id
Output:   Human-readable explanation. No modifications to anything.
Process:
  1. Use MCP tools to read the requested spec, section, or element:
     - nlspec_get for specific sections or elements
     - nlspec_list for element inventories
     - nlspec_graph for dependency maps
     - nlspec_search for finding related elements
  2. Explain it in plain language:
     - What does this system/component/function do?
     - How do the parts relate to each other?
     - What are the key data flows?
     - What does the architecture look like?
  3. Adapt depth to the question:
     - "What does this system do?" → high-level summary (abstract + architecture)
     - "Explain Functions section.3" → detailed walkthrough of that function
     - "How does auth work?" → trace the flow via nlspec_graph
     - "What scenarios cover storage?" → nlspec_search({tags: ["SEC:Functions.3"], element_type: "SCENARIO"})
  4. If asked, produce visual aids:
     - Component diagrams (mermaid)
     - Dependency graphs from nlspec_graph
     - Element inventories from nlspec_list
     - Coverage reports from nlspec_validate
  5. Check spec TYPE and adapt:
     - SYSTEM: explain what the system does and how
     - PATTERN: explain when to use it, what it constrains, how consumers apply it
     - ASSET: explain what resources are provided and what design constraints apply
  6. For dependency chain questions:
     - "What does this spec depend on?" → call nlspec_graph for the dependency tree.
       For each dependency, read its Contracts section EXPORTS.
     - "What does this spec export?" → call nlspec_get({section: "Contracts"})
     - "What patterns does this use?" → read Architecture.3, explain each pattern
     - "What would this system boot with?" → call nlspec_seed_resolve with dry_run=true
     - "Where does this constraint come from?" → call nlspec_seed_audit
     - "What breaks if I change this export?" → call nlspec_graph with direction=incoming

IMPORTANT:
  - In DESCRIBE mode you are READ-ONLY. Do not modify any specs or files.
  - Do not suggest improvements unless explicitly asked.
  - Do not critique the spec — just explain it.
  - If you spot something clearly broken (e.g., dangling reference), you may
    mention it briefly, but do not derail into SPEC mode.
  - This mode is for understanding, not editing.
```

### MODE: IMPLEMENT

```
Trigger:  "implement", "build", "create", "start"
Input:    A spec identified by namespace and spec_id
Output:   Complete codebase matching the spec
Process:
  1. Call nlspec_validate to get gap report and parse mode
     - If CRITICAL gaps: report to human, ask whether to proceed or fix spec first
     - If only WARNING/INFO gaps: note them and proceed
  2. Install dependencies:
     a. Call nlspec_list({element_type: "DEPENDENCY"}) to get all declarations
     b. Install each using the declared `install` command and `version`
     c. Run the `verify` check for each
     d. CRITICAL: After install, read the ACTUAL installed artifact's API surface,
        type definitions, or asset listing. Do NOT trust training knowledge for
        version-specific APIs. Use the installed reality when writing code.
  3. Adapt your implementation plan to what the spec provides:
     a. Call nlspec_get({section: "FileStructure"}) — if found, create the layout.
        If not found, infer from the language target and conventions.
     b. Call nlspec_list({element_type: "RECORD"}) — if found, implement all.
        If none, extract data structures from FUNCTION signatures.
     c. Call nlspec_get({section: "Errors"}) — if found, implement all error types.
        If not found, implement errors from THROWS declarations.
     d. Call nlspec_list({element_type: "FUNCTION"}) — implement in dependency order
     e. Call nlspec_list({element_type: "ENDPOINT"}) — wire to core functions
     f. Call nlspec_list({element_type: "CONFIG"}) — implement config loading
     g. Call nlspec_list({element_type: "SCENARIO"}) — write tests for ALL.
        If no scenarios: WARN human "No scenarios. Cannot validate correctness."
     h. Run all tests. Fix until ALL pass or MAX_RETRIES reached.
     i. Call nlspec_list({element_type: "ARTIFACT"}) — verify each artifact exists
        and passes its checkable command.

EXIT CRITERIA (all must be true for success):
  ✓ Code compiles / passes type checking
  ✓ ALL scenarios pass
  ✓ ALL declared ARTIFACTs exist and pass their checkable commands
  ✓ No CRITICAL gaps remain unaddressed
  → If all true: EXIT SUCCESS

RETRY LOGIC:
  MAX_RETRIES: 3 per failing scenario (not 3 total — 3 per distinct failure)
  On each retry: classify failure, apply appropriate strategy, re-run affected scenarios.
  After MAX_RETRIES for a single failure: ESCALATE.

FAILURE CLASSIFICATION (autonomous — no human needed):
  SPEC_GAP:  spec is missing info → file Category A patch, proceed best-effort
  IMPL_BUG:  code is wrong → fix the code, normal retry
  ENV_ISSUE: build environment broken → report and stop, do NOT retry
  AMBIGUITY: spec is unclear → try alternative interpretation on retry

ESCALATION: produce structured report to build/escalation-report.json (see CLAUDE.md
for full format). In dark factory mode: exit non-zero. In interactive: present to human.

TYPE-SPECIFIC BEHAVIOR:
  - SYSTEM: Implement as a deployable service/library/application.
  - PATTERN: Implement as a reusable module/library. No standalone deployment.
  - ASSET: Package and validate assets. Verify constraints are machine-readable.

DEPENDENCY BOUNDARY RULES:
  - Read Architecture.3 for USES PATTERN and USES ASSET declarations. For prior art
    patterns (no FROM), implement using your training knowledge. For novel patterns
    (FROM spec.md), read the referenced PATTERN spec via nlspec_get.
  - When you encounter an IMPORT, call nlspec_get on the referenced section only.
    Implement against the interface contract — do NOT hallucinate implementations.
  - If the contract is insufficient, STOP and report what's missing.
  - Call nlspec_seed_resolve before implementing to generate the seed manifest.
    The resolved constraints are non-negotiable.
  - Call nlspec_seed_audit for any constraint you're unsure about.
  - Treat imported definitions as READ-ONLY.
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
       nlspec_slice({section: "Functions.3"})
     - The slice contains ONLY the elements you need.
  2. If a PATCH exists, call nlspec_patch_list to find it
  3. Classify the failure (SPEC_GAP, IMPL_BUG, ENV_ISSUE, or AMBIGUITY)
  4. Act on classification:
     - SPEC_GAP: call nlspec_patch_create (Category A), proceed best-effort
     - IMPL_BUG: fix the code (normal path)
     - ENV_ISSUE: report and stop
     - AMBIGUITY: try alternative interpretation
  5. Fix ONLY the affected code
  6. Run SMOKE scenarios + AFFECTED scenarios (by [SEC:SectionName.x] tags)
  7. If all pass, you're done
  8. If SMOKE fails, something fundamental broke — stop and report

EXIT CRITERIA:
  ✓ The originally failing scenario now passes
  ✓ All SMOKE scenarios still pass
  ✓ All AFFECTED scenarios still pass
  ✓ No dependency contract violations (call nlspec_seed_resolve to verify)

RETRY LOGIC:
  MAX_RETRIES: 3 per failing scenario. Different approach each time.
  After MAX_RETRIES: ESCALATE with structured report.

DEPENDENCY CONSTRAINT RULES:
  - Before finalizing, call nlspec_seed_resolve to verify no contract violations.
  - If the fix would violate a dependency constraint, STOP and report.
  - Call nlspec_seed_audit to trace which spec owns the constraint.
```

### MODE: VALIDATE

```
Trigger:  "validate", "test", "verify", "check", "nightly"
Input:    Nothing new
Output:   Structured validation report
Process:
  1. Call nlspec_validate to check spec structural integrity and get gap report
  2. Run ALL scenario tests (full suite)
  3. Call nlspec_list({element_type: "ARTIFACT"}) — verify each artifact exists
     and passes its checkable command
  4. Call nlspec_seed_resolve to verify all dependency contracts are satisfied
  5. Run drift detection:
     a. Compare code's public API surface against spec's API declarations
     b. Compare code's error types against spec's Errors section
     c. Compare code's config schema against spec's Config section
  6. Produce structured VALIDATION REPORT:
     {
       spec_id: "...",
       mode: "VALIDATE",
       scenarios: {total: N, passed: N, failed: N},
       artifacts: {declared: N, verified: N, missing: [...]},
       contracts: {honored: N, violated: N},
       drift: {clean: true|false, drifted_elements: [...]},
       overall: "PASS" | "FAIL" | "WARN"
     }
  7. Write report to build/validation-report.json
  8. Do NOT fix anything — just report

EXIT CRITERIA:
  ✓ Report is produced (VALIDATE always succeeds as a mode)
  → overall: "PASS" if zero failures, zero artifact issues, zero drift
  → overall: "WARN" if only warnings
  → overall: "FAIL" if any scenario fails, artifact missing, or drift detected
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
  8. Call nlspec_list({element_type: "ARTIFACT"}) — verify all artifacts
  9. If spec has IMPORTS, call nlspec_seed_resolve to regenerate the seed manifest.
     Verify consolidated code honors all updated seed rules.
  10. Run drift detection to verify code and spec are in sync
  11. Clean up: remove dead code, align naming, update comments

EXIT CRITERIA:
  ✓ ALL scenarios pass
  ✓ ALL artifacts verified
  ✓ Seed manifest is current (if applicable)
  ✓ No dependency contract violations
  ✓ No drift between spec and code
  → If all true: EXIT SUCCESS with version bump

RETRY / ESCALATION: Same as IMPLEMENT — classify failures, retry up to 3 times
per distinct failure, escalate with structured report if exhausted.
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
  If you have a section name:
    nlspec_slice({namespace: "myproject", spec_id: "auth", section: "Functions.3"})

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
  "Fix requires changes across Functions.1 and Functions.3. Expanded context."

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

AFFECTED scenarios [SEC:SectionName.x]:
  - Run when the corresponding section is modified
  - Determined by [SEC:] tags in scenario headers
  - In FIX mode, run all scenarios matching the changed section(s)
  - Find them: nlspec_search({tags: ["SEC:Functions.3"], element_type: "SCENARIO"})

FULL suite:
  - Run in VALIDATE and CONSOLIDATE modes
  - Run nightly if automation is set up
  - Includes performance scenarios — these may be slow
  - Find them: nlspec_list({element_type: "SCENARIO"})
```

---

## Drift Detection — Spec vs Code Integrity

After IMPLEMENT, code and spec are in sync. Over time, they can drift. Drift detection
catches this during VALIDATE and CONSOLIDATE modes.

**What drift detection checks:**

```
1. API SURFACE DRIFT — ENDPOINTs in spec vs routes in code
2. DATA MODEL DRIFT — RECORDs in spec vs struct/class definitions in code
3. ERROR TYPE DRIFT — Errors section vs code's error types
4. CONFIG DRIFT — Config section vs code's config structs
5. ARTIFACT DRIFT — ARTIFACT declarations vs actual build outputs
```

**How to perform drift detection (Phase 1+):**

Use MCP tools to get the spec's declarations, then compare against code:
1. Call nlspec_list({element_type: "ENDPOINT"}) — get all declared endpoints
2. Call nlspec_list({element_type: "RECORD"}) — get all declared records
3. Read corresponding code files and compare
4. Report mismatches in the validation report's `drift` field

Drift is not always a bug — sometimes the code evolved correctly but the spec
didn't keep up. The agent reports drift but does not decide whose version is
correct. That's a human decision (or a SPEC mode task).

---

## Autonomous Execution — Dark Factory Mode

When instructed to run autonomously (e.g., "implement this end to end", "dark factory",
"run the full pipeline"), the agent chains modes together without human intervention.

### Autonomous Pipeline Steps

```
STEP 1: SPEC VALIDATION
  Call nlspec_validate. Check gap report.
  CRITICAL gaps + no existing code → ESCALATE.
  Otherwise → proceed.

STEP 2: IMPLEMENT
  Run IMPLEMENT mode with full exit criteria and retry logic.
  SPEC_GAP → call nlspec_patch_create, continue.
  IMPL_BUG → retry up to 3 times.
  ENV_ISSUE → ESCALATE immediately.

STEP 3: VALIDATE
  Run VALIDATE mode. Produces structured validation report.
  Check: scenarios, artifacts, contracts, drift.

STEP 4: DECISION GATE
  If PASS → EXIT SUCCESS.
  If FAIL (IMPL_BUG) → enter FIX mode, then re-VALIDATE.
  If FAIL (SPEC_GAP) → file patch via nlspec_patch_create, EXIT with patches filed.
  If FAIL (ENV_ISSUE) → EXIT with environment report.
  If FIX exhausts retries → ESCALATE.

STEP 5: EXIT
  Write build/pipeline-result.json with full structured report.
```

### Guard Rails

```
RULE: The pipeline NEVER modifies the spec. It files patches (separate documents).
RULE: The pipeline NEVER enters an infinite loop. Retry counters are strict.
RULE: The pipeline NEVER retries ENV_ISSUE. Report and stop.
RULE: Maximum total FIX attempts across all scenarios: 10.
RULE: ARTIFACT declarations define "done." Tests passing without artifacts is incomplete.
RULE: All decisions are written to structured JSON reports for traceability.
```

---

## Code Style Rules

```
RULE: Match the language idioms specified in Abstract section of the spec.
RULE: Error handling must match the Errors section exactly — no swallowing errors.
RULE: Every public function must have a doc comment referencing the spec section.
      Example: /// Implements Functions section.3: query_execute
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
  nlspec_get({namespace: "myproject", spec_id: "auth", section: "Functions.1"})

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
❌ Do NOT add dependencies not listed in the Dependencies section
❌ Do NOT modify scenario tests to match your code (scenarios are immutable)
❌ Do NOT skip SMOKE tests (they catch cascading breakage)
❌ Do NOT "improve" code style when fixing a bug (separate concerns)
❌ Do NOT create new public API not defined in the API section
❌ Do NOT modify imported elements from other specs (patch the source spec)
❌ Do NOT assume — use nlspec_search and nlspec_graph when context is insufficient
❌ Do NOT trust training knowledge for dependency APIs — read the INSTALLED artifact
❌ Do NOT substitute a different library/version than what the spec declares
❌ Do NOT use icon/asset names from memory — verify against the installed asset pack
❌ Do NOT generate config for a dependency based on training data — verify the
   config schema matches the installed version
```
