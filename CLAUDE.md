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

Specs have a TYPE: SYSTEM (running code), PATTERN (reusable blueprint), or ASSET
(static resources). Check the spec header to determine the type. PATTERN and ASSET
specs are consumed by SYSTEM specs via "USES PATTERN:" and "USES ASSET:" in Architecture.3.

Specs may also declare a LAYER: 1-Specification, 2-Realization, 3-Configuration, or
4-UserProfile. When a Layer is declared, the spec participates in the 4-layer composition
model. Check the LayerContext section for derivation chain, horizontal composition,
constraint flow, and substitution boundaries. Composition is two-dimensional:
**vertical** (L1→L2→L3→L4 derivation) and **horizontal** (multiple specs at the same
layer composing together via COMPOSES WITH). A spec at Layer N+1 derives from a spec
at Layer N — it inherits all contracts and adds layer-specific concerns. Constraints
flow downward by default, upward when user preferences are non-negotiable, and
laterally between horizontally composed peers. Specs without a Layer declaration are
standalone and do not participate in layer composition.

---

## Spec Tolerance

Specs come in many shapes. Your job is to work with what's there, not demand perfection.

**Well-structured specs** have a SECTIONS declaration block, typed elements (RECORD,
FUNCTION, SCENARIO), cross-references (USES, THROWS, [SEC:] tags), and a declared TYPE.
You can navigate them surgically and detect gaps precisely.

**Loose specs** have `## ` section headers but no SECTIONS block. Maybe they use
plain prose instead of formal RECORD/FUNCTION blocks. Maybe sections have unusual names.
You discover structure by reading the headers and scanning for element patterns.
Everything still works — you just have less to cross-check against.

**Minimal specs** might be a single markdown file with a description, some function
signatures, and a few behavioral notes. You do your best: extract what structure
exists, infer what's missing, and report gaps.

**Rules for tolerance:**
- Never refuse to implement a spec because it's missing structure.
- Never insist a user add a SECTIONS block, formal elements, or specific sections.
- When structure is missing, INFER what you can and REPORT what you can't.
- Gap detection is always advisory. The human decides whether to fix gaps.
- The NLSPEC-TEMPLATE is a recommendation, not a requirement. Specs that diverge
  from it are first-class citizens.

**Gap detection (you do this manually by reading the spec):**
- CRITICAL gaps: missing definitions referenced by other elements, broken cross-refs
- WARNING gaps: FUNCTIONs without USES/THROWS, SCENARIOs without [SEC:] tags
- INFO gaps: missing common sections for the TYPE, large unstructured prose blocks
- If there's no SECTIONS block, "missing declared sections" gaps don't apply

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
│   ├── scenarios/             # Scenario tests (derived from spec Scenarios section)
│   └── smoke/                 # Smoke tests tagged [SMOKE]
└── config/                    # Configuration files
```

---

## Operating Modes

You operate in one of seven modes. The human tells you which mode, or you infer it
from the instruction. If ambiguous, ask.

**Pipeline: DESIGN → SPEC → IMPLEMENT → VALIDATE**

### MODE: SPEC

```
Trigger:  "spec", "refine", "review spec", "write spec", "help me spec", "add scenarios"
Input:    A spec file (new or existing) and human guidance on what to improve
Output:   An improved spec file
Process:
  1. Read the spec (or start from NLSPEC-TEMPLATE.md if new)
  2. Check whether the spec has a SECTIONS block at the top:
     a. If yes: use it as the contract. Check declared sections against what's
        actually in the markdown. Flag missing or extra sections.
     b. If no: scan for ## headers to discover sections. No completeness check
        against a declaration, but you can still check internal consistency.
  3. Scan the spec for gaps (you do this by reading, not with a tool):
     a. Are FUNCTIONs missing USES and THROWS declarations?
     b. Are RECORDs missing USED BY references?
     c. Do SCENARIOs have [SEC:] tags?
     d. Are there sections with no SCENARIOs covering them (untested)?
     e. Are there RECORDs with no USED BY (orphaned)?
     f. Do all cross-references resolve (no dangling USES/THROWS)?
     g. (If SECTIONS block exists) Are all declared sections present?
     h. (If SECTIONS block exists) Are there sections not in the declaration?
  4. Report gaps to the human with severity levels:
     - CRITICAL: will cause implementation failures (missing definitions, broken refs)
     - WARNING: reduced effectiveness (missing tags, missing declarations)
     - INFO: suggestions for improvement (missing common sections, large prose blocks)
  5. If the human asked for specific help (e.g., "add scenarios for Functions section"),
     focus there. Otherwise, report findings and suggest improvements.
  6. Propose changes — do NOT modify the spec without human approval
  7. When approved, make the changes to the spec file
Validation: Spec is internally consistent. Human approves all changes.

IMPORTANT:
  - In SPEC mode you work on the SPEC, not the code.
  - You may create, modify, or reorganize spec content.
  - You do NOT write implementation code in this mode.
  - You do NOT modify scenarios without human approval (scenarios are the contract).
  - When adding FUNCTIONs, always add USES/THROWS declarations.
  - When adding RECORDs, always add USED BY references.
  - When adding SCENARIOs, always add [SEC:] tags and a tier ([SMOKE], [AFFECTED], [FULL]).
  - For loose specs, SUGGEST adding a SECTIONS block but don't require it.
  - Never refuse to work with a spec because it lacks a SECTIONS declaration.

DEPENDENCY CONTRACT RULES (for specs with IMPORTS or EXPORTS):
  - When modifying Contracts section EXPORTS, check which specs import from this one.
    Changing an override=NEVER export can break every downstream consumer.
    Report the blast radius before proposing the change.
  - When reviewing for completeness, also check Contracts section:
    i. Does every IMPORT have a corresponding EXPECTS declaration?
    j. Are all EXPORTS grounded in real sections (source_ref)?
    k. Are override levels appropriate?
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
  1. Read the problem statement (Problem section or human's description)
  2. Propose an architecture:
     a. Draw the ASCII diagram (Architecture section)
     b. Write the component inventory (Architecture section.1)
     c. Map the primary data flows (Architecture section.2)
  3. Select patterns:
     a. Identify which prior art patterns apply (from pattern-catalog.md
        or training knowledge). Write USES PATTERN declarations.
     b. Identify if any novel patterns are needed. If so, note them —
        they'll need their own TYPE: PATTERN specs.
  4. Identify assets needed:
     a. Does the system need a design system, branding, config templates,
        certificates? Check if these files already exist in the workspace.
     b. If asset files exist but no ASSET spec catalogs them, note that an
        ASSET spec needs to be created with:
        - FileStructure section: asset catalog (paths, formats, dimensions, variants)
        - Functions section: style guide (RULEs and TOKENs the agent must follow)
     c. Write USES ASSET declarations for existing or planned ASSET specs.
  5. Map dependencies:
     a. What other specs does this system import from? (Appendix B)
     b. What will this spec export to consumers? (Contracts section.1)
     c. What does this spec expect from dependencies? (Contracts section.2)
  6. Define boundaries (Boundaries section):
     a. What this system does and does NOT do
     b. What's deferred to future specs
  7. Write the abstract (Abstract section) — now that the architecture is clear
  8. Present the partial spec for human review

OUTPUT FORMAT:
  - Abstract, Problem, Architecture (with 3.1, 3.2, 3.3), 15, 16: fully written
  - All other sections: template headers only with a one-line note on what
    goes there (e.g., "DataModel: Will define the CatalogEntry, QueryResult,
    and ConstraintMask records")
  - A "Design Decisions" appendix listing key choices made and alternatives
    considered

IMPORTANT:
  - DESIGN mode produces ARCHITECTURE, not code.
  - The output is a starting point for SPEC mode, which fills in the details.
  - Always explain WHY a pattern was chosen, not just which one.
  - If the problem is too vague, ask clarifying questions before designing.
  - For multi-spec systems, also produce a dependency diagram showing which
    specs import from which, and the build order.
```

### MODE: DESCRIBE

```
Trigger:  "describe", "explain", "walk me through", "what does", "how does",
          "summarize", "overview", "tell me about", "onboard me",
          "what depends on", "what does this export", "trace the dependencies"
Input:    A spec file (or specific section/element)
Output:   Human-readable explanation. No modifications to anything.
Process:
  1. Read the requested spec, section, or element
  2. Check the spec TYPE (SYSTEM, PATTERN, or ASSET) and adapt explanation:
     - SYSTEM: explain what the system does and how it works
     - PATTERN: explain when to use it, what it constrains, how consuming specs apply it
     - ASSET: explain what resources are provided and what design constraints apply
  3. Explain in plain language:
     - What does this system/pattern/asset do?
     - How do the parts relate to each other?
     - What are the key data flows?
  4. Adapt depth to the question:
     - "What does this system do?" → high-level summary (abstract + architecture)
     - "Explain Functions section.3" → detailed walkthrough of that function
     - "What patterns does this use?" → list Architecture.3, explain each pattern's role
     - "What scenarios cover storage?" → list scenarios tagged with storage sections
  5. For dependency chain questions:
     - "What does this spec depend on?" → read IMPORTS, list dependencies with what
       is imported
     - "What does this spec export?" → read Contracts section, explain each EXPORT
     - "What would this system boot with?" → trace EXPORTS from all dependencies
       and describe the resolved seed state
     - "Where does this constraint come from?" → trace provenance through the graph
     - "What breaks if I change this export?" → follow IMPORTS in reverse

IMPORTANT:
  - In DESCRIBE mode you are READ-ONLY. Do not modify any files.
  - Do not suggest improvements unless explicitly asked.
  - When describing dependencies, read the actual imported specs. Do NOT guess.
```

### MODE: IMPLEMENT

```
Trigger:  "implement", "build", "create", "start"
Input:    Full spec (specs/{project}-spec.md)
Output:   Complete codebase matching the spec
Process:
  1. Read the FULL spec
  2. Scan for gaps (same checks as SPEC mode, but faster — you're looking for blockers):
     - If CRITICAL gaps exist (missing definitions, broken refs), report them to the
       human and ask whether to proceed or fix the spec first. Do NOT refuse — the
       human decides.
     - If only WARNING/INFO gaps, note them and proceed.
  3. Install dependencies (Dependencies section):
     a. Read all DEPENDENCY declarations
     b. Install each using the declared `install` command and `version`
     c. Run the `verify` check for each. If no verify field, use the default
        checks from the template (read types/exports, check version, inspect image)
     d. If verify fails: is the declared version actually available? If not, this is
        a SPEC_GAP (version doesn't exist). If yes, retry the install.
     e. CRITICAL: After install, DO NOT trust your training knowledge about this
        dependency's API. Read the ACTUAL installed artifact:
        - For libraries: read the installed type definitions or module exports
        - For CLIs: run --help and read the actual subcommands/flags
        - For base images: docker inspect for actual platform and packages
        - For asset packs: list the actual available assets/icons/components
        You will use this ACTUAL information — not your training data — when
        writing code that interacts with this dependency.
  4. Adapt your implementation plan to what the spec actually provides:
     a. FileStructure section present → create the directory layout from it
     b. No FileStructure → infer layout from the language target and conventions
     c. DataModel section present → implement all RECORDs, ENUMs, type aliases
     d. No DataModel → extract data structures from FUNCTION signatures
     e. Errors section present → implement all error types from it
     f. No Errors → implement errors from THROWS declarations in FUNCTIONs
     g. Functions section present → implement in dependency order
     h. API section present → wire endpoints to core functions
     i. Config section present → implement config loading
     j. Scenarios section present → write tests for ALL scenarios
     k. No Scenarios → WARN the human: "No scenarios. Cannot validate correctness."
     l. BuildAndRun section present → verify build commands work
  5. Run all tests. Fix until ALL pass or MAX_RETRIES reached.
  6. If ARTIFACT declarations exist in BuildAndRun, verify each artifact:
     a. Run the `checkable` command for each ARTIFACT
     b. Verify all declared properties match
     c. Report any missing or invalid artifacts

EXIT CRITERIA (all must be true for success):
  ✓ Code compiles / passes type checking
  ✓ ALL scenarios pass
  ✓ ALL declared ARTIFACTs exist and pass their checkable commands
  ✓ No CRITICAL gaps remain unaddressed
  → If all true: EXIT SUCCESS

RETRY LOGIC:
  MAX_RETRIES: 3 per failing scenario (not 3 total — 3 per distinct failure)
  On each retry:
    1. Classify the failure (see FAILURE CLASSIFICATION below)
    2. Apply the appropriate strategy
    3. Run affected scenarios again
  After MAX_RETRIES for a single failure: ESCALATE (see below)

FAILURE CLASSIFICATION (autonomous — no human needed):
  The agent classifies each failure into one of three categories and acts accordingly:

  SPEC_GAP — The spec is missing information needed to implement correctly.
    Signals: FUNCTION references an undefined RECORD, SCENARIO expects behavior
             not described in any FUNCTION, EXPECTS declaration has no corresponding
             IMPORT, a section is referenced in [SEC:] tags but doesn't exist
    Action:  File a Category A patch (SPEC_DEFICIENCY). Log the gap. Proceed with
             best-effort implementation of remaining sections. Do NOT block on this.

  IMPL_BUG — The code is wrong. The spec is clear but the implementation doesn't match.
    Signals: SCENARIO expected output differs from actual, type error, logic error,
             test assertion failure where the spec unambiguously defines the behavior
    Action:  Fix the code. This is a normal retry. Re-read the relevant spec section
             and try a different implementation approach on each retry.

  ENV_ISSUE — The build environment is broken. Not a spec or code problem.
    Signals: dependency install fails, compiler not found, port already in use,
             permission denied, network timeout, disk full
    Action:  Report the environment issue. Do NOT retry — retrying won't fix it.
             Log: "ENV_ISSUE: {description}. Cannot proceed until resolved."

  AMBIGUITY — The spec can be read two ways and the chosen interpretation failed.
    Signals: SCENARIO fails but the spec text is genuinely unclear about the expected
             behavior, multiple valid interpretations exist
    Action:  If autonomous (dark factory): try the alternative interpretation on retry.
             If interactive: ask the human which interpretation is correct.

ESCALATION (when retries are exhausted):
  Produce a structured ESCALATION REPORT:
    {
      mode: "IMPLEMENT",
      spec_id: "{project}",
      status: "ESCALATED",
      completed: ["DataModel", "Functions.1", "Functions.2", "API"],
      blocked: ["Functions.3"],
      failures: [
        {scenario: 7, category: "IMPL_BUG", attempts: 3, last_error: "..."},
        {scenario: 12, category: "SPEC_GAP", patch_filed: "PATCH-001", description: "..."}
      ],
      artifacts: {declared: 2, verified: 1, failed: ["docker-image"]},
      suggestion: "Functions.3 scenario 7 may need spec clarification — see PATCH-001"
    }
  In dark factory mode: write this to build/escalation-report.json and exit non-zero.
  In interactive mode: present to the human and wait.

TYPE-SPECIFIC BEHAVIOR:
  - SYSTEM: Implement as a deployable service/library/application.
  - PATTERN: Implement as a reusable module/library. No standalone deployment.
    Validate via Scenarios section scenarios (constraint validation, not end-to-end).
  - ASSET: Package and validate assets. Verify constraints are machine-readable.

DEPENDENCY BOUNDARY RULES (critical for multi-spec projects):
  - Read Architecture.3 for USES PATTERN and USES ASSET declarations. For prior art
    patterns (no FROM), implement using your training knowledge with the specified
    customization. For novel patterns (FROM spec.md), read the referenced spec.
  - When you encounter an IMPORT, read ONLY the referenced section. Implement
    against the interface contract — not an invented implementation.
  - Do NOT hallucinate implementations for imported components. If the contract
    is insufficient, STOP and report what's missing.
  - If a seed manifest exists (build/seed-manifest.json), read it before
    implementing. The resolved constraints are non-negotiable.
  - Treat imported definitions as READ-ONLY.

LAYER COMPOSITION RULES (when spec declares a Layer):
  - Read the LayerContext section FIRST. It tells you: what this spec derives from
    and composes with (LayerContext.1), the full stack (LayerContext.2), and constraint
    flow (LayerContext.3).
  - If DERIVES FROM declares a parent spec, read the parent's Contracts section EXPORTS.
    Every inherited EXPORT is a constraint you MUST satisfy — treat inherited EXPORTS
    marked override=NEVER as immutable invariants.
  - If COMPOSES WITH declares horizontal peers, read each peer's EXPORTS. Verify
    that every `depends_on` interface is satisfied by a peer's EXPORT. Treat
    co-required peers as mandatory — the layer is incomplete without them.
  - Follow cross-layer references [Ln:spec-id:Section.x] to trace constraints back
    to their source layer. When validating, ensure the implementation honors constraints
    from every layer in the derivation chain.
  - For UPWARD constraint flow (LayerContext.3): if the spec declares upward exports,
    validate them against the parent layer. If the parent constraint is override=NEVER,
    report a conflict — do not silently violate it.
  - For LATERAL constraint flow: validate interface contracts between composing peers.
    Both sides of a lateral interface must agree on the exported contract.
  - Substitution boundary (LayerContext.4): your implementation must be swappable at
    this layer without breaking specs above, below, or beside (lateral peers).
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
     c. Read ONLY that spec section (e.g., Functions section.3)
     d. Note which RECORDs the function references — read ONLY those from DataModel
     e. Note which errors it can throw — read ONLY those from Errors
     f. Find scenarios tagged [SEC:Functions.3] — read ONLY those from Scenarios section
     g. STOP READING. You now have your context slice.
  2. If a PATCH spec was provided, read that too
  3. Classify the failure (SPEC_GAP, IMPL_BUG, ENV_ISSUE, or AMBIGUITY)
  4. Act on classification:
     - SPEC_GAP: file a Category A patch, implement best-effort, note the gap
     - IMPL_BUG: fix the code (normal path)
     - ENV_ISSUE: report and stop — do not retry
     - AMBIGUITY: try alternative interpretation, note the uncertainty
  5. Fix ONLY the affected code
  6. Do NOT refactor unrelated code
  7. Do NOT modify files unrelated to the fix
  8. Run SMOKE scenarios + AFFECTED scenarios (by [SEC:SectionName.x] tags)
  9. If all pass, you're done
  10. If SMOKE fails, something fundamental broke — stop and report

EXIT CRITERIA:
  ✓ The originally failing scenario now passes
  ✓ All SMOKE scenarios still pass
  ✓ All AFFECTED scenarios (by [SEC:] tag) still pass
  ✓ No dependency contract violations introduced
  → If all true: EXIT SUCCESS

RETRY LOGIC:
  MAX_RETRIES: 3 for the specific failing scenario
  On each retry: try a different approach. Do NOT repeat the same fix.
  After MAX_RETRIES: ESCALATE with structured report (same format as IMPLEMENT).

DEPENDENCY CONSTRAINT RULES:
  - Before finalizing a fix, verify it does not violate any dependency contract
    or pattern constraint from Architecture.3.
  - If the fix would violate a constraint from a dependency, STOP and report.
  - Do NOT work around dependency constraints — if the constraint is wrong,
    the dependency spec needs to change.
```

### MODE: VALIDATE

```
Trigger:  "validate", "test", "verify", "check", "nightly"
Input:    Nothing new (or full spec for full validation)
Output:   Structured validation report + dependency contract status
Process:
  1. Run ALL scenario tests (full suite)
  2. If ARTIFACT declarations exist, verify each artifact:
     a. Does the artifact exist at the declared path?
     b. Does the checkable command succeed?
     c. Do declared properties match?
  3. If spec has IMPORTS or USES PATTERN/ASSET, verify the implementation
     honors all dependency contracts, pattern constraints, and asset constraints
  4. If a seed manifest exists, verify it is current (not stale)
  5. Run drift detection (see Drift Detection below):
     a. Compare code's public API surface against spec's API declarations
     b. Compare code's error types against spec's Errors section
     c. Compare code's config schema against spec's Config section
  6. Produce a structured VALIDATION REPORT:
     {
       spec_id: "{project}",
       mode: "VALIDATE",
       timestamp: "...",
       scenarios: {total: N, passed: N, failed: N, skipped: N},
       artifacts: {declared: N, verified: N, missing: [...], invalid: [...]},
       contracts: {honored: N, violated: N, violations: [...]},
       drift: {clean: true|false, drifted_elements: [...]},
       seed_manifest: "current" | "stale" | "not_applicable",
       overall: "PASS" | "FAIL" | "WARN"
     }
  7. Write report to build/validation-report.json
  8. Do NOT fix anything — just report

EXIT CRITERIA:
  ✓ Report is produced (VALIDATE always succeeds as a mode — it reports, not fixes)
  → overall: "PASS" if zero failures, zero artifact issues, zero drift, zero violations
  → overall: "WARN" if only warnings (orphaned records, missing performance targets)
  → overall: "FAIL" if any scenario fails, artifact missing, contract violated, or drift detected
Validation: Report only. Human decides next action (or dark factory chains to FIX).
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
  6. Verify all ARTIFACT declarations still hold
  7. If spec has IMPORTS, re-check dependency contracts:
     a. Have any dependency EXPORTS or PATTERN constraints changed?
     b. Do absorbed patches introduce new dependencies?
     c. Verify consolidated code honors all updated seed rules
  8. Clean up: remove dead code, align naming, update comments

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

## Drift Detection — Spec vs Code Integrity

After IMPLEMENT, code and spec are in sync. Over time, they can drift: someone edits
code directly, a dependency changes, or patches accumulate without consolidation.
Drift detection catches this.

**What drift detection checks:**

```
1. API SURFACE DRIFT
   Compare the spec's API section (ENDPOINTs) against the code's actual routes/handlers.
   - Missing endpoints (spec declares, code doesn't implement)
   - Extra endpoints (code exposes, spec doesn't declare)
   - Signature mismatches (different parameters, return types)

2. DATA MODEL DRIFT
   Compare the spec's DataModel RECORDs against the code's struct/class definitions.
   - Missing fields, extra fields, type mismatches
   - Missing RECORDs, extra types not in spec

3. ERROR TYPE DRIFT
   Compare the spec's Errors section against the code's error types.
   - Missing error variants, extra variants, wrong error codes

4. CONFIG DRIFT
   Compare the spec's Config section against the code's config structs/loading.
   - Missing config keys, extra keys, wrong defaults, wrong types

5. ARTIFACT DRIFT
   Compare the spec's ARTIFACT declarations against actual build outputs.
   - Missing artifacts, wrong paths, failed checkable commands
```

**How to perform drift detection (Phase 0):**

You don't have a dedicated tool. Instead, in VALIDATE mode:
1. Read the spec's API, DataModel, Errors, Config, and BuildAndRun sections
2. Read the corresponding code files
3. Compare manually: grep for endpoint registrations, struct definitions, error
   enums, config structs
4. Report any mismatches as drift in the validation report

**When to run drift detection:**
- Always during VALIDATE mode
- Always during CONSOLIDATE mode (before refactoring)
- On demand: "check for drift", "is the code still in sync?"

**Drift is not always a bug.** Sometimes it signals a spec that needs updating
(the code evolved correctly but the spec didn't keep up). The agent reports drift
but does not decide whose version is correct — that's a human decision.

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
  FUNCTION (Functions section.x) or ENDPOINT (API.x) is broken.
  If given a scenario number, read that scenario's [SEC:] tags.

STEP 3: Read the affected section.
  Command: Read Functions section.x (or 6.x) from the spec.
  This tells you the function's inputs, outputs, behavior, errors.
  Cost: ~50-150 lines.

STEP 4: Read referenced RECORDs.
  The function definition references RECORDs by name (e.g., "takes a QueryRequest,
  returns a QueryResponse"). Read ONLY those RECORD definitions from DataModel.
  Do NOT read all of DataModel.
  Cost: ~20-50 lines per RECORD.

STEP 5: Read referenced errors.
  The function lists THROWS errors. Read ONLY those error definitions from Errors.
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
  [SEC:Functions.3] [SEC:DataModel.1] [SMOKE]     -- tells you which sections this validates

In RECORD definitions:
  USED BY: function_a, function_b  -- tells you which functions reference this
```

### When context is insufficient:

```
RULE: If you need a RECORD definition you don't have, read it from the spec.
  Do NOT ask the human. Navigate the spec file yourself.

RULE: If you discover the bug spans multiple sections (e.g., Functions.3 AND Functions.1),
  read the additional section. But REPORT this to the human:
  "Fix requires changes across Functions.1 and Functions.3. Expanded context."

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

AFFECTED scenarios [SEC:SectionName.x]:
  - Run when the corresponding section is modified
  - Determined by [SEC:] tags in scenario headers
  - In FIX mode, run all scenarios matching the changed section(s)

FULL suite:
  - Run in VALIDATE and CONSOLIDATE modes
  - Run nightly if automation is set up
  - Includes performance scenarios — these may be slow
```

---

## Autonomous Execution — Dark Factory Mode

When instructed to run autonomously (e.g., "implement this end to end", "dark factory",
"run the full pipeline"), the agent chains modes together without human intervention.
The human's job is writing the spec and reviewing the final output. Everything between
is autonomous.

### The Execution DAG

```
                    ┌─────────┐
                    │  START   │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ VALIDATE │ ← scan spec for gaps + check drift
                    │  (spec)  │
                    └────┬─────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
         CRITICAL    WARNING     CLEAN
         gaps found  gaps found   │
              │          │         │
         ┌────▼────┐    │    ┌────▼──────┐
         │ ESCALATE │    │    │ IMPLEMENT │
         │ (report) │    │    └────┬──────┘
         └─────────┘     │         │
                         │    ┌────▼──────┐
                    ┌────▼──┐ │ VALIDATE  │ ← test + artifact + drift
                    │IMPLEMENT│ │  (code)  │
                    └────┬──┘ └────┬──────┘
                         │         │
                    ┌────▼──────┐  │
                    │ VALIDATE  │  │
                    │  (code)   │  ├─── PASS → EXIT SUCCESS
                    └────┬──────┘  │
                         │         ├─── FAIL (IMPL_BUG) → FIX (retry loop)
                         │         │
                    same as above  ├─── FAIL (SPEC_GAP) → file patch → ESCALATE
                                   │
                                   └─── FAIL (ENV_ISSUE) → ESCALATE immediately
```

### Autonomous Pipeline Steps

```
STEP 1: SPEC VALIDATION
  Read the spec and scan for gaps (same checks as SPEC mode step 3).
  If CRITICAL gaps AND no code exists yet:
    → ESCALATE. The spec isn't ready. Report what's missing.
  If CRITICAL gaps AND code already exists:
    → Note gaps. Proceed — the existing code may already handle some of it.
  If WARNING/INFO gaps only:
    → Note gaps. Proceed.

STEP 2: IMPLEMENT
  Run IMPLEMENT mode with full exit criteria.
  The retry loop handles IMPL_BUG failures automatically (up to 3 per scenario).
  SPEC_GAP failures produce patches. ENV_ISSUE failures escalate immediately.

STEP 3: VALIDATE
  Run VALIDATE mode. Produces the validation report.
  Check: scenarios, artifacts, contracts, drift.

STEP 4: DECISION GATE
  Read the validation report.
  If overall = "PASS":
    → EXIT SUCCESS. Write build/pipeline-result.json.
  If overall = "WARN":
    → EXIT SUCCESS with warnings. Write report.
  If overall = "FAIL":
    → For each failure, classify and act:
      - IMPL_BUG: enter FIX mode for that scenario. After fix, re-run VALIDATE.
      - SPEC_GAP: file a patch. If this is the only blocker, EXIT with patches filed.
      - ENV_ISSUE: EXIT with environment report.
    → If FIX succeeds, loop back to STEP 3 (VALIDATE again).
    → If FIX exhausts retries, ESCALATE.

STEP 5: EXIT
  Produce build/pipeline-result.json:
    {
      spec_id: "...",
      pipeline: "autonomous",
      result: "SUCCESS" | "PARTIAL" | "ESCALATED",
      validation_report: "build/validation-report.json",
      escalation_report: "build/escalation-report.json" | null,
      patches_filed: ["PATCH-001", ...],
      duration_seconds: N,
      scenarios: {total: N, passed: N, failed: N},
      artifacts: {declared: N, verified: N}
    }
```

### Guard Rails for Autonomous Execution

```
RULE: The autonomous pipeline NEVER modifies the spec. It can FILE patches
  (which are separate documents), but the spec itself is immutable during execution.

RULE: The pipeline NEVER enters an infinite loop. Every FIX attempt decrements
  a retry counter. When exhausted, it ESCALATES instead of retrying.

RULE: The pipeline NEVER proceeds past ENV_ISSUE. Environment problems are outside
  the agent's control. Retrying will not help. Report and stop.

RULE: The pipeline writes structured JSON reports. Every decision is traceable.
  A human reviewing the output can see exactly what happened, what was tried,
  and why the pipeline escalated (if it did).

RULE: SPEC_GAP patches are filed but NOT applied. The human reviews them.
  The pipeline continues with best-effort implementation of the remaining spec.

RULE: Maximum total FIX attempts across all scenarios: 10. This prevents
  degenerate cases where many scenarios fail and the pipeline spends hours
  retrying. After 10 total fix attempts, ESCALATE the whole pipeline.

RULE: The pipeline respects the spec's ARTIFACT declarations as the definition
  of "done." Code that compiles and passes tests but doesn't produce the declared
  artifacts is NOT a successful implementation.
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

### Cross-Layer References

When a spec declares a Layer, it may contain cross-layer references:
```
[Ln:{spec-id}:{Section}.{subsection}]
```

When you encounter a cross-layer reference:
1. Identify the layer number (Ln) and spec-id
2. If the referenced spec exists, read the referenced section for context
3. If the referenced spec is marked `(planned)` in the LAYER STACK, note it as
   unresolvable — do not block on it
4. Cross-layer references are traceability markers. They tell you WHERE a constraint
   originates. Follow them during validation to ensure your implementation honors
   constraints from every layer in the derivation chain.

---

## Anti-Patterns — Do NOT Do These

```
❌ Do NOT read the full spec when in FIX mode (wasteful, loses focus)
❌ Do NOT rewrite files unrelated to the current task
❌ Do NOT add dependencies not listed in the Dependencies section
❌ Do NOT modify scenario tests to match your code (scenarios are immutable)
❌ Do NOT skip SMOKE tests (they catch cascading breakage)
❌ Do NOT "improve" code style when fixing a bug (separate concerns)
❌ Do NOT create new public API not defined in the API section
❌ Do NOT assume — ask when the context slice is insufficient
❌ Do NOT trust training knowledge for dependency APIs — read the INSTALLED artifact
❌ Do NOT substitute a different library/version than what the spec declares
❌ Do NOT use icon/asset names from memory — verify against the installed asset pack
❌ Do NOT generate config for a dependency based on training data — verify the
   config schema matches the installed version (e.g., airflow 3.x config ≠ 2.x config)
```

---

## Transition to Phase 1

When the project transitions to MCP-based spec management:
1. Replace this file with `specs/CLAUDE-MCP.md`
2. The agent will use MCP tools instead of reading spec files directly
3. Context slicing becomes automatic (nlspec_slice) instead of manual
4. All operating modes remain the same — only the access method changes
