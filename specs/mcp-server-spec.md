# nlspec MCP Server — Natural Language Specification

> **Version:** 0.1.0
> **Author:** Divyendu Deepak Singh
> **Date:** February 2026
> **Type:** SYSTEM
> **License:** CC-BY-4.0

> **IMPORT:** This spec imports from `nlspec/bootstrap` (specs/bootstrap-spec.md).
> All RECORDs, ENUMs, FUNCTIONs, and MCP TOOLs defined in bootstrap are available
> here without redefinition. This spec EXTENDS the bootstrap system.

---

## Table of Contents

1. [Abstract](#abstract)
2. [Problem Statement](#problem-statement)
3. [Architecture Overview](#architecture-overview)
4. [Data Model](#datamodel)
5. [Core Functions](#functions)
6. [API Surface — MCP Tools](#api--mcp-tools)
7. [Error Model](#errors)
8. [Configuration](#config)
9. [Deployment Artifacts](#deployment-artifacts)
10. [Scenarios](#scenarios)
11. [Dependencies](#dependencies)
12. [File Structure](#filestructure)
13. [Maintenance Workflow](#maintenance-workflow)
14. [Build and Run](#buildandrun-and-run)
15. [Boundaries](#boundaries)
16. [Contracts](#contracts-contracts)

---

## Abstract

The nlspec MCP Server extends the bootstrap system (parser, store, query engine, 8 CRUD
tools) with context slicing, patch management, spec validation, graph operations,
namespaces, spec decomposition, substrate detection and querying, S3+OTF storage,
and the `nlspec_import` tool.

This spec supports the phase transitions described in NLSPEC-SYSTEM.md:
- **Phase 1 → 2:** `nlspec_split` decomposes a monolith spec into N specs with
  correct imports, helping you transition from single-spec to multi-spec.
- **Phase 2 → 3:** Namespaces, validation, and patch management scale to
  organizational complexity.
- **Self-bootstrap:** Once running, the system imports and manages its own specs
  using the same tools it provides.

---

## Problem Statement

### Current State
The bootstrap system provides CRUD + search on spec elements. An agent can read, write,
and query elements in any loaded spec.

### Deficiency
Six capabilities are missing for production use:

1. **Context slicing** — the agent reads full sections when fixing bugs. It should read
   only the minimal set of elements reachable from the failing scenario.
2. **Patch management** — there is no structured way to track bug fixes. Patches are
   ad-hoc changes with no lifecycle, no categorization, no absorption workflow.
3. **Validation** — there is no way to check a spec's structural integrity (dangling
   references, orphaned records, untested sections).
4. **Namespace & import** — specs are loaded by file path. There is no namespace system
   for organizing specs across projects, and no tool for importing specs into the
   running system.
5. **Substrate awareness** — when a project branches into multiple platforms or versions,
   agents have no way to navigate the 3D spec topology. They need a linear chain
   (L1→L4) for a specific substrate, not the full branching tree.
6. **Cloud storage** — specs are read from local filesystem only. Production deployments
   need S3-backed storage with on-the-fly graph building and cache invalidation.

### Target State
A complete MCP server that supports the full spec lifecycle: write → validate → implement
→ fix (with context slicing) → patch → absorb → evolve. Specs live in namespaces and
are imported via `nlspec_import`. The system manages its own specs the same way it
manages any other project's specs. Agents navigate multi-substrate projects via linear
chain queries. Spec files can live in S3 with on-the-fly graph building in memory.

### Key Insight
Once you have `nlspec_import`, the system bootstraps itself. Import the bootstrap spec
and this spec into the running server. From that point, every change to nlspec itself
goes through the same tools: `nlspec_slice` to understand the impact, `nlspec_patch_create`
to track the fix, `nlspec_validate` to verify integrity, `nlspec_patch_absorb` to merge
it back.

---

## Architecture Overview

```
+-----------------------------------------------------------+
|                    MCP Clients                             |
|  (Claude Code, Cursor, Claude Desktop, any MCP client)    |
+---------------------------+-------------------------------+
                            | MCP Protocol (stdio)
                            v
+-----------------------------------------------------------+
|                   nlspec MCP Server                        |
|                                                            |
|  +-----------+  +------------+  +-------------+           |
|  | Bootstrap |  | Advanced   |  | Namespace   |           |
|  | Tools (8) |  | Tools (10) |  | Manager     |           |
|  +-----------+  +------------+  +-------------+           |
|       |               |               |                    |
|       +-------+-------+-------+-------+                    |
|               |               |                            |
|    +----------v-----------+   |                            |
|    |    Core Engine       |   |                            |
|    |  (from bootstrap)    |   |                            |
|    |  - Spec Parser       |   |                            |
|    |  - Spec Store        |   |                            |
|    |  - Query Engine      |   |                            |
|    +----------+-----------+   |                            |
|               |               |                            |
|    +----------v-----------+   |                            |
|    |  Extended Engine     |<--+                            |
|    |  - Context Slicer    |                                |
|    |  - Patch Manager     |                                |
|    |  - Spec Validator    |                                |
|    |  - Graph Engine      |                                |
|    |  - Substrate Engine  |                                |
|    +----------+-----------+                                |
|               |                                            |
|    +----------v-----------+                                |
|    |  Persistence (S3+OTF)|                                |
|    |  Source of truth:     |                                |
|    |  - .md files          |                                |
|    |    (local or S3)      |                                |
|    |  Derived indices:     |                                |
|    |  - SQLite (elements)  |                                |
|    |  - In-memory graph    |                                |
|    |    (substrates)       |                                |
|    |  - patches/ dir       |                                |
|    +----------------------+                                |
+-----------------------------------------------------------+
```

### Architecture.1 Component Inventory (additions to bootstrap)

### Component: Context Slicer
- **Responsibility:** Given a section or scenario, trace USES/USED BY/IMPORT edges and
  extract the minimal set of SpecElements needed for a bug fix
- **Owns:** Dependency graph traversal logic
- **Calls:** Query Engine (to follow references), Spec Store (to read elements)
- **Called by:** Core Engine
- **Lifecycle:** Per-slice operation, read-only

### Component: Patch Manager
- **Responsibility:** Create, list, absorb, and delete patches. Track patch lifecycle.
- **Owns:** Patch files in patches/ directory, patch metadata
- **Calls:** Spec Store (to modify specs on absorb)
- **Called by:** Core Engine
- **Lifecycle:** Persistent across sessions

### Component: Spec Validator
- **Responsibility:** Check structural integrity of a spec (dangling references,
  orphaned records, missing sections, untested sections)
- **Owns:** Validation rules
- **Calls:** Query Engine, Spec Store
- **Called by:** Core Engine
- **Lifecycle:** Per-validation, read-only

### Component: Graph Engine
- **Responsibility:** Compute dependency graphs for specs or elements
- **Owns:** Graph traversal logic
- **Calls:** Query Engine
- **Called by:** Core Engine
- **Lifecycle:** Per-query, read-only

### Component: Substrate Engine
- **Responsibility:** Detect substrate branching in the DERIVES FROM graph, build
  the 3D substrate topology, resolve linear spec chains for agent consumption
- **Owns:** In-memory substrate graph (cached), substrate detection heuristics
- **Calls:** Graph Engine (for DERIVES FROM traversal), Spec Store (for spec headers)
- **Called by:** Core Engine, nlspec_substrate_query, nlspec_substrate_graph
- **Lifecycle:** Cached across queries, invalidated on spec import/change

### Component: Namespace Manager
- **Responsibility:** Organize specs into namespaces. Resolve namespace-qualified
  identifiers. Handle `nlspec_import`.
- **Owns:** Namespace registry, namespace-to-path mapping
- **Calls:** Spec Store, Spec Parser
- **Called by:** Core Engine, all MCP tools
- **Lifecycle:** Persistent across sessions

### Architecture.2 Data Flows (additions to bootstrap)

### Flow: Agent requests a context slice for a bug fix
1. Agent calls `nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})`
2. MCP Server routes to Context Slicer
3. Context Slicer reads SCENARIO 7, extracts [SEC:Functions.3] [SEC:DataModel.1] tags
4. Context Slicer reads Functions.3, extracts USES: QueryRequest, QueryResponse
5. Context Slicer reads DataModel for QueryRequest, QueryResponse RECORDs
6. Context Slicer reads Functions.3 THROWS, pulls relevant errors from Errors
7. If any USES target is an IMPORT, follows cross-spec reference (may cross namespaces)
8. Assembles all elements into a context slice
9. Returns the slice

Latency budget: < 100ms

### Flow: Agent imports a spec into the system
1. Agent calls `nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})`
2. Namespace Manager registers "nlspec/bootstrap" → "specs/bootstrap-spec.md"
3. Spec Parser parses the file
4. Spec Store indexes all elements under namespace "nlspec", spec_id "bootstrap"
5. Returns the Spec record

Latency budget: < 500ms (first import, includes full parse)

---

## DataModel

IMPORT Spec, Section, SpecElement, Reference, SpecMetadata, SearchResult,
       SpecImport, ElementType, ReferenceType FROM nlspec/bootstrap DataModel

### DataModel.1 Additional Records

```
RECORD Namespace:
  name         : String              -- namespace identifier (e.g., "nlspec", "myproject", "org/team")
  specs        : List<String>        -- spec IDs in this namespace
  created_at   : Timestamp

  USED BY: namespace_manager, nlspec_import, nlspec_list

  INVARIANTS:
  - name matches [a-z0-9][a-z0-9-/]* (lowercase, hyphens, forward slashes for hierarchy)
  - name is unique across the system
```

```
RECORD ContextSlice:
  trigger      : String              -- what caused the slice (e.g., "SCENARIO 7", "Functions.3")
  elements     : List<SpecElement>   -- the assembled elements, ordered by section then position
  total_lines  : u64                 -- approximate rendered size in lines
  specs_touched: List<String>        -- which specs contributed elements (may cross namespaces)
  trace        : List<String>        -- dependency chain as human-readable strings

  USED BY: context_slicer, nlspec_slice
```

```
RECORD Patch:
  id           : String              -- "PATCH-{NNN}" auto-assigned
  namespace    : String              -- namespace of the spec being patched
  spec_id      : String              -- which spec this patches
  category     : PatchCategory       -- A (spec deficiency), B (impl error), C (missing capability)
  status       : PatchStatus         -- ACTIVE, ABSORBED, REJECTED
  sections     : List<String>        -- affected section names
  scenarios    : List<String>        -- failing scenario numbers
  description  : String              -- what the patch fixes
  content      : String              -- the patch spec markdown content
  path         : String              -- filesystem path to patch file
  created_at   : Timestamp
  absorbed_at  : Timestamp | None

  USED BY: patch_manager, nlspec_patch_create, nlspec_patch_list, nlspec_patch_absorb
```

```
RECORD ValidationResult:
  namespace    : String
  spec_id      : String
  parse_mode   : String              -- "strict" (has SECTIONS) or "loose" (discovered)
  errors       : List<ValidationError>
  warnings     : List<ValidationWarning>
  gaps         : GapReport           -- structural gap analysis (from bootstrap parser)
  is_valid     : bool                -- true if zero errors (warnings and gaps are OK)

  USED BY: spec_validator, nlspec_validate
```

```
RECORD ValidationError:
  element_id   : String | None       -- which element has the error (None for spec-level)
  error_type   : String              -- "dangling_reference", "missing_section", etc.
  message      : String
  suggestion   : String | None

  USED BY: spec_validator
```

```
RECORD ValidationWarning:
  element_id   : String | None
  warning_type : String
  message      : String

  USED BY: spec_validator
```

```
RECORD LayerStackEntry:
  spec_id         : String              -- namespace/spec-id of the spec at this layer
  layer           : Integer             -- layer number (1-4)
  layer_name      : String              -- Specification | Realization | Configuration | UserProfile
  derives_from    : String | None       -- parent spec-id (None for L1)
  children        : List<String>        -- child spec-ids that derive from this
  status          : String              -- "exists" | "planned"
  one_line        : String              -- one-line purpose from the LAYER STACK
  composes_with   : List<CompositionPeer>  -- horizontal peers at the same layer

  USED BY: build_layer_stack, validate_layer_constraints
```

```
RECORD CompositionPeer:
  spec_id         : String              -- the peer spec's identifier
  interface       : String              -- what this spec exposes to the peer
  depends_on      : String              -- what this spec needs from the peer
  relationship    : String              -- "co-required" | "optional" | "alternative"

  USED BY: build_layer_stack, validate_layer_constraints
```

```
RECORD LayerValidationReport:
  spec_id         : String              -- the spec being validated
  layer           : Integer             -- its layer number
  derivation_valid: Boolean             -- DERIVES FROM chain is consistent
  downward_satisfied: List<String>      -- parent EXPORTS this spec honors
  downward_violated: List<String>       -- parent EXPORTS this spec violates
  upward_conflicts: List<String>        -- upward exports that conflict with parent NEVER constraints
  lateral_issues  : List<String>        -- issues from horizontal peer validation
  cross_layer_refs: List<String>        -- all [Ln:...] refs and whether they resolve
  overall         : String              -- PASS | WARN | FAIL

  USED BY: validate_layer_constraints
```

```
RECORD Substrate:
  id              : String              -- derived identifier (e.g., "ios", "phase-2a", "with-payments")
  type            : SubstrateType       -- spatial | temporal | feature
  branching_layer : Integer             -- the layer at which this substrate branches (1-4)
  branching_spec  : String              -- namespace/spec-id of the spec that branches into this substrate
  root_spec       : String              -- namespace/spec-id of this substrate's root (the branching child)
  stack           : List<LayerStackEntry>  -- the full L1→L4 chain within this substrate
  cross_substrate : List<String>        -- spec-ids from other substrates that IMPORT from this one

  USED BY: build_substrate_graph, resolve_substrate_chain, nlspec_substrate_query

  INVARIANTS:
  - branching_layer >= 1 and <= 4
  - root_spec exists in the store
  - stack is ordered L1→L4 with no gaps at the branching point
```

```
RECORD SubstrateGraph:
  namespace       : String              -- the namespace this graph covers
  substrates      : List<Substrate>     -- all detected substrates
  branching_points: List<BranchingPoint>  -- where the spec graph fans out
  total_specs     : u64                 -- total specs across all substrates
  built_at        : Timestamp           -- when this graph was last computed

  USED BY: build_substrate_graph, nlspec_substrate_graph

  INVARIANTS:
  - every spec in the namespace appears in exactly one substrate's stack
    (or in the shared ancestor chain above a branching point)
```

```
RECORD BranchingPoint:
  spec_id         : String              -- the parent spec that branches
  layer           : Integer             -- the layer of the parent spec
  children        : List<String>        -- the child spec-ids that form substrate roots
  substrate_type  : SubstrateType       -- inferred type of the branching

  USED BY: build_substrate_graph
```

```
RECORD SubstrateChain:
  namespace       : String
  substrate_id    : String              -- which substrate was resolved
  chain           : List<LayerStackEntry>  -- linear L1→L4 spec chain for this substrate
  total_specs     : u64                 -- number of specs in the chain
  shared_ancestors: List<String>        -- specs above the branching point (shared with other substrates)

  USED BY: resolve_substrate_chain, nlspec_substrate_query
```

### DataModel.2 Additional Enumerations

```
ENUM PatchCategory:
  SPEC_DEFICIENCY     -- Category A: spec was wrong
  IMPL_ERROR          -- Category B: implementation was wrong
  MISSING_CAPABILITY  -- Category C: spec was incomplete
```

```
ENUM PatchStatus:
  ACTIVE
  ABSORBED
  REJECTED
```

```
ENUM SubstrateType:
  SPATIAL       -- platform or target variants coexisting simultaneously (web, iOS, android)
  TEMPORAL      -- version history — the same system evolving over time (phase-1, phase-2a)
  FEATURE       -- feature flag variants — different capabilities enabled
```

### DataModel.3 Namespace-Qualified Identifiers

All spec identifiers become namespace-qualified:

```
Full element ID:   {namespace}/{spec_id}:{section}:{type}:{name}
Example:           myproject/auth:Functions.3:function:validate_token

Short spec ID:     {namespace}/{spec_id}
Example:           nlspec/bootstrap

Namespace only:    {namespace}
Example:           myproject
```

When namespace is omitted, it defaults to the "default" namespace.
Bootstrap tools (nlspec_get, nlspec_search, etc.) accept an optional `namespace`
parameter. Existing behavior is preserved when namespace is not specified.

---

## Functions

### Functions.1 Context Slicer

```
FUNCTION slice_for_scenario(namespace: String, spec_id: SpecId, scenario_number: u64) -> ContextSlice
  USES: ContextSlice, SpecElement, Reference
  THROWS: NotFound

  BEHAVIOR:
  1. Find SCENARIO with the given number in the given spec
  2. Extract [SEC:x.x] tags from the scenario
  3. For each tagged section:
     a. Read all FUNCTIONs in that section
     b. For each FUNCTION, follow USES references to RECORDs
     c. For each FUNCTION, follow THROWS references to error types
     d. For each USES/THROWS target, if it's an IMPORT, resolve cross-spec (may cross namespaces)
  4. Collect the scenario itself + all reached elements
  5. Remove duplicates
  6. Order by: declaration order, then element position
  7. Build the trace (dependency chain as human-readable string)
  8. Return ContextSlice

  NOTES:
  - Traversal depth is bounded at 3 hops (SCENARIO -> FUNCTION -> RECORD -> done)
  - Cross-spec IMPORTs are followed across namespaces
  - The slice includes ALL scenarios tagged with the same sections (for related context)

  PERFORMANCE:
  - Target: < 100ms
  - Typical slice: 200-500 lines (10-30 elements)
```

```
FUNCTION slice_for_section(namespace: String, spec_id: SpecId, section: SectionName) -> ContextSlice
  USES: ContextSlice
  THROWS: NotFound

  BEHAVIOR:
  1. Read all elements in the given section
  2. For each element, follow outgoing references (USES, THROWS, IMPORTS)
  3. Find all SCENARIOs tagged with this section
  4. Assemble and return ContextSlice
```

### Functions.2 Patch Manager

```
FUNCTION patch_create(namespace: String, spec_id: SpecId, category: PatchCategory, sections: List<SectionName>, scenarios: List<u64>, description: String, content: String) -> Patch
  USES: Patch
  THROWS: IOError

  BEHAVIOR:
  1. Assign next patch number (PATCH-001, PATCH-002, etc.)
  2. Create patch file in patches/{namespace}/{spec_id}/ directory
  3. Set status to ACTIVE
  4. Return the Patch record
```

```
FUNCTION patch_list(namespace: String | None, spec_id: SpecId | None, status: PatchStatus | None) -> List<Patch>
  USES: Patch

  BEHAVIOR:
  1. Scan patches/ directory
  2. Filter by namespace, spec_id, and/or status
  3. Return ordered by creation date
```

```
FUNCTION patch_absorb(patch_id: String) -> Patch
  USES: Patch
  THROWS: NotFound, PatchConflict

  BEHAVIOR:
  1. Read the patch
  2. Apply patch content to the main spec (update affected sections)
  3. Set patch status to ABSORBED
  4. Bump spec version (patch level)
  5. Move patch file to patches/absorbed/
  6. Return the updated Patch

  NOTES:
  - If the spec has changed since the patch was created (section content differs),
    throw PatchConflict — human must resolve
```

### Functions.3 Spec Validator

```
FUNCTION validate_spec(namespace: String, spec_id: SpecId) -> ValidationResult
  USES: ValidationResult, ValidationError, ValidationWarning, GapReport
  THROWS: NotFound

  BEHAVIOR:
  1. Load the spec. Determine parse mode from the Spec record:
     - If section_decl was parsed from a SECTIONS block → strict mode
     - If section_decl was inferred from ## headers → loose mode
  2. Run gap analysis (bootstrap parser's analyze_gaps). Attach GapReport.
  3. Check all References resolve:
     a. Every USES target exists as a RECORD or ENUM
     b. Every THROWS target exists in the error definitions
     c. Every IMPORT target exists in the referenced spec (may cross namespaces)
     d. Every [SEC:x.x] tag references an existing section
  4. Check symmetry: if A USES B, B should have USED BY A
  5. (Strict mode only) Check declared sections vs present sections:
     a. Sections in declaration but missing from markdown → error
     b. Sections in markdown but not in declaration → warning
  6. (Both modes) Check element completeness:
     a. All SCENARIOs have at least one [SEC:] tag
     b. All FUNCTIONs have USES and THROWS declarations
  7. Produce warnings for:
     a. RECORDs with no USED BY (orphaned records)
     b. Sections with no SCENARIOs (untested sections)
     c. FUNCTIONs with no PERFORMANCE targets
     d. (Loose mode) No SECTIONS block — suggest adding one for better tooling
  8. Return ValidationResult with parse_mode, errors, warnings, and gaps

  NOTES:
  - The validator NEVER refuses a spec. Even a minimal markdown file gets validated.
  - Gaps are advisory. is_valid is based on errors only (not warnings or gaps).
  - In loose mode, "missing declared sections" checks don't apply.
  - The completeness score in GapReport is a rough measure: 1.0 means no gaps,
    0.0 means the spec is essentially empty.

  PERFORMANCE:
  - Target: < 500ms for a project with 10 specs
```

### Functions.4 Graph Engine

```
FUNCTION compute_graph(namespace: String | None, spec_id: SpecId | None, element_id: ElementId | None, depth: u64, direction: String) -> Graph
  USES: SpecElement, Reference

  BEHAVIOR:
  1. If element_id: start from that element
  2. If spec_id: start from all elements in that spec
  3. If namespace only: start from all specs in that namespace
  4. Traverse references up to depth hops in the given direction
  5. Return nodes and edges

  RETURNS:
    nodes: List<{id: ElementId, type: ElementType, name: String, namespace: String}>
    edges: List<{from: ElementId, to: ElementId, ref_type: ReferenceType}>
```

### Functions.5 Namespace Manager

```
FUNCTION namespace_create(name: String) -> Namespace
  USES: Namespace
  THROWS: AlreadyExists

  BEHAVIOR:
  1. Validate name format
  2. Create namespace record
  3. Return Namespace
```

```
FUNCTION namespace_import(namespace: String, spec_id: String, path: String) -> Spec
  USES: Namespace, Spec
  THROWS: NotFound, ParseError, AlreadyExists

  BEHAVIOR:
  1. If namespace doesn't exist, create it
  2. Parse the spec file at path
  3. Register the spec under {namespace}/{spec_id}
  4. Index all elements with namespace-qualified IDs
  5. Return the Spec record

  NOTES:
  - This is how the system bootstraps itself:
    namespace_import("nlspec", "bootstrap", "specs/bootstrap-spec.md")
    namespace_import("nlspec", "mcp-server", "specs/mcp-server-spec.md")
  - After import, the spec is queryable via all MCP tools
  - Re-importing overwrites the existing index (re-parse from file)
```

### Functions.6 Spec Decomposition

```
RECORD SplitSuggestion:
  clusters     : List<SplitCluster>   -- suggested decomposition
  cross_refs   : List<Reference>      -- references that will become IMPORTs
  estimated_lines: List<u64>          -- estimated line count per cluster

  USED BY: spec_split, nlspec_split
```

```
RECORD SplitCluster:
  name         : String               -- suggested spec name (e.g., "auth", "storage")
  sections     : List<String>         -- section names in this cluster
  elements     : List<String>         -- element IDs in this cluster
  element_count: u64                  -- number of elements
  rationale    : String               -- why these elements cluster together

  USED BY: spec_split, nlspec_split
```

```
FUNCTION spec_split_suggest(namespace: String, spec_id: SpecId, strategy: String) -> SplitSuggestion
  USES: SplitSuggestion, SplitCluster, Reference
  THROWS: NotFound

  BEHAVIOR:
  1. Load the spec's dependency graph (all elements and their references)
  2. Apply clustering strategy:
     - "cluster": graph-based clustering — find groups of elements with dense
       internal references and sparse external references (community detection)
     - "by_section": group by top-level section name (DataModel + Functions become
       one spec, the API section becomes another, etc.)
     - "by_concern": use element names and tags to identify domain clusters
       (auth-related, storage-related, api-related)
  3. For each cluster, identify cross-cluster references that will become IMPORTs
  4. Estimate line counts per cluster
  5. Return SplitSuggestion (analysis only, does not modify anything)

  NOTES:
  - This is analysis-only. The human reviews the suggestion and decides.
  - The agent can then execute the split using spec_split_execute.
  - Strategies can be combined: cluster first, then refine by concern.
```

```
FUNCTION spec_split_execute(namespace: String, spec_id: SpecId, clusters: List<SplitCluster>, target_namespace: String) -> List<Spec>
  USES: SplitCluster, Spec
  THROWS: NotFound, IOError

  BEHAVIOR:
  1. For each cluster:
     a. Create a new spec file from the template
     b. Copy elements from the original spec into the new spec
     c. Add IMPORT declarations for cross-cluster references
     d. Generate Boundaries section noting the split origin
  2. Update the original spec's IMPORT declarations to reference new specs
  3. Register all new specs under target_namespace
  4. Return the list of created specs

  NOTES:
  - The original spec is preserved (not deleted). The human decides when to
    retire it.
  - IMPORT declarations are generated with correct section and element references.
  - SCENARIOs that span multiple clusters are copied to ALL relevant new specs
    (with appropriate [SEC:] tag updates).
```

### Functions.7 Layer Composition

```
FUNCTION build_layer_stack(spec_id: String, namespace: String) -> List<LayerStackEntry>
  USES: Spec, SpecMetadata, LayerStackEntry
  THROWS: SpecNotFound, ParseError

  BEHAVIOR:
  1. Read the target spec's LayerContext section
  2. Parse DERIVES FROM to find the parent spec
  3. Parse LAYER STACK to find siblings and children
  4. For each referenced spec, check if it exists in the store
  5. Mark existing specs as "exists", missing ones as "planned"
  6. When building the layer tree, also collect COMPOSES WITH declarations to populate
     composes_with on each LayerStackEntry. Horizontal peers share the same layer_number
     and parent_spec_id.
  7. Return the full tree as a list of LayerStackEntry, ordered L1→L2→L3→L4
```

```
FUNCTION validate_layer_constraints(spec_id: String, namespace: String) -> LayerValidationReport
  USES: Spec, SpecMetadata, LayerStackEntry, SeedExport, LayerValidationReport
  THROWS: SpecNotFound, ParseError

  BEHAVIOR:
  1. Build the layer stack for this spec via build_layer_stack
  2. Read this spec's DERIVES FROM to identify the parent
  3. Read the parent's Contracts section EXPORTS
  4. Check each parent EXPORT with layer_target matching this layer or "derived":
     a. If override=NEVER: verify this spec does not contradict it → downward_satisfied or downward_violated
     b. If override=WITH_JUSTIFICATION: check if justification is documented
  5. Read this spec's Contracts section for UPWARD exports
  6. For each UPWARD export, trace to the parent and check compatibility:
     a. If parent has a NEVER constraint on the same target → upward_conflicts
  7. For LATERAL constraint flow: verify that every depends_on in a COMPOSES WITH
     declaration is satisfied by an EXPORT from the referenced peer spec. Report
     unsatisfied lateral dependencies as validation issues.
  8. For co-required peers: verify all are present. Report missing co-required peers.
  9. For alternative peers: verify at most one is selected.
  10. Resolve all [Ln:...] cross-layer references in this spec:
     a. For each, check if the referenced spec and section exist
  11. Compute overall: PASS if no violations or conflicts, WARN if only unresolved planned refs, FAIL otherwise
```

### Functions.8 Substrate Engine

```
FUNCTION build_substrate_graph(namespace: String) -> SubstrateGraph
  USES: Substrate, SubstrateGraph, BranchingPoint, LayerStackEntry
  THROWS: NamespaceError

  BEHAVIOR:
  1. Load all specs in the given namespace from the store
  2. For each spec, read its Layer declaration and DERIVES FROM edge
  3. Build a directed graph: nodes are specs, edges are DERIVES FROM relationships
  4. Walk the graph to find branching points: any spec with 2+ children at the
     same or next layer
  5. For each branching point, create a BranchingPoint record
  6. For each branch, follow the chain downward through L3/L4 to build the
     full substrate stack
  7. Infer substrate type from naming conventions and graph topology:
     - If children are at the same layer as parent → likely spatial (multi-product)
     - If children form a progression (each DERIVES FROM previous) → temporal
     - Otherwise → spatial (default)
  8. Detect cross-substrate imports: specs that IMPORT from a different substrate
  9. Return SubstrateGraph with all substrates and branching points

  PERFORMANCE:
  - Target: < 200ms for a namespace with 50 specs
  - The graph is cached in memory and invalidated on spec change

  NOTES:
  - Substrate detection is heuristic — the type field is a best guess from topology
  - Substrates are not declared in specs; they emerge from the DERIVES FROM graph
  - A spec that has no branching below it is a single-substrate namespace (the trivial case)
```

```
FUNCTION resolve_substrate_chain(namespace: String, substrate_id: String, from_layer: Integer, to_layer: Integer) -> SubstrateChain
  USES: SubstrateChain, Substrate, LayerStackEntry
  THROWS: NotFound, NamespaceError

  BEHAVIOR:
  1. Build or retrieve cached substrate graph for the namespace
  2. Find the substrate matching substrate_id
  3. Walk the substrate's stack from from_layer to to_layer
  4. Include shared ancestors above the branching point
  5. Return SubstrateChain: a linear chain of specs the agent needs

  NOTES:
  - This is the key function for agent consumption: it cuts through the 3D topology
    and returns a flat, ordered list of specs
  - from_layer defaults to 1, to_layer defaults to 4
  - If the substrate has gaps (e.g., no L4), the chain stops at the deepest available layer
  - The chain includes shared ancestors so the agent sees the full derivation context

  PERFORMANCE:
  - Target: < 50ms (graph is cached, this is just a walk)
```

```
FUNCTION invalidate_substrate_cache(namespace: String) -> void
  BEHAVIOR:
  1. Mark the cached SubstrateGraph for the namespace as stale
  2. Next call to build_substrate_graph or resolve_substrate_chain will rebuild

  NOTES:
  - Called automatically when nlspec_import is used for a spec in this namespace
  - Called when the file watcher detects a spec file change
  - For S3 storage: triggered by S3 event notification via SNS/SQS
```

---

## API — MCP Tools

Eight additional MCP tools. Each takes JSON parameters, returns JSON results.
All bootstrap tools (nlspec_init, nlspec_get, nlspec_list, nlspec_search,
nlspec_create, nlspec_update, nlspec_delete) remain available and now accept
an optional `namespace` parameter.

### API.1 Import & Namespaces

```
MCP TOOL: nlspec_import
  DESCRIPTION: Import a spec file into the system under a namespace.
  This is the primary way specs enter the system.

  PARAMETERS:
    namespace    : String              -- namespace to import into (e.g., "myproject", "nlspec")
    spec_id      : String              -- spec identifier within the namespace
    path         : String              -- filesystem path to the markdown file

  RETURNS:
    spec         : Spec                -- the parsed and indexed spec
    namespace    : Namespace           -- the namespace (created if new)
    element_count: u64                 -- number of elements indexed

  EXAMPLES:
    nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})
    -> Imports the bootstrap spec. System now manages itself.

    nlspec_import({namespace: "myproject", spec_id: "auth", path: "../auth-service/specs/auth-spec.md"})
    -> Imports a project spec. Now queryable via nlspec_get, nlspec_search, etc.

  NOTES:
  - If the namespace doesn't exist, it is created automatically
  - If the spec already exists in the namespace, it is re-parsed and re-indexed
  - The markdown file remains the source of truth at its original path
  - The system only indexes; it does not copy the file
```

```
MCP TOOL: nlspec_namespaces
  DESCRIPTION: List all namespaces and their specs.

  PARAMETERS:
    namespace    : String | None       -- filter to a specific namespace

  RETURNS:
    namespaces   : List<Namespace>

  EXAMPLES:
    nlspec_namespaces({})
    -> Returns all namespaces: [{name: "nlspec", specs: ["bootstrap", "mcp-server"]},
                                {name: "myproject", specs: ["auth", "storage"]}]

    nlspec_namespaces({namespace: "nlspec"})
    -> Returns: [{name: "nlspec", specs: ["bootstrap", "mcp-server"]}]
```

### API.2 Context Slicing

```
MCP TOOL: nlspec_slice
  DESCRIPTION: Extract minimal context for a bug fix. Given a scenario or section,
  trace dependency edges and return only what the agent needs.

  PARAMETERS:
    namespace    : String              -- spec namespace
    spec_id      : SpecId              -- which spec
    scenario     : u64 | None          -- slice for this scenario number
    section      : SectionName | None -- slice for this section
    format       : "json" | "markdown" -- output format (default: "markdown")

  RETURNS:
    slice        : ContextSlice
    markdown     : String | None       -- if format is "markdown"

  EXAMPLE:
    nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
    -> Returns: {
        slice: {
          trigger: "SCENARIO 7",
          elements: [...],
          total_lines: 287,
          specs_touched: ["myproject/auth", "myproject/storage"],
          trace: ["SCENARIO 7 -> [SEC:Functions.3] -> FUNCTION validate_token -> USES TokenRecord -> ..."]
        }
      }
```

### API.3 Patch Management

```
MCP TOOL: nlspec_patch_create
  DESCRIPTION: Create a new patch for a spec.

  PARAMETERS:
    namespace    : String
    spec_id      : SpecId
    category     : PatchCategory
    sections     : List<SectionName>
    scenarios    : List<u64>
    description  : String
    content      : String              -- patch content in markdown

  RETURNS:
    patch        : Patch
```

```
MCP TOOL: nlspec_patch_list
  DESCRIPTION: List patches, optionally filtered.

  PARAMETERS:
    namespace    : String | None
    spec_id      : SpecId | None
    status       : PatchStatus | None

  RETURNS:
    patches      : List<Patch>
```

```
MCP TOOL: nlspec_patch_absorb
  DESCRIPTION: Absorb a patch into the main spec.

  PARAMETERS:
    patch_id     : String

  RETURNS:
    patch        : Patch               -- updated with ABSORBED status
    spec         : Spec                -- updated with bumped version
```

### API.4 Validation

```
MCP TOOL: nlspec_validate
  DESCRIPTION: Validate a spec's structural integrity and detect gaps.
  Returns parse mode (strict/loose), validation errors, warnings, and a
  GapReport with severity-categorized structural gaps. Works with any spec
  regardless of how well-structured it is — never refuses a spec.

  PARAMETERS:
    namespace    : String
    spec_id      : SpecId

  RETURNS:
    result       : ValidationResult

  EXAMPLE:
    nlspec_validate({namespace: "myproject", spec_id: "auth"})
    -> Returns: {
        result: {
          namespace: "myproject",
          spec_id: "auth",
          parse_mode: "strict",
          is_valid: false,
          errors: [{element_id: "myproject/auth:Functions.3:function:validate_token",
                    error_type: "dangling_reference",
                    message: "USES TokenRecord but no RECORD named TokenRecord found"}],
          warnings: [{element_id: "myproject/auth:DataModel.2:record:SessionStats",
                     warning_type: "orphaned",
                     message: "RECORD SessionStats has no USED BY references"}],
          gaps: {
            spec_id: "auth",
            mode: "strict",
            critical: [{severity: "CRITICAL", code: "GAP-002",
                       message: "FUNCTION validate_token references undefined RECORD TokenRecord",
                       section: "Functions.3", elements: ["validate_token", "TokenRecord"],
                       suggestion: "Add RECORD TokenRecord to DataModel or fix the USES reference"}],
            warnings: [{severity: "WARNING", code: "GAP-007",
                       message: "RECORD SessionStats has no USED BY references",
                       section: "DataModel.2", elements: ["SessionStats"],
                       suggestion: "Add USED BY or remove if truly unused"}],
            info: [],
            completeness: 0.82
          }
        }
      }

  EXAMPLE (loose spec):
    nlspec_validate({namespace: "sandbox", spec_id: "quick-prototype"})
    -> Returns: {
        result: {
          namespace: "sandbox",
          spec_id: "quick-prototype",
          parse_mode: "loose",
          is_valid: true,
          errors: [],
          warnings: [],
          gaps: {
            spec_id: "quick-prototype",
            mode: "loose",
            critical: [{severity: "CRITICAL", code: "GAP-001",
                       message: "No Scenarios section — nothing to validate against",
                       section: null, elements: [],
                       suggestion: "Add a Scenarios section with at least one SCENARIO"}],
            warnings: [{severity: "WARNING", code: "GAP-005",
                       message: "3 FUNCTIONs lack USES declarations",
                       section: "Core", elements: ["handle_request", "process_data", "emit_result"],
                       suggestion: "Add USES declarations for context slicing support"}],
            info: [{severity: "INFO", code: "GAP-011",
                   message: "No SECTIONS declaration — parser is in loose mode",
                   section: null, elements: [],
                   suggestion: "Add a SECTIONS block for better tooling and completeness checking"}],
            completeness: 0.45
          }
        }
      }
```

### API.5 Graph Operations

```
MCP TOOL: nlspec_graph
  DESCRIPTION: Get the dependency graph for a spec or element.

  PARAMETERS:
    namespace    : String | None       -- scope to namespace
    spec_id      : SpecId | None       -- scope to spec
    element_id   : ElementId | None    -- center on one element
    depth        : u64                 -- max traversal depth (default: 2)
    direction    : "outgoing" | "incoming" | "both"  -- default: "both"

  RETURNS:
    nodes        : List<{id: ElementId, type: ElementType, name: String, namespace: String}>
    edges        : List<{from: ElementId, to: ElementId, ref_type: ReferenceType}>

  NOTES:
  - "What will break if I change this RECORD?" -> graph(element_id, direction="incoming")
  - Graph traversal crosses namespace boundaries when following IMPORT references
```

### API.6 Spec Decomposition

```
MCP TOOL: nlspec_split
  DESCRIPTION: Analyze a spec and suggest how to decompose it into smaller specs.
  Supports the Phase 1 → Phase 2 transition (single spec → multiple specs).

  PARAMETERS:
    namespace    : String              -- current namespace of the spec
    spec_id      : SpecId              -- spec to analyze
    strategy     : "cluster" | "by_section" | "by_concern"  -- decomposition strategy
    execute      : bool                -- false = suggest only, true = execute the split
    target_namespace : String | None   -- namespace for new specs (default: same namespace)

  RETURNS:
    suggestion   : SplitSuggestion     -- always returned (the analysis)
    created_specs: List<Spec> | None   -- only if execute=true

  EXAMPLES:
    nlspec_split({namespace: "myproject", spec_id: "monolith", strategy: "cluster", execute: false})
    -> Returns: {
        suggestion: {
          clusters: [
            {name: "auth", sections: ["DataModel.1", "DataModel.2", "Functions.1", "Functions.2"], element_count: 47,
             rationale: "Dense references between auth records and auth functions"},
            {name: "storage", sections: ["DataModel.3", "Functions.3", "Functions.4"], element_count: 31,
             rationale: "Storage records and functions form isolated subgraph"},
            {name: "api", sections: ["API.1", "API.2", "API.3"], element_count: 22,
             rationale: "API endpoints reference auth and storage but are self-contained"}
          ],
          cross_refs: [
            {from: "auth:Functions.1:function:validate_token", to: "storage:DataModel.3:record:Session"},
            ...
          ],
          estimated_lines: [850, 620, 440]
        }
      }

    nlspec_split({namespace: "myproject", spec_id: "monolith", strategy: "cluster",
                  execute: true, target_namespace: "myproject"})
    -> Creates: myproject/auth, myproject/storage, myproject/api
    -> Each with correct IMPORT declarations
    -> Original spec preserved

  NOTES:
  - Always suggest first (execute=false), review with user, then execute
  - The original spec is never deleted — user decides when to retire it
  - Scenarios spanning multiple clusters are duplicated to all relevant new specs
  - execute=true requires human confirmation (the tool asks before proceeding)
```

### API.7 Drift Detection

```
MCP TOOL: nlspec_drift
  DESCRIPTION: Compare a spec's declarations against the actual codebase to detect
  divergence. Checks API endpoints, data model records, error types, config schema,
  and artifact outputs. Returns a structured drift report.

  The agent provides the code analysis — the tool provides the spec side. The agent
  reads code files and calls this tool with what it found. The tool compares against
  the spec's declarations and reports mismatches.

  PARAMETERS:
    namespace    : String
    spec_id      : SpecId
    code_surface : {                    -- what the agent found in the code
      endpoints  : List<{method: String, path: String, params: List<String>}> | None
      records    : List<{name: String, fields: List<{name: String, type: String}>}> | None
      errors     : List<{name: String, code: String}> | None
      configs    : List<{key: String, type: String, default: String}> | None
      artifacts  : List<{name: String, path: String, exists: bool}> | None
    }

  RETURNS:
    drift_report : {
      spec_id    : String,
      clean      : bool,                -- true if no drift detected
      drifted    : List<{
        category : "API" | "DATA_MODEL" | "ERROR" | "CONFIG" | "ARTIFACT",
        element  : String,              -- element name
        spec_says: String,              -- what the spec declares
        code_says: String,              -- what the agent found in code
        severity : "MISSING_IN_CODE" | "EXTRA_IN_CODE" | "MISMATCH"
      }>
    }

  EXAMPLE:
    nlspec_drift({
      namespace: "myproject", spec_id: "auth",
      code_surface: {
        endpoints: [
          {method: "POST", path: "/auth/login", params: ["username", "password"]},
          {method: "GET", path: "/auth/status", params: []},
          {method: "DELETE", path: "/auth/purge", params: ["before_date"]}
        ],
        records: [
          {name: "Token", fields: [{name: "id", type: "String"}, {name: "expires", type: "Timestamp"}]}
        ]
      }
    })
    -> Returns: {
        drift_report: {
          spec_id: "auth",
          clean: false,
          drifted: [
            {category: "API", element: "DELETE /auth/purge",
             spec_says: "not declared", code_says: "DELETE /auth/purge with param before_date",
             severity: "EXTRA_IN_CODE"},
            {category: "DATA_MODEL", element: "Token.user_id",
             spec_says: "field user_id: UserId", code_says: "field not present",
             severity: "MISSING_IN_CODE"}
          ]
        }
      }

  NOTES:
  - The agent is responsible for reading code and extracting the code_surface.
    This tool only compares against the spec.
  - Drift is advisory — EXTRA_IN_CODE might mean the spec needs updating,
    MISSING_IN_CODE might mean the code needs fixing. The agent reports, humans decide.
  - Used automatically during VALIDATE and CONSOLIDATE modes.
```

### API.8 Layer Composition

```
MCP TOOL: nlspec_layer_stack
  DESCRIPTION: Get the full layer composition tree for a spec
  INPUT:
    namespace : String
    spec_id   : String
  OUTPUT:
    stack     : List<LayerStackEntry>   -- full L1→L2→L3→L4 tree
    this_spec : String                  -- which entry is the queried spec
  ERRORS:
    SpecNotFound: spec doesn't exist
    NotLayered: spec has no Layer declaration
  NOTES:
    - Returns the full derivation tree, not just this spec's branch
    - Specs marked (planned) in the LAYER STACK are included with status "planned"
```

```
MCP TOOL: nlspec_layer_validate
  DESCRIPTION: Validate cross-layer constraint flow for a spec
  INPUT:
    namespace : String
    spec_id   : String
  OUTPUT:
    report    : LayerValidationReport
  ERRORS:
    SpecNotFound: spec doesn't exist
    NotLayered: spec has no Layer declaration
  NOTES:
    - Checks both downward (inherited) and upward (propagated) constraints
    - Cross-layer references to planned specs are flagged as WARN, not FAIL
```

### API.9 Substrate Operations

```
MCP TOOL: nlspec_substrate_query
  DESCRIPTION: Resolve a substrate chain — get the linear spec path from L1 to L4
  for a specific substrate. This is the primary tool for agents navigating complex
  multi-substrate projects. Instead of understanding the full 3D topology, the agent
  asks for a specific substrate and gets a flat, ordered list of specs.

  PARAMETERS:
    namespace    : String              -- the namespace to query
    substrate    : String              -- substrate identifier (e.g., "ios", "phase-2a", "web-prod-us")
    from_layer   : Integer | None      -- start layer (default: 1)
    to_layer     : Integer | None      -- end layer (default: 4)
    include_content : bool             -- if true, include spec content (default: false, returns refs only)

  RETURNS:
    chain        : SubstrateChain      -- linear L1→L4 chain for this substrate
    specs        : List<{spec_id: String, layer: Integer, content: String | None}>

  EXAMPLES:
    nlspec_substrate_query({namespace: "payments", substrate: "ios"})
    -> Returns: {
        chain: {
          namespace: "payments",
          substrate_id: "ios",
          chain: [
            {spec_id: "payments/l1-contracts", layer: 1, ...},
            {spec_id: "payments/l2-ios-realization", layer: 2, ...},
            {spec_id: "payments/l3-ios-prod-config", layer: 3, ...},
            {spec_id: "payments/l4-ios-us-profile", layer: 4, ...}
          ],
          total_specs: 4,
          shared_ancestors: ["payments/l1-contracts"]
        }
      }

    nlspec_substrate_query({namespace: "nlspec", substrate: "phase-2a", from_layer: 1, to_layer: 2})
    -> Returns the bootstrap + mcp-server specs (Phase 2a substrate, L1-L2 only)

  NOTES:
  - If the substrate doesn't exist, returns NotFound with a list of available substrates
  - Shared ancestors (specs above the branching point) are included in the chain
  - With include_content=true, each spec's full markdown content is returned
    (useful for agents that need to read the specs inline)
  - The agent does NOT need to understand substrate topology — just ask for a name
```

```
MCP TOOL: nlspec_substrate_graph
  DESCRIPTION: Get the full substrate topology for a namespace. Returns all detected
  substrates, branching points, and cross-substrate dependencies. Used by visualization
  tools (3D UIs) and for understanding project topology.

  PARAMETERS:
    namespace    : String              -- the namespace to analyze
    include_cross: bool                -- include cross-substrate import edges (default: false)

  RETURNS:
    graph        : SubstrateGraph      -- full substrate topology

  EXAMPLES:
    nlspec_substrate_graph({namespace: "payments"})
    -> Returns: {
        graph: {
          namespace: "payments",
          substrates: [
            {id: "web", type: "SPATIAL", branching_layer: 1, root_spec: "payments/l2-web-realization", ...},
            {id: "ios", type: "SPATIAL", branching_layer: 1, root_spec: "payments/l2-ios-realization", ...},
            {id: "android", type: "SPATIAL", branching_layer: 1, root_spec: "payments/l2-android-realization", ...}
          ],
          branching_points: [
            {spec_id: "payments/l1-contracts", layer: 1, children: ["payments/l2-web-realization", "payments/l2-ios-realization", "payments/l2-android-realization"], substrate_type: "SPATIAL"}
          ],
          total_specs: 13,
          built_at: "2026-03-20T..."
        }
      }

  NOTES:
  - This is an expensive operation (builds the full graph). Results are cached.
  - With include_cross=true, each substrate's cross_substrate field is populated
    with specs from other substrates that IMPORT from it
  - A namespace with no branching returns a single-substrate graph (the trivial case)
  - Used by L2Mify's 3D visualization and other UI tools, not typically by coding agents
    (coding agents use nlspec_substrate_query for a specific chain)
```

---

## Errors

IMPORT NlspecError, ParseError, NotFound, IOError FROM nlspec/bootstrap Errors

Additional errors:

```
NlspecError (extended)
  +-- PatchConflict          -- spec has changed since patch was created
  +-- ValidationFailed       -- spec has structural errors (not the same as invalid input)
  +-- NamespaceError         -- invalid namespace name or namespace operation failed
  +-- AlreadyExists          -- namespace/spec combination already exists (on strict import)
  +-- CyclicImport           -- import creates a cycle (warning, not fatal)
```

---

## Config

IMPORT all config from nlspec/bootstrap Config

Additional configuration:

```
CONFIG nlspec.namespaces.default
  type: String
  default: "default"
  description: Namespace used when no namespace is specified in tool calls
  env: NLSPEC_DEFAULT_NAMESPACE

CONFIG nlspec.patches.dir
  type: String
  default: "patches/"
  description: Directory for patch files, organized by namespace/spec_id
  env: NLSPEC_PATCHES_DIR

CONFIG nlspec.validation.strict
  type: bool
  default: false
  description: When true, warnings are treated as errors in nlspec_validate
  env: NLSPEC_VALIDATION_STRICT

CONFIG nlspec.slice.max_depth
  type: u64
  default: 3
  description: Maximum reference traversal depth for context slicing
  env: NLSPEC_SLICE_MAX_DEPTH

CONFIG nlspec.import.auto_namespace
  type: bool
  default: true
  description: When true, nlspec_import creates namespaces automatically
  env: NLSPEC_IMPORT_AUTO_NAMESPACE

CONFIG nlspec.storage.backend
  type: String
  default: "local"
  description: Storage backend for spec files. "local" reads from filesystem,
               "s3" reads from an S3 bucket.
  env: NLSPEC_STORAGE_BACKEND
  values: "local" | "s3"

CONFIG nlspec.storage.s3.bucket
  type: String | None
  default: None
  description: S3 bucket name when storage backend is "s3"
  env: NLSPEC_S3_BUCKET

CONFIG nlspec.storage.s3.prefix
  type: String
  default: "specs/"
  description: S3 key prefix for spec files
  env: NLSPEC_S3_PREFIX

CONFIG nlspec.storage.s3.region
  type: String
  default: "us-east-1"
  description: AWS region for S3 bucket
  env: NLSPEC_S3_REGION

CONFIG nlspec.substrate.cache_ttl_ms
  type: u64
  default: 300000
  description: Time-to-live for the in-memory substrate graph cache (milliseconds).
               Set to 0 to disable caching (rebuild on every query).
  env: NLSPEC_SUBSTRATE_CACHE_TTL

CONFIG nlspec.storage.watch
  type: bool
  default: true
  description: Enable file watching for cache invalidation. For local storage,
               uses filesystem watcher. For S3, listens on SNS/SQS for S3 event
               notifications. When false, cache only invalidates on re-import.
  env: NLSPEC_STORAGE_WATCH
```

---

## Deployment Artifacts

### Deployment.1 Distribution

```
Distribution:
  PyPI:   pip install nlspec-server              (local dev, stdio transport)
  Docker: ghcr.io/nlspec/nlspec-server:latest    (production, SSE transport)

The package includes both bootstrap and MCP server capabilities.
The bootstrap tools are always available. The advanced tools (slice, patch,
validate, graph, import, namespaces, substrates) are part of the same server.
```

### Deployment.2 MCP Configuration

**Production (container, SSE):**
```json
{
  "mcpServers": {
    "nlspec": {
      "url": "http://nlspec-server:8080/sse"
    }
  }
}
```

**Local dev (stdio):**
```json
{
  "mcpServers": {
    "nlspec": {
      "command": "nlspec-server",
      "args": ["--specs-dir", "./specs"]
    }
  }
}
```

---

## Scenarios

### Scenarios.1 Self-Bootstrap

```
SCENARIO 1: System imports its own specs                            [SEC:API.1] [SMOKE]
  GIVEN: nlspec MCP server is running with no specs loaded
  WHEN: Agent calls nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})
  AND: Agent calls nlspec_import({namespace: "nlspec", spec_id: "mcp-server", path: "specs/mcp-server-spec.md"})
  THEN:
  - nlspec_namespaces returns [{name: "nlspec", specs: ["bootstrap", "mcp-server"]}]
  - nlspec_get({namespace: "nlspec", spec_id: "bootstrap", section: "DataModel.1"}) returns Spec record
  - nlspec_search({namespace: "nlspec", query: "ContextSlice"}) finds the record in mcp-server spec
```

```
SCENARIO 2: System validates its own specs                          [SEC:API.4] [SMOKE]
  GIVEN: Both nlspec specs are imported (SCENARIO 1 complete)
  WHEN: Agent calls nlspec_validate({namespace: "nlspec", spec_id: "bootstrap"})
  THEN: result.is_valid is true, zero errors
  WHEN: Agent calls nlspec_validate({namespace: "nlspec", spec_id: "mcp-server"})
  THEN: result.is_valid is true, zero errors
```

### Scenarios.2 Context Slicing

```
SCENARIO 3: Slice for a scenario                                    [SEC:API.2] [SEC:Functions.1]
  GIVEN: A spec "myproject/auth" is imported with:
    - DataModel.1: RECORD TokenRecord
    - Functions.3: FUNCTION validate_token (USES: TokenRecord, THROWS: AuthError)
    - Errors: AuthError definition
    - Scenarios: SCENARIO 7 tagged [SEC:Functions.3]
  WHEN: Agent calls nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  THEN:
  - slice.trigger is "SCENARIO 7"
  - slice.elements contains: SCENARIO 7, FUNCTION validate_token, RECORD TokenRecord, AuthError
  - slice.total_lines is between 50 and 500
  - slice.trace contains a human-readable dependency chain
```

```
SCENARIO 4: Slice follows cross-namespace imports                   [SEC:API.2] [SEC:Functions.1]
  GIVEN: "myproject/auth" has IMPORT UserRecord FROM myproject/users DataModel.1
  AND: "myproject/users" is imported with DataModel.1 containing RECORD UserRecord
  WHEN: Agent calls nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  AND: SCENARIO 7's dependency chain reaches IMPORT UserRecord
  THEN: slice.elements includes UserRecord from myproject/users
  AND: slice.specs_touched includes "myproject/auth" and "myproject/users"
```

### Scenarios.3 Patch Management

```
SCENARIO 5: Patch lifecycle                                         [SEC:API.3] [SEC:Functions.2]
  GIVEN: "myproject/auth" spec is imported, version "0.1.0"
  WHEN: Agent calls nlspec_patch_create({
    namespace: "myproject", spec_id: "auth",
    category: "IMPL_ERROR", sections: ["Functions.3"], scenarios: [7],
    description: "validate_token does not check expiry",
    content: "FUNCTION validate_token: add step 3: check token.expires_at > now()"
  })
  THEN: Returns patch with id "PATCH-001", status "ACTIVE"
  WHEN: Agent calls nlspec_patch_list({namespace: "myproject", spec_id: "auth"})
  THEN: Returns [PATCH-001] with status ACTIVE
  WHEN: Agent calls nlspec_patch_absorb({patch_id: "PATCH-001"})
  THEN: Patch status is "ABSORBED"
  AND: Spec version is "0.1.1"
```

### Scenarios.4 Validation

```
SCENARIO 6: Detect dangling reference                               [SEC:API.4] [SEC:Functions.3]
  GIVEN: A spec where FUNCTION foo USES: BarRecord, but no RECORD BarRecord exists
  WHEN: Agent calls nlspec_validate for that spec
  THEN: result.is_valid is false
  AND: errors contains {error_type: "dangling_reference", message contains "BarRecord"}
```

```
SCENARIO 7: Detect orphaned record                                  [SEC:API.4] [SEC:Functions.3]
  GIVEN: A spec where RECORD OrphanedThing exists but no FUNCTION has USES: OrphanedThing
  WHEN: Agent calls nlspec_validate for that spec
  THEN: warnings contains {warning_type: "orphaned", message contains "OrphanedThing"}
  AND: result.is_valid is true (warnings don't fail validation)
```

### Scenarios.5 Graph Operations

```
SCENARIO 8: Graph shows incoming dependencies                       [SEC:API.5] [SEC:Functions.4]
  GIVEN: RECORD Entry is USED BY: storage_get, storage_put, storage_delete
  WHEN: Agent calls nlspec_graph({element_id: "myproject/kv:4.1:record:Entry", direction: "incoming"})
  THEN: nodes includes storage_get, storage_put, storage_delete
  AND: edges show USED_BY relationships from each function to Entry
```

### Scenarios.6 Namespace Operations

```
SCENARIO 9: Multiple namespaces coexist                             [SEC:API.1]
  GIVEN: Specs imported under "nlspec" and "myproject" namespaces
  WHEN: Agent calls nlspec_search({query: "Spec"})
  THEN: Results include elements from BOTH namespaces
  WHEN: Agent calls nlspec_search({namespace: "nlspec", query: "Spec"})
  THEN: Results include ONLY elements from "nlspec" namespace
```

```
SCENARIO 10: Re-import refreshes index                              [SEC:API.1]
  GIVEN: "myproject/auth" is imported
  WHEN: Human edits the markdown file (adds a new RECORD)
  AND: Agent calls nlspec_import with the same namespace/spec_id/path
  THEN: The new RECORD appears in nlspec_search results
  AND: The old elements that still exist are preserved
```

### Scenarios.7 Spec Decomposition

```
SCENARIO 11: Suggest decomposition of a monolith spec               [SEC:API.6] [SEC:Functions.6]
  GIVEN: "myproject/monolith" is imported with 300+ elements across all sections
  AND: DataModel.1-DataModel.2, Functions.1-Functions.2 have dense internal references (auth cluster)
  AND: DataModel.3, Functions.3-Functions.4 have dense internal references (storage cluster)
  AND: API.1-API.3 reference both clusters (api cluster)
  WHEN: Agent calls nlspec_split({namespace: "myproject", spec_id: "monolith",
                                   strategy: "cluster", execute: false})
  THEN: suggestion.clusters has 3 entries
  AND: Each cluster has a name, sections, element_count, and rationale
  AND: cross_refs identifies references that will become IMPORTs
  AND: No files are created or modified (suggestion only)
```

```
SCENARIO 12: Execute decomposition creates new specs                 [SEC:API.6] [SEC:Functions.6]
  GIVEN: SCENARIO 11 suggestion has been reviewed and approved
  WHEN: Agent calls nlspec_split({namespace: "myproject", spec_id: "monolith",
                                   strategy: "cluster", execute: true})
  THEN: Three new spec files are created
  AND: Each new spec has correct IMPORT declarations for cross-cluster references
  AND: Each new spec follows the nlspec template
  AND: Original monolith spec is preserved (not deleted)
  AND: nlspec_validate passes for all three new specs
```

```
SCENARIO 13: Split preserves scenarios across clusters               [SEC:API.6] [SEC:Functions.6]
  GIVEN: Original spec has SCENARIO 7 tagged [SEC:Functions.1] [SEC:Functions.3]
  AND: Functions.1 is in the "auth" cluster, Functions.3 is in the "storage" cluster
  WHEN: Split is executed
  THEN: SCENARIO 7 appears in BOTH the auth spec and the storage spec
  AND: Each copy has appropriate [SEC:] tags for its own sections
```

### Scenarios.8 Substrate Operations

```
SCENARIO 14: Detect spatial substrates from DERIVES FROM graph         [SEC:API.9] [SEC:Functions.8]
  GIVEN: Namespace "payments" has:
    - L1 spec "l1-contracts" (Layer: 1-Specification)
    - L2 spec "l2-web" (Layer: 2-Realization, DERIVES FROM: l1-contracts)
    - L2 spec "l2-ios" (Layer: 2-Realization, DERIVES FROM: l1-contracts)
    - L2 spec "l2-android" (Layer: 2-Realization, DERIVES FROM: l1-contracts)
    - L3 spec "l3-web-prod" (Layer: 3-Configuration, DERIVES FROM: l2-web)
    - L3 spec "l3-ios-prod" (Layer: 3-Configuration, DERIVES FROM: l2-ios)
  WHEN: Agent calls nlspec_substrate_graph({namespace: "payments"})
  THEN:
  - graph.substrates has 3 entries (web, ios, android)
  - graph.branching_points has 1 entry at l1-contracts, layer 1
  - Each substrate's type is SPATIAL
  - web substrate's stack includes l1-contracts → l2-web → l3-web-prod
  - ios substrate's stack includes l1-contracts → l2-ios → l3-ios-prod
```

```
SCENARIO 15: Resolve linear substrate chain for an agent               [SEC:API.9] [SEC:Functions.8]
  GIVEN: SCENARIO 14 namespace is set up
  WHEN: Agent calls nlspec_substrate_query({namespace: "payments", substrate: "ios"})
  THEN:
  - chain.substrate_id is "ios"
  - chain.chain has 3 entries: l1-contracts (L1), l2-ios (L2), l3-ios-prod (L3)
  - chain.shared_ancestors includes "payments/l1-contracts"
  - chain.total_specs is 3
```

```
SCENARIO 16: Detect temporal substrates (NLSpec's own phases)          [SEC:API.9] [SEC:Functions.8]
  GIVEN: Namespace "nlspec" has:
    - bootstrap-spec (the core, no DERIVES FROM)
    - mcp-server-spec (DERIVES FROM bootstrap-spec)
    - seed-resolver-spec (DERIVES FROM mcp-server-spec)
  WHEN: Agent calls nlspec_substrate_graph({namespace: "nlspec"})
  THEN:
  - graph.substrates detected (each phase as a temporal substrate)
  - graph.branching_points exist where the chain fans out
  - Substrate type is TEMPORAL (progression detected)
```

```
SCENARIO 17: Substrate query with unknown substrate returns available list   [SEC:API.9]
  GIVEN: Namespace "payments" has substrates: web, ios, android
  WHEN: Agent calls nlspec_substrate_query({namespace: "payments", substrate: "desktop"})
  THEN: Error NotFound is returned
  AND: Error message includes available substrates: ["web", "ios", "android"]
```

---

## Dependencies

IMPORT dependencies from nlspec/bootstrap Dependencies

Additional: none. The MCP server spec uses only what bootstrap already provides
(Python 3.11+, FastMCP, sqlite3 stdlib, NetworkX, boto3).

---

## FileStructure

```
nlspec-project/
├── specs/
│   ├── bootstrap-spec.md            -- core system spec (this is an nlspec)
│   └── mcp-server-spec.md           -- this file (this is an nlspec)
├── patches/
│   ├── nlspec/                      -- patches for nlspec's own specs
│   │   ├── bootstrap/
│   │   └── mcp-server/
│   └── myproject/                   -- patches for project specs
│       ├── auth/
│       └── storage/
├── .nlspec/
│   ├── index.sqlite                 -- element index (all namespaces)
│   └── namespaces.json              -- namespace registry
├── CLAUDE.md                        -- agent instructions
├── NLSPEC-TEMPLATE.md               -- blank template for new specs
└── NLSPEC-SYSTEM.md                 -- system overview
```

---

## Maintenance Workflow

Same as bootstrap (Maintenance) with one addition:

**Self-maintenance:** Patches to the bootstrap spec or this spec follow the same
workflow as any other spec. Create a patch with `nlspec_patch_create`, fix the
implementation, validate with `nlspec_validate`, absorb with `nlspec_patch_absorb`.
The system maintains itself.

### Bug Categories (same as bootstrap)

| Category | Spec was... | Action |
|----------|------------|--------|
| A: Spec Deficiency | Wrong/ambiguous | Fix spec, add scenario, version bump |
| B: Implementation Error | Correct | Write patch, agent fixes code |
| C: Missing Capability | Incomplete | Add to spec, add scenarios, version bump |

---

## BuildAndRun and Run

### Prerequisites
Same as bootstrap: Python 3.11+, uv (recommended) or pip

### Install
```bash
uv pip install -e ".[dev]"
```

### Test
```bash
pytest                                    # all scenarios (bootstrap + mcp-server)
pytest -k "scenario_01"                   # specific scenario
```

### Run as MCP Server (local dev, stdio)
```bash
nlspec-server --specs-dir ./specs
```

### Run as MCP Server (container, SSE)
```bash
docker run -p 8080:8080 -v ./specs:/specs ghcr.io/nlspec/nlspec-server:latest \
    --transport sse --host 0.0.0.0 --port 8080 --specs-dir /specs
```

### Self-Bootstrap Sequence
```bash
# Start the server (local dev)
nlspec-server --specs-dir ./specs

# In an MCP client (Claude Code, Cursor, etc.):
nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "mcp-server", path: "specs/mcp-server-spec.md"})

# System now manages itself. Verify:
nlspec_validate({namespace: "nlspec", spec_id: "bootstrap"})
nlspec_validate({namespace: "nlspec", spec_id: "mcp-server"})
```

---

## Boundaries

### This Spec Adds:
- Context slicing (minimal context for bug fixes)
- Patch management (create, list, absorb lifecycle)
- Spec validation (structural integrity checks)
- Graph operations (dependency traversal)
- Namespaces (organize specs across projects)
- nlspec_import (bring specs into the system)
- nlspec_split (decompose monolith specs into multiple specs)
- Substrate detection and querying (3D spec topology, linear chain resolution)
- S3+OTF storage model (files as source of truth, in-memory graph as derived index)
- Self-management (system manages its own specs)
- Phase transition support (Phase 0 → 1 → 2 → 3)

### This Spec Does NOT:
- Redefine anything from bootstrap — it only extends
- Implement code generation — agent's job, not nlspec's
- Execute tests — agent runs tests using test runners
- Enforce specific project structure — namespaces are freeform
- Require all tools — bootstrap tools work standalone without these extensions
- Prevent namespace cycles — warns about them, like spec import cycles
- Cross-layer seed resolution (combining layer constraints with dependency constraints) — future extension via seed-resolver integration
- Cross-substrate interaction detection (identifying imports between substrates) — detected but not visualized; future extension
- Require a graph database — the substrate graph is an in-memory derived index, not a persistent store

### Future Extensions (not in this spec):
- nlspec_describe: MCP App rendering interactive spec overview (dashboard)
- nlspec_visualize: MCP App rendering interactive diagrams (D3.js force-directed, Three.js 3D substrate view)
- nlspec_diff: Compare two versions of a spec
- nlspec_migrate: Upgrade a spec to a newer template version
- Spec versioning beyond semver (branching, merging, temporal substrate diffing)
- Remote spec registries (pull specs from URLs or package registries)
- Cross-substrate contract validation (ensuring cross-substrate imports are satisfied)
- Substrate-aware patch management (patches scoped to a specific substrate)

---

## Contracts Contracts

```
### EXPORTS

EXPORT ContextSlicer:
  type        : CONSTRAINT
  target      : agent_context
  condition   : ALWAYS
  value       : "Dependency-traced context slicing via nlspec_slice — returns minimal
                element set for a bug fix or section change, following USES/THROWS/
                SEC tags across spec boundaries"
  override    : NEVER
  source_ref  : [SEC:Functions.1]

EXPORT PatchManager:
  type        : POLICY
  target      : spec_lifecycle
  condition   : ALWAYS
  value       : "Tracked patch lifecycle: create → list → absorb/reject. Patches are
                temporary amendments absorbed at version bumps."
  override    : WITH_JUSTIFICATION
  source_ref  : [SEC:Functions.2]

EXPORT SpecValidator:
  type        : CONSTRAINT
  target      : spec_integrity
  condition   : ALWAYS
  value       : "Structural validation: dangling references, orphaned elements, missing
                sections, untested sections, broken cross-spec imports"
  override    : NEVER
  source_ref  : [SEC:Functions.3]

EXPORT GraphEngine:
  type        : CONSTRAINT
  target      : dependency_analysis
  condition   : ALWAYS
  value       : "Dependency graph queries: incoming/outgoing references, impact analysis,
                cross-spec traversal"
  override    : NEVER
  source_ref  : [SEC:Functions.4]

EXPORT NamespaceManager:
  type        : SEED_DATA
  target      : spec_organization
  condition   : ALWAYS
  value       : "Namespace registry for organizing specs by project, team, or domain"
  override    : UNRESTRICTED
  source_ref  : [SEC:Functions.5]

EXPORT SplitEngine:
  type        : POLICY
  target      : spec_decomposition
  condition   : "consumer.phase >= 2"
  value       : "Monolith spec analysis and decomposition into multiple specs with
                correct IMPORT declarations"
  override    : WITH_JUSTIFICATION
  source_ref  : [SEC:Functions.6]

EXPORT SubstrateEngine:
  type        : CONSTRAINT
  target      : spec_topology
  condition   : ALWAYS
  value       : "3D substrate detection and querying — infers substrate branching from
                DERIVES FROM graph, resolves linear spec chains for agent consumption,
                builds in-memory substrate graph with caching"
  override    : NEVER
  source_ref  : [SEC:Functions.8]

EXPORT S3StorageBackend:
  type        : POLICY
  target      : spec_storage
  condition   : "config.storage.backend == 's3'"
  value       : "Spec files stored in S3 as source of truth, with on-the-fly graph
                building in memory and S3 event notifications for cache invalidation"
  override    : WITH_JUSTIFICATION
  source_ref  : [SEC:Config]

EXPORT ExtendedTools:
  type        : SEED_DATA
  target      : mcp_server
  condition   : ALWAYS
  value       : "10 MCP tools: nlspec_import, nlspec_namespaces, nlspec_slice,
                nlspec_validate, nlspec_graph, nlspec_split, nlspec_patch_create,
                nlspec_patch_list, nlspec_patch_absorb, nlspec_drift,
                nlspec_layer_stack, nlspec_layer_validate,
                nlspec_substrate_query, nlspec_substrate_graph"
  override    : NEVER
  source_ref  : [SEC:API]

### EXPECTS

EXPECTS SpecParser:
  from        : nlspec/bootstrap
  type        : CONSTRAINT
  description : "Ability to parse spec markdown into structured sections and elements"
  required    : true
  fallback    : none

EXPECTS SpecStore:
  from        : nlspec/bootstrap
  type        : CONSTRAINT
  description : "CRUD operations on spec elements with persistence"
  required    : true
  fallback    : none

EXPECTS QueryEngine:
  from        : nlspec/bootstrap
  type        : CONSTRAINT
  description : "Text and structural search across spec elements"
  required    : true
  fallback    : none

EXPECTS CRUDTools:
  from        : nlspec/bootstrap
  type        : SEED_DATA
  description : "Base 8 MCP tools that this spec extends"
  required    : true
  fallback    : none

### CONFLICT RESOLUTION

Standard rules from NLSPEC-TEMPLATE Contracts section apply.
```

---

## Appendix A: Glossary

IMPORT glossary from nlspec/bootstrap Appendix A

Additional terms:
```
Namespace:       Organizational grouping for specs (e.g., "nlspec", "myproject", "org/team")
Context Slice:   Minimal set of spec elements needed for a specific bug fix or change
Patch:           Tracked change to a spec with lifecycle (ACTIVE -> ABSORBED/REJECTED)
Self-Bootstrap:  The system importing and managing its own specification files
Split:           Decomposing a monolith spec into multiple specs with correct imports
Phase:           Growth stage (0: one spec, 1: structured access, 2: multi-spec, 3: org-scale)
Substrate:       A complete layer stack branching from a shared ancestor — represents a
                 platform variant (spatial), version (temporal), or feature flag (feature)
Branching Point: A spec in the DERIVES FROM graph with 2+ children, creating substrate divergence
Substrate Chain: A linear L1→L4 spec path through a specific substrate — what an agent consumes
S3+OTF:          Storage model where spec files live in S3 (source of truth) and the MCP server
                 builds graphs on-the-fly in memory (derived index), invalidated by S3 events
```

---

## Appendix B: Cross-Spec References

This spec imports from:
- `nlspec/bootstrap` — Spec, Section, SpecElement, Reference, SpecMetadata, SearchResult,
  SpecImport, ElementType, ReferenceType, all parser/store/query functions, 8 MCP tools,
  error model, configuration, dependencies

This spec is imported by:
- (none yet — future specs like nlspec/dashboard could import from here)

---

## Appendix C: Revision History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Feb 2026 | Initial spec. Context slicing, patch management, validation, graph, namespaces, nlspec_import, nlspec_split. |
