# nlspec MCP Server — Natural Language Specification

> **Version:** 0.1.0
> **Author:** Divyendu Deepak Singh
> **Date:** February 2026
> **License:** CC-BY-4.0

> **IMPORT:** This spec imports from `nlspec/bootstrap` (specs/bootstrap-spec.md).
> All RECORDs, ENUMs, FUNCTIONs, and MCP TOOLs defined in bootstrap are available
> here without redefinition. This spec EXTENDS the bootstrap system.

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Problem Statement](#2-problem-statement)
3. [Architecture Overview](#3-architecture-overview)
4. [Data Model](#4-data-model)
5. [Core Functions](#5-core-functions)
6. [API Surface — MCP Tools](#6-api-surface--mcp-tools)
7. [Error Model](#7-error-model)
8. [Configuration](#8-configuration)
9. [Deployment Artifacts](#9-deployment-artifacts)
10. [Scenarios](#10-scenarios)
11. [Dependencies](#11-dependencies)
12. [File Structure](#12-file-structure)
13. [Maintenance Workflow](#13-maintenance-workflow)
14. [Build and Run](#14-build-and-run)
15. [Boundaries](#15-boundaries)

---

## 1. Abstract

The nlspec MCP Server extends the bootstrap system (parser, store, query engine, 8 CRUD
tools) with context slicing, patch management, spec validation, graph operations,
namespaces, spec decomposition, and the `nlspec_import` tool.

This spec supports the phase transitions described in NLSPEC-SYSTEM.md:
- **Phase 1 → 2:** `nlspec_split` decomposes a monolith spec into N specs with
  correct imports, helping you transition from single-spec to multi-spec.
- **Phase 2 → 3:** Namespaces, validation, and patch management scale to
  organizational complexity.
- **Self-bootstrap:** Once running, the system imports and manages its own specs
  using the same tools it provides.

---

## 2. Problem Statement

### Current State
The bootstrap system provides CRUD + search on spec elements. An agent can read, write,
and query elements in any loaded spec.

### Deficiency
Four capabilities are missing for production use:

1. **Context slicing** — the agent reads full sections when fixing bugs. It should read
   only the minimal set of elements reachable from the failing scenario.
2. **Patch management** — there is no structured way to track bug fixes. Patches are
   ad-hoc changes with no lifecycle, no categorization, no absorption workflow.
3. **Validation** — there is no way to check a spec's structural integrity (dangling
   references, orphaned records, untested sections).
4. **Namespace & import** — specs are loaded by file path. There is no namespace system
   for organizing specs across projects, and no tool for importing specs into the
   running system.

### Target State
A complete MCP server that supports the full spec lifecycle: write → validate → implement
→ fix (with context slicing) → patch → absorb → evolve. Specs live in namespaces and
are imported via `nlspec_import`. The system manages its own specs the same way it
manages any other project's specs.

### Key Insight
Once you have `nlspec_import`, the system bootstraps itself. Import the bootstrap spec
and this spec into the running server. From that point, every change to nlspec itself
goes through the same tools: `nlspec_slice` to understand the impact, `nlspec_patch_create`
to track the fix, `nlspec_validate` to verify integrity, `nlspec_patch_absorb` to merge
it back.

---

## 3. Architecture Overview

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
|  | Tools (8) |  | Tools (7)  |  | Manager     |           |
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
|    +----------+-----------+                                |
|               |                                            |
|    +----------v-----------+                                |
|    |  Persistence         |                                |
|    |  - .md files (truth) |                                |
|    |  - SQLite (index)    |                                |
|    |  - patches/ dir      |                                |
|    +----------------------+                                |
+-----------------------------------------------------------+
```

### 3.1 Component Inventory (additions to bootstrap)

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

### Component: Namespace Manager
- **Responsibility:** Organize specs into namespaces. Resolve namespace-qualified
  identifiers. Handle `nlspec_import`.
- **Owns:** Namespace registry, namespace-to-path mapping
- **Calls:** Spec Store, Spec Parser
- **Called by:** Core Engine, all MCP tools
- **Lifecycle:** Persistent across sessions

### 3.2 Data Flows (additions to bootstrap)

### Flow: Agent requests a context slice for a bug fix
1. Agent calls `nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})`
2. MCP Server routes to Context Slicer
3. Context Slicer reads SCENARIO 7, extracts [SEC:5.3] [SEC:4.1] tags
4. Context Slicer reads Section 5.3, extracts USES: QueryRequest, QueryResponse
5. Context Slicer reads Section 4 for QueryRequest, QueryResponse RECORDs
6. Context Slicer reads Section 5.3 THROWS, pulls relevant errors from Section 7
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

## 4. Data Model

IMPORT Spec, Section, SpecElement, Reference, SpecMetadata, SearchResult,
       SpecImport, ElementType, ReferenceType FROM nlspec/bootstrap Section 4

### 4.1 Additional Records

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
  trigger      : String              -- what caused the slice (e.g., "SCENARIO 7", "Section 5.3")
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
  sections     : List<String>        -- affected section numbers
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
  errors       : List<ValidationError>
  warnings     : List<ValidationWarning>
  is_valid     : bool                -- true if zero errors (warnings are OK)

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

### 4.2 Additional Enumerations

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

### 4.3 Namespace-Qualified Identifiers

All spec identifiers become namespace-qualified:

```
Full element ID:   {namespace}/{spec_id}:{section}:{type}:{name}
Example:           myproject/auth:5.3:function:validate_token

Short spec ID:     {namespace}/{spec_id}
Example:           nlspec/bootstrap

Namespace only:    {namespace}
Example:           myproject
```

When namespace is omitted, it defaults to the "default" namespace.
Bootstrap tools (nlspec_get, nlspec_search, etc.) accept an optional `namespace`
parameter. Existing behavior is preserved when namespace is not specified.

---

## 5. Core Functions

### 5.1 Context Slicer

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
  6. Order by: section number ascending, then element position
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
FUNCTION slice_for_section(namespace: String, spec_id: SpecId, section: SectionNumber) -> ContextSlice
  USES: ContextSlice
  THROWS: NotFound

  BEHAVIOR:
  1. Read all elements in the given section
  2. For each element, follow outgoing references (USES, THROWS, IMPORTS)
  3. Find all SCENARIOs tagged with this section
  4. Assemble and return ContextSlice
```

### 5.2 Patch Manager

```
FUNCTION patch_create(namespace: String, spec_id: SpecId, category: PatchCategory, sections: List<SectionNumber>, scenarios: List<u64>, description: String, content: String) -> Patch
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

### 5.3 Spec Validator

```
FUNCTION validate_spec(namespace: String, spec_id: SpecId) -> ValidationResult
  USES: ValidationResult, ValidationError, ValidationWarning
  THROWS: NotFound

  BEHAVIOR:
  1. Check all required sections exist (1-15 per nlspec template standard)
  2. Check all References resolve:
     a. Every USES target exists as a RECORD or ENUM
     b. Every THROWS target exists in the Error Model section
     c. Every IMPORT target exists in the referenced spec (may cross namespaces)
     d. Every [SEC:x.x] tag references an existing section
  3. Check symmetry: if A USES B, B should have USED BY A
  4. Check all SCENARIOs have at least one [SEC:] tag
  5. Check all FUNCTIONs have USES and THROWS declarations
  6. Produce warnings for:
     a. RECORDs with no USED BY (orphaned records)
     b. Sections with no SCENARIOs (untested sections)
     c. FUNCTIONs with no PERFORMANCE targets
  7. Return ValidationResult

  PERFORMANCE:
  - Target: < 500ms for a project with 10 specs
```

### 5.4 Graph Engine

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

### 5.5 Namespace Manager

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

### 5.6 Spec Decomposition

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
  sections     : List<String>         -- section numbers in this cluster
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
     - "by_section": group by top-level section number (Sections 4-5 become
       one spec, Section 6 becomes another, etc.)
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
     d. Generate Section 15 (Boundaries) noting the split origin
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

---

## 6. API Surface — MCP Tools

Eight additional MCP tools. Each takes JSON parameters, returns JSON results.
All bootstrap tools (nlspec_init, nlspec_get, nlspec_list, nlspec_search,
nlspec_create, nlspec_update, nlspec_delete) remain available and now accept
an optional `namespace` parameter.

### 6.1 Import & Namespaces

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

### 6.2 Context Slicing

```
MCP TOOL: nlspec_slice
  DESCRIPTION: Extract minimal context for a bug fix. Given a scenario or section,
  trace dependency edges and return only what the agent needs.

  PARAMETERS:
    namespace    : String              -- spec namespace
    spec_id      : SpecId              -- which spec
    scenario     : u64 | None          -- slice for this scenario number
    section      : SectionNumber | None -- slice for this section
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
          trace: ["SCENARIO 7 -> [SEC:5.3] -> FUNCTION validate_token -> USES TokenRecord -> ..."]
        }
      }
```

### 6.3 Patch Management

```
MCP TOOL: nlspec_patch_create
  DESCRIPTION: Create a new patch for a spec.

  PARAMETERS:
    namespace    : String
    spec_id      : SpecId
    category     : PatchCategory
    sections     : List<SectionNumber>
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

### 6.4 Validation

```
MCP TOOL: nlspec_validate
  DESCRIPTION: Validate a spec's structural integrity.

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
          is_valid: false,
          errors: [{element_id: "myproject/auth:5.3:function:validate_token",
                    error_type: "dangling_reference",
                    message: "USES TokenRecord but no RECORD named TokenRecord found"}],
          warnings: [{element_id: "myproject/auth:4.2:record:SessionStats",
                     warning_type: "orphaned",
                     message: "RECORD SessionStats has no USED BY references"}]
        }
      }
```

### 6.5 Graph Operations

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

### 6.6 Spec Decomposition

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
            {name: "auth", sections: ["4.1", "4.2", "5.1", "5.2"], element_count: 47,
             rationale: "Dense references between auth records and auth functions"},
            {name: "storage", sections: ["4.3", "5.3", "5.4"], element_count: 31,
             rationale: "Storage records and functions form isolated subgraph"},
            {name: "api", sections: ["6.1", "6.2", "6.3"], element_count: 22,
             rationale: "API endpoints reference auth and storage but are self-contained"}
          ],
          cross_refs: [
            {from: "auth:5.1:function:validate_token", to: "storage:4.3:record:Session"},
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

---

## 7. Error Model

IMPORT NlspecError, ParseError, NotFound, IOError FROM nlspec/bootstrap Section 7

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

## 8. Configuration

IMPORT all config from nlspec/bootstrap Section 8

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
```

---

## 9. Deployment Artifacts

### 9.1 Distribution

```
Distribution: npm package @nlspec/server
Installation: npm install -g @nlspec/server  OR  npx @nlspec/server

The package includes both bootstrap and MCP server capabilities.
The bootstrap tools are always available. The advanced tools (slice, patch,
validate, graph, import, namespaces) are part of the same server binary.
```

### 9.2 MCP Configuration

```json
{
  "mcpServers": {
    "nlspec": {
      "command": "npx",
      "args": ["@nlspec/server", "--specs-dir", "./specs"]
    }
  }
}
```

---

## 10. Scenarios

### 10.1 Self-Bootstrap

```
SCENARIO 1: System imports its own specs                            [SEC:6.1] [SMOKE]
  GIVEN: nlspec MCP server is running with no specs loaded
  WHEN: Agent calls nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})
  AND: Agent calls nlspec_import({namespace: "nlspec", spec_id: "mcp-server", path: "specs/mcp-server-spec.md"})
  THEN:
  - nlspec_namespaces returns [{name: "nlspec", specs: ["bootstrap", "mcp-server"]}]
  - nlspec_get({namespace: "nlspec", spec_id: "bootstrap", section: "4.1"}) returns Spec record
  - nlspec_search({namespace: "nlspec", query: "ContextSlice"}) finds the record in mcp-server spec
```

```
SCENARIO 2: System validates its own specs                          [SEC:6.4] [SMOKE]
  GIVEN: Both nlspec specs are imported (SCENARIO 1 complete)
  WHEN: Agent calls nlspec_validate({namespace: "nlspec", spec_id: "bootstrap"})
  THEN: result.is_valid is true, zero errors
  WHEN: Agent calls nlspec_validate({namespace: "nlspec", spec_id: "mcp-server"})
  THEN: result.is_valid is true, zero errors
```

### 10.2 Context Slicing

```
SCENARIO 3: Slice for a scenario                                    [SEC:6.2] [SEC:5.1]
  GIVEN: A spec "myproject/auth" is imported with:
    - Section 4.1: RECORD TokenRecord
    - Section 5.3: FUNCTION validate_token (USES: TokenRecord, THROWS: AuthError)
    - Section 7: AuthError definition
    - Section 10: SCENARIO 7 tagged [SEC:5.3]
  WHEN: Agent calls nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  THEN:
  - slice.trigger is "SCENARIO 7"
  - slice.elements contains: SCENARIO 7, FUNCTION validate_token, RECORD TokenRecord, AuthError
  - slice.total_lines is between 50 and 500
  - slice.trace contains a human-readable dependency chain
```

```
SCENARIO 4: Slice follows cross-namespace imports                   [SEC:6.2] [SEC:5.1]
  GIVEN: "myproject/auth" has IMPORT UserRecord FROM myproject/users Section 4.1
  AND: "myproject/users" is imported with Section 4.1 containing RECORD UserRecord
  WHEN: Agent calls nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  AND: SCENARIO 7's dependency chain reaches IMPORT UserRecord
  THEN: slice.elements includes UserRecord from myproject/users
  AND: slice.specs_touched includes "myproject/auth" and "myproject/users"
```

### 10.3 Patch Management

```
SCENARIO 5: Patch lifecycle                                         [SEC:6.3] [SEC:5.2]
  GIVEN: "myproject/auth" spec is imported, version "0.1.0"
  WHEN: Agent calls nlspec_patch_create({
    namespace: "myproject", spec_id: "auth",
    category: "IMPL_ERROR", sections: ["5.3"], scenarios: [7],
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

### 10.4 Validation

```
SCENARIO 6: Detect dangling reference                               [SEC:6.4] [SEC:5.3]
  GIVEN: A spec where FUNCTION foo USES: BarRecord, but no RECORD BarRecord exists
  WHEN: Agent calls nlspec_validate for that spec
  THEN: result.is_valid is false
  AND: errors contains {error_type: "dangling_reference", message contains "BarRecord"}
```

```
SCENARIO 7: Detect orphaned record                                  [SEC:6.4] [SEC:5.3]
  GIVEN: A spec where RECORD OrphanedThing exists but no FUNCTION has USES: OrphanedThing
  WHEN: Agent calls nlspec_validate for that spec
  THEN: warnings contains {warning_type: "orphaned", message contains "OrphanedThing"}
  AND: result.is_valid is true (warnings don't fail validation)
```

### 10.5 Graph Operations

```
SCENARIO 8: Graph shows incoming dependencies                       [SEC:6.5] [SEC:5.4]
  GIVEN: RECORD Entry is USED BY: storage_get, storage_put, storage_delete
  WHEN: Agent calls nlspec_graph({element_id: "myproject/kv:4.1:record:Entry", direction: "incoming"})
  THEN: nodes includes storage_get, storage_put, storage_delete
  AND: edges show USED_BY relationships from each function to Entry
```

### 10.6 Namespace Operations

```
SCENARIO 9: Multiple namespaces coexist                             [SEC:6.1]
  GIVEN: Specs imported under "nlspec" and "myproject" namespaces
  WHEN: Agent calls nlspec_search({query: "Spec"})
  THEN: Results include elements from BOTH namespaces
  WHEN: Agent calls nlspec_search({namespace: "nlspec", query: "Spec"})
  THEN: Results include ONLY elements from "nlspec" namespace
```

```
SCENARIO 10: Re-import refreshes index                              [SEC:6.1]
  GIVEN: "myproject/auth" is imported
  WHEN: Human edits the markdown file (adds a new RECORD)
  AND: Agent calls nlspec_import with the same namespace/spec_id/path
  THEN: The new RECORD appears in nlspec_search results
  AND: The old elements that still exist are preserved
```

### 10.7 Spec Decomposition

```
SCENARIO 11: Suggest decomposition of a monolith spec               [SEC:6.6] [SEC:5.6]
  GIVEN: "myproject/monolith" is imported with 300+ elements across 15 sections
  AND: Sections 4.1-4.2, 5.1-5.2 have dense internal references (auth cluster)
  AND: Sections 4.3, 5.3-5.4 have dense internal references (storage cluster)
  AND: Sections 6.1-6.3 reference both clusters (api cluster)
  WHEN: Agent calls nlspec_split({namespace: "myproject", spec_id: "monolith",
                                   strategy: "cluster", execute: false})
  THEN: suggestion.clusters has 3 entries
  AND: Each cluster has a name, sections, element_count, and rationale
  AND: cross_refs identifies references that will become IMPORTs
  AND: No files are created or modified (suggestion only)
```

```
SCENARIO 12: Execute decomposition creates new specs                 [SEC:6.6] [SEC:5.6]
  GIVEN: SCENARIO 11 suggestion has been reviewed and approved
  WHEN: Agent calls nlspec_split({namespace: "myproject", spec_id: "monolith",
                                   strategy: "cluster", execute: true})
  THEN: Three new spec files are created
  AND: Each new spec has correct IMPORT declarations for cross-cluster references
  AND: Each new spec follows the 15-section template
  AND: Original monolith spec is preserved (not deleted)
  AND: nlspec_validate passes for all three new specs
```

```
SCENARIO 13: Split preserves scenarios across clusters               [SEC:6.6] [SEC:5.6]
  GIVEN: Original spec has SCENARIO 7 tagged [SEC:5.1] [SEC:5.3]
  AND: Section 5.1 is in the "auth" cluster, Section 5.3 is in the "storage" cluster
  WHEN: Split is executed
  THEN: SCENARIO 7 appears in BOTH the auth spec and the storage spec
  AND: Each copy has appropriate [SEC:] tags for its own sections
```

---

## 11. Dependencies

IMPORT dependencies from nlspec/bootstrap Section 11

Additional: none. The MCP server spec uses only what bootstrap already provides
(Node.js, @modelcontextprotocol/sdk, better-sqlite3, TypeScript).

---

## 12. File Structure

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

## 13. Maintenance Workflow

Same as bootstrap (Section 13) with one addition:

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

## 14. Build and Run

### Prerequisites
Same as bootstrap: Node.js >= 20, npm, TypeScript

### Build
```bash
npm run build
```

### Test
```bash
npm test                              # all scenarios (bootstrap + mcp-server)
npm test -- --filter "SCENARIO 1"     # specific scenario
```

### Run as MCP Server
```bash
npx @nlspec/server --specs-dir ./specs
```

### Self-Bootstrap Sequence
```bash
# Start the server
npx @nlspec/server --specs-dir ./specs

# In an MCP client (Claude Code, Cursor, etc.):
nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "mcp-server", path: "specs/mcp-server-spec.md"})

# System now manages itself. Verify:
nlspec_validate({namespace: "nlspec", spec_id: "bootstrap"})
nlspec_validate({namespace: "nlspec", spec_id: "mcp-server"})
```

---

## 15. Boundaries

### This Spec Adds:
- Context slicing (minimal context for bug fixes)
- Patch management (create, list, absorb lifecycle)
- Spec validation (structural integrity checks)
- Graph operations (dependency traversal)
- Namespaces (organize specs across projects)
- nlspec_import (bring specs into the system)
- nlspec_split (decompose monolith specs into multiple specs)
- Self-management (system manages its own specs)
- Phase transition support (Phase 0 → 1 → 2 → 3)

### This Spec Does NOT:
- Redefine anything from bootstrap — it only extends
- Implement code generation — agent's job, not nlspec's
- Execute tests — agent runs tests using test runners
- Enforce specific project structure — namespaces are freeform
- Require all tools — bootstrap tools work standalone without these extensions
- Prevent namespace cycles — warns about them, like spec import cycles

### Future Extensions (not in this spec):
- nlspec_describe: MCP App rendering interactive spec overview (dashboard)
- nlspec_visualize: MCP App rendering interactive diagrams (D3.js force-directed)
- nlspec_diff: Compare two versions of a spec
- nlspec_migrate: Upgrade a spec to a newer template version
- Spec versioning beyond semver (branching, merging)
- Remote spec registries (pull specs from URLs or package registries)

---

## Appendix A: Glossary

IMPORT glossary from nlspec/bootstrap Appendix A

Additional terms:
```
Namespace:      Organizational grouping for specs (e.g., "nlspec", "myproject", "org/team")
Context Slice:  Minimal set of spec elements needed for a specific bug fix or change
Patch:          Tracked change to a spec with lifecycle (ACTIVE -> ABSORBED/REJECTED)
Self-Bootstrap: The system importing and managing its own specification files
Split:          Decomposing a monolith spec into multiple specs with correct imports
Phase:          Growth stage (0: one spec, 1: structured access, 2: multi-spec, 3: org-scale)
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
