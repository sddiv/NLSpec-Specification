# CLAUDE-MCP.md — Agent Operating Instructions

> This file tells the coding agent how to work in this repository.
> It is read automatically by Claude Code, Cursor, and similar tools.
> Do NOT modify this file unless instructed by the human operator.

---

## Spec Access — Two Paths

This repository contains two representations of the same specs:

### Path A: Monolithic Specs (simple agents, quick start)

```
specs/bootstrap-spec_retired.md     ← Phase 1: parser, store, query, 8 tools
specs/mcp-server-spec_retired.md    ← Phase 2: extends Phase 1 with 13 more tools
```

Use these if you're a general-purpose coding agent working on a single task.
Each file is self-contained — read one file and implement from it.

### Path B: Layered Specs (advanced agents, gRPC consumers)

```
specs/mcp/
├── L1/L1-nlspec-server-spec.md          ← Contracts (all records, functions, 21 tools)
├── L2/L2-nlspec-workflows-spec.md       ← Workflows (data flows, scenarios, build order)
├── L3/L3-nlspec-config.md               ← Config (PostgreSQL, deployment, dependencies)
└── L4/L4-nlspec-seed-spec.md            ← Self-bootstrap seed data
```

Use these when:
- Building a system that connects to nlspec-server via gRPC
- Working on a specific layer (e.g., "fix the config" → read only L3)
- Running spec-artifact alignment (markers reference layer-qualified element IDs)
- You need to understand the derivation chain between layers

Both paths describe the same system. The layered version adds phase and layer
context that the monolithic version doesn't have.

---

## Choosing Your Path

```
IF you are:
  - A coding agent given "implement the nlspec server"
  - Working from Claude Desktop with just MCP tools
  - Doing a one-shot implementation
  → Use Path A (monolithic). Read bootstrap-spec_retired.md first, build Phase 1,
    then read mcp-server-spec_retired.md and extend.

IF you are:
  - Building a gRPC consumer of nlspec-server
  - Working on a specific layer (just L3 config, just L1 contracts)
  - Running spec-align for marker-based diffing
  - Building the gRPC interface (defined in layered specs only)
  → Use Path B (layered). Read the specific phase/layer you need.
```

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

### Bootstrap Tools (Phase 1)

```
nlspec_init         Initialize a new spec from template
nlspec_get          Read a specific element or section
nlspec_list         List specs, sections, or elements with filters
nlspec_search       Text and structural search across specs
nlspec_create       Add a new element to a spec
nlspec_update       Modify an existing element
nlspec_delete       Remove an element from a spec
nlspec_query        Structured query with filters
```

### Extended Tools (Phase 2)

```
nlspec_import       Import a spec file into the system under a namespace
nlspec_namespaces   List all namespaces and their specs
nlspec_slice        Extract minimal context for a bug fix
nlspec_validate     Check a spec's structural integrity and gap report
nlspec_graph        Get the dependency graph for a spec or element
nlspec_split        Analyze and decompose a monolith spec
nlspec_patch_create Create a tracked patch for a spec
nlspec_patch_list   List patches with filters
nlspec_patch_absorb Merge a patch back into the main spec
nlspec_drift        Compare spec declarations against code surface
nlspec_layer_stack  Get the full layer composition tree for a spec
nlspec_layer_validate Validate cross-layer constraint flow
nlspec_substrate_query  Query substrate topology
nlspec_substrate_graph  Get full substrate graph for a namespace
```

All tools accept an optional `namespace` parameter. When omitted, queries
search across all namespaces.

---

## Operating Modes

You operate in one of seven modes. The human tells you which mode, or you infer it.

**Pipeline: DESIGN → SPEC → IMPLEMENT → VALIDATE**

### MODE: SPEC
```
Trigger:  "spec", "refine", "review spec", "add scenarios"
Input:    A spec identified by namespace/spec_id
Output:   An improved spec
Process:
  1. Call nlspec_validate — check gap report
  2. Call nlspec_list to survey elements
  3. Review gaps: CRITICAL (report), WARNING (include), INFO (suggest)
  4. Propose changes — do NOT modify without human approval
  5. When approved, use nlspec_create/update
  6. Validate again to confirm
MARKERS: When creating new spec elements, note the expected marker ID
  for future implementations: "Implementers: mark with @nlspec {id}"
```

### MODE: IMPLEMENT
```
Trigger:  "implement", "build", "create"
Input:    A spec identified by namespace/spec_id
Output:   Complete codebase with markers
Process:
  1. Call nlspec_validate — check gaps, proceed or escalate
  2. Install dependencies from spec
  3. Implement: Records → Errors → Functions → Endpoints → Config
  4. EMBED MARKERS on every generated function/class/record
  5. Write tests for ALL scenarios
  6. Run all tests, fix until passing
  7. Verify all ARTIFACT declarations

EXIT CRITERIA:
  ✓ Code compiles / passes type checking
  ✓ ALL scenarios pass
  ✓ ALL artifacts exist and pass their checkable commands
  ✓ ALL generated code has @nlspec markers
  ✓ No CRITICAL gaps remain unaddressed

MARKER VERIFICATION:
  After implementation, run a marker audit:
  - Count spec elements (nlspec_list)
  - Count markers in generated code (grep @nlspec)
  - Every spec element should have a corresponding marker
  - Report unmarked elements as implementation gaps
```

### MODE: FIX
```
Trigger:  "fix", "bug", "broken", "failing"
Input:    A bug description or failing scenario number
Output:   Targeted code changes with updated markers
Process:
  1. Call nlspec_slice for minimal context
  2. Classify: SPEC_GAP, IMPL_BUG, ENV_ISSUE, AMBIGUITY
  3. Fix ONLY affected code
  4. UPDATE markers on changed functions (@v hash, @lines range)
  5. Run SMOKE + AFFECTED scenarios
```

### MODE: VALIDATE
```
Trigger:  "validate", "test", "verify"
Output:   Structured validation report
Process:
  1. nlspec_validate for spec integrity
  2. Run ALL scenario tests
  3. Verify artifacts
  4. Drift detection: compare code API surface against spec
  5. MARKER AUDIT: verify all spec elements have markers in code
  6. Produce report to build/validation-report.json
```

### MODE: DESCRIBE
```
Trigger:  "describe", "explain", "walk me through"
Output:   Human-readable explanation (read-only, no changes)
```

### MODE: DESIGN
```
Trigger:  "design", "architect", "propose"
Output:   Partial spec with Architecture, Abstract, Problem filled in
```

### MODE: CONSOLIDATE
```
Trigger:  "consolidate", "absorb patches", "version bump"
Process:  Absorb patches → refactor code → re-validate → update markers
```

---

## Context Slicing

In FIX mode, use nlspec_slice to get only what you need:

```
nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  → Returns: trigger, elements, trace, total_lines

Typical slice: 200-500 lines (10-30 elements).
The server does the graph traversal for you.
```

---

## Scenario Execution

```
SMOKE [SMOKE]:   Run on EVERY change. Must pass first. < 30 seconds.
AFFECTED [SEC:]: Run when corresponding section modified.
FULL:            Run in VALIDATE and CONSOLIDATE modes.
```

---

## Code Style

```
RULE: Match language idioms from spec's Abstract section
RULE: Error handling must match Errors section exactly
RULE: Every public function gets a doc comment + @nlspec marker
RULE: Every test file references the scenario number
RULE: No features not in the spec
RULE: Prefer clarity over cleverness
```

---

## Anti-Patterns

```
❌ Do NOT read spec files directly (use MCP tools)
❌ Do NOT read the full spec in FIX mode (use nlspec_slice)
❌ Do NOT generate code without @nlspec markers
❌ Do NOT modify scenarios to match your code (scenarios are immutable)
❌ Do NOT skip SMOKE tests
❌ Do NOT add features not in the spec
❌ Do NOT modify imported elements from other specs (patch the source)
❌ Do NOT trust training knowledge for dependency APIs (read installed artifacts)
```

---

## Working with Layered Specs (Path B)

When using the layered phase specs:

```
RULE: Read L1 first (contracts), then L2 (workflows), then L3 (config)
RULE: Marker element IDs use phase/layer prefix: phase1/L1::Functions.parse_spec
RULE: Use nlspec_layer_stack to understand the full derivation tree
RULE: Use nlspec_layer_validate to check cross-layer constraints
RULE: L1 changes affect L2+L3+L4 — check impact before modifying
RULE: L3 config changes do NOT affect L1 contracts (config is independent)
```

---

## Working with Multiple Specs (Phase 2)

```
RULE: Use namespace-qualified queries
RULE: nlspec_slice follows cross-spec imports automatically
RULE: Never modify another spec's elements — patch it instead
RULE: Check impact with nlspec_graph before changing shared records
RULE: After structural changes, run nlspec_validate
```

---

## Dual Interface (MCP + gRPC)

The nlspec server exposes two interfaces:
- **MCP via SSE (:8080)** — for coding agents connecting interactively
- **gRPC (:50060)** — for internal consumers (high-throughput spec queries, graph traversal)

Coding agents always use MCP. The gRPC interface is for programmatic consumers
that need high-throughput access (bulk spec queries, graph traversal, version checks).

Proto definitions: `nlspec/proto/nlspec.proto`
