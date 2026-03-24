# nlspec Server — Layer 1: Contracts & Interfaces

> **Version:** 1.0.0
> **Author:** Divyendu Deepak Singh
> **Date:** March 2026
> **Type:** COMPONENT
> **Language Target:** Python
> **Status:** Draft
> **Layer:** 1-Specification
> **License:** CC-BY-4.0

---

## Table of Contents

1. [Abstract](#abstract)
2. [Problem Statement](#problem-statement)
3. [Architecture Overview](#architecture-overview)
4. [Data Model](#datamodel)
5. [Core Functions](#functions)
6. [API Surface — MCP Tools](#api--mcp-tools)
7. [Error Model](#errors)
8. [Contracts](#contracts)
9. [Component Inventory](#component-inventory)

---

## Abstract

**nlspec** is the core MCP server that parses nlspec markdown files into structured, queryable elements and exposes CRUD + search operations as MCP tools. An AI coding agent connects via MCP and can read, create, update, delete, and search spec elements without parsing raw markdown itself.

The system spans two phases:

**Phase 1-Bootstrap (Core):** The minimal system that provides the foundation:
- **Parser:** `parse_spec` → markdown → structured elements with gap analysis
- **Store:** CRUD operations on specs and elements (PostgreSQL or SQLite)
- **Query Engine:** Full-text search and structural (reference-based) search
- **8 MCP Tools:** `nlspec_init`, `nlspec_get`, `nlspec_list`, `nlspec_search`, `nlspec_create`, `nlspec_update`, `nlspec_delete`, plus `nlspec_query`
- **Single-project mode:** All specs are in the default project (Phase 1)

**Phase 2-Extended (Advanced):** Extends bootstrap with enterprise capabilities:
- **Namespaces:** Multi-project support
- **Context Slicing:** Extract minimal context for a specific problem
- **Patch Management:** Track and absorb spec improvements
- **Spec Validation:** Detect structural issues (dangling refs, orphaned elements, broken imports)
- **Graph Operations:** Understand spec relationships and dependencies
- **Substrate Detection:** Identify spatial/temporal/feature branching
- **Spec Decomposition:** Suggest and execute splitting of monolith specs into multi-spec systems
- **Drift Detection:** Compare spec against code
- **gRPC Interface:** Alternative to MCP (port 50060) for internal tools
- **PostgreSQL Shared Database:** Support multi-instance deployments
- **14 additional MCP Tools:** Import, namespaces, slice, patch (create/list/absorb), diff, validate, graph, split, drift, layer, substrate query, substrate graph
- **21 total tools** (8 Phase 1 + 14 Phase 2)

This spec defines all records, functions, tools, and errors for both phases in a single unified contract.

---

## Problem Statement

### Current State

AI coding agents receive specs as raw markdown. The agent must manually parse section headers, follow cross-references, and re-read files on every interaction. There is no structured access. "Find all FUNCTIONs that reference Entry" requires the agent to grep or scan the entire file.

### Deficiency

No structured CRUD. No search. No indexing. Every agent interaction re-parses the same markdown. Cross-spec references (`IMPORT X FROM other-spec.md DataModel.2`) require the agent to open files and navigate manually. Single-project operation doesn't scale to organizational complexity.

### Target State

An MCP server that parses nlspec markdown into indexed, queryable elements. The agent calls `nlspec_get` to read a FUNCTION, `nlspec_search` to find elements by name, type, or reference, and `nlspec_create` to add a new SCENARIO — all via native MCP tool calls. The system scales to multi-project organizations with namespaces, advanced validation, context slicing, and decomposition.

### Key Insight

nlspec markdown has implicit structure: RECORDs reference FUNCTIONs via USED BY, FUNCTIONs declare USES and THROWS, SCENARIOs tag sections with [SEC:x.x]. Parsing this structure into an indexed store turns a flat document into a queryable database while keeping human-readable markdown as the source of truth.

---

## Architecture Overview

```
+----------------------------+  +----------------------------+
|       MCP Clients          |  |    gRPC Clients           |
| (Claude Code, Cursor,      |  |    (internal consumers)   |
|  Claude Desktop)           |  |                            |
+-------------+--------------+  +-------------+--------------+
              |                               |
              | MCP (stdio / SSE)             | gRPC (:50060)
              v                               v
+-----------------------------------------------------------+
|                   nlspec MCP Server                        |
|                                                            |
|  +-------------------+  +-------------------+             |
|  |  MCP Tool Layer   |  |  gRPC Service     |             |
|  |  (20 tools)       |  |  Layer            |             |
|  +---------+---------+  +---------+---------+             |
|            |                      |                        |
|            +----------+-----------+                        |
|                       |                                    |
|            +----------v-----------+                        |
|            |    Core Engine       |                        |
|            |                      |                        |
|            |  +----------------+  |                        |
|            |  | Spec Parser    |  | Markdown → Elements    |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  | Spec Store     |  | CRUD on Elements       |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  | Query Engine   |  | Search + Refs          |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Namespace Mgr   |  | Multi-project          |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Context Slicer  |  | Extract minimal context |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Patch Manager   |  | Track improvements     |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Spec Validator  |  | Validate structure     |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Graph Engine    |  | Build dependency graph |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Substrate Mgr   |  | Detect branching       |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Spec Splitter   |  | Decompose specs        |
|            |  +----------------+  |                        |
|            |  +----------------+  |                        |
|            |  |Drift Detector  |  | Compare code vs spec   |
|            |  +----------------+  |                        |
|            +-------------------+                          |
|                       |                                    |
|                       v                                    |
|            +-------------------+                          |
|            |   PostgreSQL      |                          |
|            |   (Shared DB)     |                          |
|            +-------------------+                          |
|                       |                                    |
|            +-------------------+                          |
|            |    Git Version    |                          |
|            |    Control        |                          |
|            +-------------------+                          |
+-----------------------------------------------------------+
```

**Transport:**
- **Phase 1:** MCP stdio only — the MCP server runs as a subprocess spawned by Claude Code or other agent
- **Phase 2:** MCP (stdio + SSE) and gRPC (:50060) — supports local and cloud deployment, internal services

---

## DataModel

### RECORD: Spec

Represents a single nlspec markdown file after parsing.

```
RECORD Spec
  FIELDS:
    id: String                      -- spec identifier (e.g., "kv-store", "auth")
    namespace: String               -- project namespace (Phase 1: "default", Phase 2: user-defined)
    name: String                    -- human-readable name
    path: String                    -- markdown file path (relative to project_dir)
    version: String                 -- semantic version (e.g., "0.1.0")
    status: SpecStatus              -- "Draft", "Stable", "Deprecated"
    abstract: String                -- summary text from spec Abstract section
    sections: Section[]             -- list of parsed sections
    element_count: u64              -- total elements (RECORD, FUNCTION, SCENARIO, etc.)
    created_at: Timestamp           -- when indexed
    updated_at: Timestamp           -- when last modified
    gaps: Gap[]                     -- identified gaps from parse_spec analysis
    layer: u64 | None               -- layer number (1, 2, 3, 4) if declared
    derives_from: String | None     -- parent spec_id if DERIVES FROM declared

  USED BY: store_get, store_list, nlspec_get, nlspec_list, nlspec_init, nlspec_import
  USES: Section, Gap
```

### RECORD: Section

A logical section within a spec (e.g., "DataModel.1", "Functions.2", "Scenarios").

```
RECORD Section
  FIELDS:
    spec_id: String                 -- parent spec
    name: String                    -- section name (e.g., "DataModel.1")
    title: String                   -- section header (e.g., "Data Model — Part 1")
    element_type: ElementType       -- what this section contains (RECORD, FUNCTION, SCENARIO, etc.)
    elements: SpecElement[]         -- list of elements in this section
    markdown: String                -- raw markdown of this section

  USED BY: Spec, store_get, nlspec_get
  USES: SpecElement, ElementType
```

### RECORD: SpecElement

An individual parsed element (FUNCTION, RECORD, SCENARIO, CONFIG, ERROR, ENUM).

```
RECORD SpecElement
  FIELDS:
    id: String                      -- unique element ID within spec (e.g., "kv-store:DataModel.1:record:Entry")
    spec_id: String                 -- parent spec
    section: String                 -- parent section name
    name: String                    -- element name (e.g., "storage_get", "Entry")
    element_type: ElementType       -- RECORD, FUNCTION, SCENARIO, CONFIG, ERROR, ENUM
    content: String                 -- full markdown body
    references: Reference[]         -- outgoing references (USES, THROWS, SEC tags, IMPORT)
    tags: String[]                  -- semantic tags (e.g., ["SEC:Functions.3", "SMOKE"])
    created_at: Timestamp           -- when created
    updated_at: Timestamp           -- when last modified

  USED BY: Section, query_search, query_references, nlspec_get, nlspec_create, nlspec_update, context_slicer
  USES: ElementType, Reference
```

### RECORD: Reference

Represents a declared relationship from one element to another.

```
RECORD Reference
  FIELDS:
    id: String                      -- unique ID
    source_element_id: String       -- where the reference originates
    target_name: String             -- name being referenced (e.g., "Entry", "AuthError", "UserRecord")
    target_spec_id: String          -- spec containing target (Phase 1: same spec; Phase 2: can be cross-namespace)
    target_namespace: String | None -- target namespace (Phase 2 only)
    reference_type: ReferenceType   -- USES, THROWS, USED_BY, SEC, IMPORT, CALLS
    resolved: bool                  -- true if target element exists

  USED BY: SpecElement, query_references, nlspec_search, context_slicer
  USES: ReferenceType
```

### RECORD: SpecMetadata

Metadata extracted from spec header.

```
RECORD SpecMetadata
  FIELDS:
    spec_id: String
    version: String                 -- from header
    author: String                  -- from header
    date: String                    -- publication date
    type: String                    -- "SYSTEM", "COMPONENT", "LIBRARY"
    language_target: String         -- "Python", "Go", "Rust", etc.
    status: String                  -- "Draft", "Stable", "Deprecated"
    layer: u64 | None               -- layer number (1-4)

  USED BY: Spec
```

### RECORD: SearchResult

Returned from `nlspec_search`.

```
RECORD SearchResult
  FIELDS:
    element_id: String              -- full element ID
    spec_id: String
    namespace: String               -- which namespace (Phase 2)
    section: String
    name: String
    element_type: ElementType
    match_reason: String            -- why it matched (e.g., "text contains 'TTL'", "USES Entry")
    relevance: float                -- 0.0–1.0 relevance score
    snippet: String                 -- truncated content excerpt with highlights

  USED BY: query_search, nlspec_search
```

### RECORD: GapReport

Result of gap analysis from `parse_spec` or `analyze_gaps`.

```
RECORD GapReport
  FIELDS:
    spec_id: String
    gaps: Gap[]                     -- identified issues
    total_critical: u64             -- count of CRITICAL gaps
    total_warning: u64              -- count of WARNING gaps
    total_info: u64                 -- count of INFO gaps

  USED BY: Spec
```

### RECORD: Gap

An identified issue in a spec (missing section, dangling reference, etc.).

```
RECORD Gap
  FIELDS:
    id: String                      -- unique gap ID
    spec_id: String
    severity: GapSeverity           -- CRITICAL, WARNING, INFO
    category: String                -- "dangling_reference", "orphaned_element", "undeclared_section", etc.
    element_id: String | None       -- affected element, if applicable
    message: String                 -- human-readable description
    suggestion: String              -- optional fix suggestion

  USED BY: GapReport
  USES: GapSeverity
```

### RECORD: Namespace

Represents a project or organizational unit (Phase 2).

```
RECORD Namespace
  FIELDS:
    name: String                    -- namespace identifier (e.g., "myproject", "nlspec")
    description: String             -- human-readable description
    specs: Spec[]                   -- specs in this namespace
    created_at: Timestamp
    updated_at: Timestamp

  USED BY: nlspec_namespaces, nlspec_import, namespace_manager
```

### RECORD: ContextSlice

Result of context slicing operation (Phase 2).

```
RECORD ContextSlice
  FIELDS:
    namespace: String               -- source namespace
    spec_id: String                 -- primary spec
    trigger: String                 -- what triggered the slice (scenario ID, function name, etc.)
    elements: SpecElement[]         -- collected elements
    specs_touched: String[]         -- list of specs that were traversed
    total_lines: u64                -- estimated markdown lines in slice
    trace: String                   -- human-readable dependency chain
    depth: u64                      -- traversal depth actually used

  USED BY: nlspec_slice
  USES: SpecElement
```

### RECORD: Patch

Represents a suggested or applied improvement to a spec (Phase 2).

```
RECORD Patch
  FIELDS:
    id: String                      -- unique patch ID (e.g., "PATCH-001")
    namespace: String               -- target namespace
    spec_id: String                 -- target spec
    category: PatchCategory         -- A (deficiency), B (error), C (missing capability)
    sections: String[]              -- affected sections (e.g., ["Functions.3"])
    scenarios: u64[]                -- affected scenario IDs
    description: String             -- why this patch exists
    content: String                 -- patch text (proposed spec change)
    status: PatchStatus             -- ACTIVE, ABSORBED, REJECTED
    created_at: Timestamp
    absorbed_at: Timestamp | None   -- when patch was merged
    version_before: String          -- spec version before absorption
    version_after: String           -- spec version after absorption

  USED BY: nlspec_patch_create, nlspec_patch_list, nlspec_patch_absorb, nlspec_diff_since
  USES: PatchCategory, PatchStatus
```

### RECORD: StructuredDiff

Represents the structured difference between two versions of a spec, at element granularity with sub-element classification (Phase 2).

```
RECORD StructuredDiff
  FIELDS:
    spec_id: String                 -- which spec was diffed
    namespace: String               -- namespace of the spec
    from_version: String            -- starting version (e.g., "0.1.0")
    to_version: String              -- ending version (e.g., "0.2.0")
    changes: ElementChange[]        -- list of element-level changes
    patches_included: String[]      -- patch IDs absorbed between from_version and to_version
    diffed_at: Timestamp

  USED BY: nlspec_diff_since, diff_since
  USES: ElementChange

  INVARIANTS:
  - from_version < to_version (semver ordering)
  - changes is never empty (if versions differ, something changed)
  - patches_included lists only ABSORBED patches between the two versions
```

### RECORD: ElementChange

A single element-level change within a StructuredDiff (Phase 2).

```
RECORD ElementChange
  FIELDS:
    element_id: String              -- fully qualified: namespace/spec_id:section:type:name
    element_name: String            -- human-readable name (e.g., "validate_order")
    element_type: ElementType       -- RECORD, FUNCTION, SCENARIO, etc.
    section: String                 -- parent section (e.g., "Functions.3")
    change_type: ChangeType         -- MODIFIED, ADDED, REMOVED
    sub_changes: SubChange[]        -- what specifically changed within the element
    impact: ChangeImpact            -- how this affects dependent elements
    hash_before: String | None      -- content hash before (None if ADDED)
    hash_after: String | None       -- content hash after (None if REMOVED)
    patch_ids: String[]             -- which patches caused this change (may be empty for direct edits)

  USED BY: StructuredDiff, nlspec_diff_since
  USES: ElementType, ChangeType, SubChange, ChangeImpact

  INVARIANTS:
  - hash_before is None iff change_type == ADDED
  - hash_after is None iff change_type == REMOVED
  - sub_changes is not empty for MODIFIED elements
  - sub_changes is empty for ADDED and REMOVED elements
```

### ENUM: ChangeType

```
ENUM ChangeType
  MODIFIED                          -- element exists in both versions, content differs
  ADDED                             -- element is new in to_version
  REMOVED                           -- element was deleted in to_version
```

### ENUM: SubChange

Classifies what specifically changed within a MODIFIED element (Phase 2).

```
ENUM SubChange
  SIGNATURE                         -- function params or return type changed
  PRECONDITIONS                     -- preconditions added/modified/removed
  POSTCONDITIONS                    -- postconditions changed
  DATA_FIELDS                       -- record fields added/modified/removed
  INVARIANTS                        -- invariant constraints changed
  ERROR_CASES                       -- error conditions or throws changed
  BEHAVIOR                          -- function behavior steps changed (logic change)
  SCENARIOS                         -- scenario steps or assertions changed
  REFERENCES                        -- USES/THROWS/USED_BY declarations changed
  METADATA                          -- non-functional: notes, performance targets, descriptions
```

### ENUM: ChangeImpact

Classifies how a change affects elements that depend on the changed element (Phase 2).

```
ENUM ChangeImpact
  INTERFACE_BREAKING                -- callers must change (signature, required data field, removed export)
  LOGIC_ONLY                        -- same interface, different internal behavior (preconditions, behavior steps)
  ADDITIVE                          -- new capability, no breakage (new optional field, new function, new scenario)
  REMOVAL                           -- element deleted, all references must be updated
```

### RECORD: ValidationResult

Result of spec validation (Phase 2).

```
RECORD ValidationResult
  FIELDS:
    namespace: String
    spec_id: String
    is_valid: bool                  -- true iff errors.len() == 0
    errors: ValidationError[]
    warnings: ValidationWarning[]
    gaps: Gap[]                     -- from analyze_gaps
    checked_at: Timestamp

  USED BY: nlspec_validate
  USES: ValidationError, ValidationWarning, Gap
```

### RECORD: ValidationError

A validation failure that blocks correctness (Phase 2).

```
RECORD ValidationError
  FIELDS:
    error_type: String              -- "dangling_reference", "broken_import", "missing_section", etc.
    element_id: String              -- affected element, if applicable
    message: String                 -- human-readable error
    suggestion: String              -- recommended fix

  USED BY: ValidationResult
```

### RECORD: ValidationWarning

A validation issue that is not critical (Phase 2).

```
RECORD ValidationWarning
  FIELDS:
    warning_type: String            -- "orphaned_element", "undeclared_section", etc.
    element_id: String
    message: String
    suggestion: String | None

  USED BY: ValidationResult
```

### RECORD: SubstrateNode

A spec node in the substrate graph (Phase 2).

```
RECORD SubstrateNode
  FIELDS:
    spec_id: String
    layer: u64 | None               -- layer number if declared
    parent_spec_id: String | None   -- DERIVES FROM parent
    children: String[]              -- child spec IDs (DERIVES FROM this)

  USED BY: SubstrateGraph
```

### RECORD: SubstrateEdge

A directed edge between specs (parent → child) (Phase 2).

```
RECORD SubstrateEdge
  FIELDS:
    from_spec_id: String            -- parent
    to_spec_id: String              -- child
    edge_type: String               -- "derives_from", "imports", etc.

  USED BY: SubstrateGraph
```

### RECORD: SubstrateGraph

The full dependency graph for a namespace (Phase 2).

```
RECORD SubstrateGraph
  FIELDS:
    namespace: String
    nodes: SubstrateNode[]
    edges: SubstrateEdge[]
    substrates: Substrate[]         -- identified spatial/temporal/feature branches
    built_at: Timestamp
    cache_ttl_ms: u64               -- how long this graph is valid

  USED BY: nlspec_substrate_graph
  USES: SubstrateNode, SubstrateEdge, Substrate
```

### RECORD: Substrate

A detected branching point (spatial, temporal, or feature-based) (Phase 2).

```
RECORD Substrate
  FIELDS:
    id: String                      -- substrate identifier (e.g., "ios", "v2", "feature-auth")
    type: String                    -- "spatial" (platform), "temporal" (version), "feature"
    root_spec_id: String            -- common ancestor
    branching_point: String         -- where the split occurs
    branches: SubstrateChain[]      -- chains within this substrate

  USED BY: SubstrateGraph
  USES: SubstrateChain
```

### RECORD: SubstrateChain

A linear L1→L4 chain within a substrate (Phase 2).

```
RECORD SubstrateChain
  FIELDS:
    substrate_id: String            -- parent substrate
    entries: SubstrateChainEntry[]  -- ordered list of specs L1, L2, L3, L4
    shared_ancestors: String[]      -- specs common to all branches

  USED BY: Substrate, nlspec_substrate_query
  USES: SubstrateChainEntry
```

### RECORD: SubstrateChainEntry

A single spec in a substrate chain (Phase 2).

```
RECORD SubstrateChainEntry
  FIELDS:
    layer: u64                      -- layer number (1, 2, 3, 4)
    spec_id: String
    content: String | None          -- optional; if include_content=true

  USED BY: SubstrateChain
```

### RECORD: SplitSuggestion

Result of nlspec_split when execute=false (Phase 2).

```
RECORD SplitSuggestion
  FIELDS:
    namespace: String
    source_spec_id: String
    strategy: String                -- "cluster", "layer", "manual"
    clusters: SplitCluster[]        -- proposed decomposition
    cross_refs: Reference[]         -- will become IMPORTs
    rationale: String               -- why this split is recommended

  USED BY: nlspec_split (when execute=false)
  USES: SplitCluster, Reference
```

### RECORD: SplitCluster

A proposed sub-spec in a split (Phase 2).

```
RECORD SplitCluster
  FIELDS:
    name: String                    -- proposed spec name
    sections: String[]              -- sections included in this cluster
    element_count: u64
    rationale: String               -- why these elements go together

  USED BY: SplitSuggestion
```

### RECORD: DriftReport

Result of drift detection between spec and code (Phase 2).

```
RECORD DriftReport
  FIELDS:
    namespace: String
    spec_id: String
    clean: bool                     -- true iff drifted.len() == 0
    drifted: DriftItem[]            -- mismatches found
    checked_at: Timestamp

  USED BY: nlspec_drift
  USES: DriftItem
```

### RECORD: DriftItem

A single mismatch between spec and code (Phase 2).

```
RECORD DriftItem
  FIELDS:
    category: String                -- "API", "DATA_MODEL", "ERROR", etc.
    element: String                 -- what drifted (e.g., "DELETE /auth/purge")
    spec_says: String               -- spec expectation
    code_says: String               -- code reality
    severity: String                -- "EXTRA_IN_CODE", "MISSING_IN_CODE", "MODIFIED"

  USED BY: DriftReport
```

### RECORD: CodeSurface

Agent's representation of code structure (for drift detection) (Phase 2).

```
RECORD CodeSurface
  FIELDS:
    endpoints: Endpoint[]           -- HTTP endpoints found in code
    records: Record[]               -- data structures found in code
    errors: Error[]                 -- error types found in code
    configs: Config[]               -- configurations found in code
    artifacts: Artifact[]           -- other artifacts

  USED BY: nlspec_drift
```

### RECORD: LayerStackEntry

An entry in a layer stack (L1, L2, L3, L4) (Phase 2).

```
RECORD LayerStackEntry
  FIELDS:
    layer: u64                      -- 1, 2, 3, or 4
    spec_id: String
    title: String
    description: String

  USED BY: nlspec_layer_stack
```

### RECORD: CompositionPeer

A spec that is at the same layer and shares a parent (Phase 2).

```
RECORD CompositionPeer
  FIELDS:
    spec_id: String
    parent_spec_id: String | None
    sibling_count: u64

  USED BY: nlspec_layer_validate (informational)
```

### RECORD: LayerValidationReport

Result of layer constraint validation (Phase 2).

```
RECORD LayerValidationReport
  FIELDS:
    namespace: String
    spec_id: String
    layer: u64
    is_valid: bool
    issues: String[]                -- constraint violations
    peers: CompositionPeer[]        -- siblings at same layer
    checked_at: Timestamp

  USED BY: nlspec_layer_validate
  USES: CompositionPeer
```

### ENUM: ElementType

```
ENUM ElementType
  RECORD
  FUNCTION
  SCENARIO
  ERROR
  CONFIG
  ENUM
  ENDPOINT
  SECTION
```

### ENUM: ReferenceType

```
ENUM ReferenceType
  USES                             -- function uses record
  THROWS                           -- function throws error
  USED_BY                          -- inverse of USES
  SEC                              -- scenario tags section [SEC:x.x]
  IMPORT                           -- cross-spec import (Phase 2)
  CALLS                            -- function calls another function
  DERIVES_FROM                     -- spec derives from parent (Phase 2)
```

### ENUM: SpecType

```
ENUM SpecType
  SYSTEM
  COMPONENT
  LIBRARY
  INTERFACE
```

### ENUM: GapSeverity

```
ENUM GapSeverity
  CRITICAL                         -- blocks functionality
  WARNING                          -- might cause issues
  INFO                             -- informational only
```

### ENUM: PatchCategory

```
ENUM PatchCategory
  A                                -- Spec Deficiency (spec was wrong/ambiguous)
  B                                -- Implementation Error (code was wrong)
  C                                -- Missing Capability (feature not in spec)
```

### ENUM: PatchStatus

```
ENUM PatchStatus
  ACTIVE                           -- patch exists, not yet applied
  ABSORBED                         -- patch has been merged into spec
  REJECTED                         -- patch was declined
```

### ENUM: SpecStatus

```
ENUM SpecStatus
  DRAFT
  STABLE
  DEPRECATED
```

---

## Functions

### FUNCTION: parse_spec

Parse a markdown file into a Spec with sections, elements, and references.

```
FUNCTION parse_spec
  INPUT:
    markdown: String                -- raw markdown content
    spec_id: String                 -- identifier for this spec
    namespace: String | None        -- namespace (Phase 2; default "default")

  OUTPUT:
    Spec                            -- parsed spec with elements and gaps

  WORKFLOW:
    1. Extract header (metadata)
    2. Split into sections (by ## heading)
    3. For each section:
       a. Extract title
       b. Parse elements (RECORD:, FUNCTION:, SCENARIO:, etc.)
       c. For each element:
          - Extract name, type, content
          - Parse references (USES X, THROWS Y, [SEC:...], IMPORT ...)
          - Check for dangling refs, orphaned elements
    4. Run analyze_gaps on entire spec
    5. Return Spec with sections, elements, gaps populated

  THROWS:
    ParseError                      -- markdown is malformed
    InvalidSpecID                   -- spec_id is invalid format

  LATENCY: < 200ms for specs < 5MB
  ASSUMES: markdown is valid UTF-8
```

### FUNCTION: analyze_gaps

Analyze a spec or element for structural issues.

```
FUNCTION analyze_gaps
  INPUT:
    spec: Spec | SpecElement       -- what to analyze
    namespace: String | None        -- namespace context (Phase 2)

  OUTPUT:
    Gap[]                           -- identified gaps

  CHECKS:
    - Dangling references: USES/THROWS/IMPORT reference a non-existent target
    - Orphaned elements: RECORD/CONFIG exists but nothing USES it
    - Undeclared sections: elements reference sections not declared
    - Missing metadata: spec lacks required fields
    - Broken imports: IMPORT references non-existent specs (Phase 2)

  LATENCY: < 50ms for specs < 100 elements
```

### FUNCTION: store_load

Load all specs from disk into index.

```
FUNCTION store_load
  INPUT:
    project_dir: String             -- root directory
    specs_dir: String               -- subdirectory containing *.md files
    namespace: String | None        -- namespace (Phase 2)

  OUTPUT:
    Spec[]                          -- loaded and indexed specs

  WORKFLOW:
    1. Scan specs_dir for *.md files
    2. For each file:
       a. Read content
       b. Call parse_spec
       c. INSERT into database
    3. Build FTS index
    4. Return loaded specs

  THROWS:
    FileNotFound
    DatabaseError
```

### FUNCTION: store_get

Retrieve a spec, section, or element by ID.

```
FUNCTION store_get
  INPUT:
    spec_id: String
    namespace: String | None        -- namespace (Phase 2; default "default")
    section: String | None          -- optional; if set, return that section only
    element_id: String | None       -- optional; if set, return that element only

  OUTPUT:
    Spec | Section | SpecElement    -- depends on input

  WORKFLOW:
    1. Query database:
       - If both spec_id + element_id: return SpecElement
       - Else if spec_id + section: return Section with all elements
       - Else if spec_id only: return Spec
    2. Populate references for each element
    3. Return structured object

  THROWS:
    SpecNotFound
    ElementNotFound
    SectionNotFound

  LATENCY: < 50ms (with warm database connection)
```

### FUNCTION: store_create

Create and insert a new element into a spec.

```
FUNCTION store_create
  INPUT:
    spec_id: String                 -- parent spec
    namespace: String | None        -- namespace (Phase 2)
    section: String                 -- parent section name
    element_type: ElementType       -- RECORD, FUNCTION, SCENARIO, etc.
    name: String                    -- element name
    content: String                 -- markdown body
    tags: String[]                  -- semantic tags

  OUTPUT:
    SpecElement                     -- created element with assigned ID

  WORKFLOW:
    1. Validate input (name not empty, spec exists, section exists)
    2. Check uniqueness: (spec_id, section, element_type, name) must be unique
    3. Parse content for references
    4. INSERT into database (elements, references)
    5. UPDATE markdown file (append to end of section)
    6. Return created SpecElement

  THROWS:
    SpecNotFound
    SectionNotFound
    DuplicateElement
    ParseError

  SIDE EFFECTS:
    - Markdown file is modified (atomic rename)
    - Git sees the change (if git enabled)

  LATENCY: < 100ms
```

### FUNCTION: store_update

Modify an existing element.

```
FUNCTION store_update
  INPUT:
    element_id: String              -- full element ID
    content: String                 -- new markdown body
    tags: String[] | None           -- optional tag update

  OUTPUT:
    SpecElement                     -- updated element

  WORKFLOW:
    1. Load current element
    2. Validate: element exists, content is parseable
    3. Reparse content for references
    4. UPDATE database (content, references)
    5. Find and replace in markdown file (atomic)
    6. Return updated element

  THROWS:
    ElementNotFound
    ParseError
    FileNotFound

  SIDE EFFECTS:
    - Markdown file is modified
    - Git sees the change

  LATENCY: < 100ms
```

### FUNCTION: store_delete

Remove an element from the index and markdown file.

```
FUNCTION store_delete
  INPUT:
    element_id: String              -- full element ID

  OUTPUT:
    bool                            -- true if deleted

  WORKFLOW:
    1. Load element
    2. DELETE from database (element, its references)
    3. Find element in markdown file
    4. Remove that block (atomic)
    5. Return true

  THROWS:
    ElementNotFound

  SIDE EFFECTS:
    - Markdown file is modified
    - Git sees the change
```

### FUNCTION: query_search

Search for elements by text or reference.

```
FUNCTION query_search
  INPUT:
    query: String | None            -- free-text query
    element_type: ElementType | None -- filter by type
    tags: String[] | None           -- filter by tags (all must match)
    references: String | None       -- find elements that USES/THROWS X
    spec_id: String | None          -- search in specific spec
    namespace: String | None        -- search in specific namespace (Phase 2)
    limit: u64                      -- max results (default 50)

  OUTPUT:
    SearchResult[]                  -- ranked by relevance

  WORKFLOW:
    1. Build query:
       - If references: find elements where ref matches target_name, reference_type=USES/THROWS/CALLS
       - Else: use FTS query on content
    2. Filter by element_type, tags, spec_id, namespace
    3. Rank by relevance (FTS score or reference proximity)
    4. LIMIT and return

  ASSUMES:
    - FTS index is warm
    - element_type filter reduces candidates significantly

  LATENCY: < 50ms (FTS) or < 30ms (reference-based)
```

### FUNCTION: query_references

Find incoming and outgoing references for an element.

```
FUNCTION query_references
  INPUT:
    element_id: String              -- element to analyze
    direction: String               -- "in" (incoming), "out" (outgoing), "both"

  OUTPUT:
    Reference[]                     -- matching references

  WORKFLOW:
    1. Load element
    2. If direction="out": return element.references
    3. If direction="in": query for all elements that reference this one
    4. If direction="both": union of in + out
    5. Populate reference targets (element names, resolved status)
    6. Return Reference[]

  LATENCY: < 30ms
```

### FUNCTION: context_slicer

Extract minimal context for a specific problem (scenario, function, etc.) (Phase 2).

```
FUNCTION context_slicer
  INPUT:
    namespace: String
    spec_id: String
    trigger: String                 -- scenario ID, function name, record name, etc.
    max_depth: u64                  -- max traversal depth (default 3)
    max_elements: u64               -- max elements to collect (default 500)

  OUTPUT:
    ContextSlice                    -- collected elements

  WORKFLOW:
    1. Identify trigger element (SCENARIO, FUNCTION, RECORD, etc.)
    2. Depth-first traversal from trigger:
       a. Collect element itself
       b. Follow outgoing references (USES, THROWS, SEC tags)
       c. Recursively collect referenced elements up to max_depth
       d. For each IMPORT statement, recursively slice from imported spec
    3. Order elements by section, then position
    4. Estimate total_lines by counting rendered markdown
    5. Build human-readable trace
    6. Return ContextSlice

  LATENCY: < 100ms
  ASSUMES: max_depth is small (2-3), cross-namespace imports are few
```

### FUNCTION: patch_create

Create a new patch for a spec (Phase 2).

```
FUNCTION patch_create
  INPUT:
    namespace: String
    spec_id: String
    category: PatchCategory         -- A, B, or C
    sections: String[]              -- affected sections
    scenarios: u64[]                -- affected scenario IDs
    description: String             -- why this patch
    content: String                 -- proposed change

  OUTPUT:
    Patch                           -- created with status=ACTIVE

  WORKFLOW:
    1. Allocate patch ID (PATCH-NNN)
    2. INSERT into database
    3. Write patch file to patches/{namespace}/{spec_id}/PATCH-NNN.md
    4. Return Patch

  THROWS:
    SpecNotFound
    InvalidCategory

  SIDE EFFECTS: Patch file is created on disk
```

### FUNCTION: patch_list

List all patches for a spec (Phase 2).

```
FUNCTION patch_list
  INPUT:
    namespace: String
    spec_id: String
    status: PatchStatus | None      -- filter by status

  OUTPUT:
    Patch[]                         -- all patches for spec

  WORKFLOW:
    1. Query database for patches in (namespace, spec_id)
    2. Filter by status if provided
    3. Return Patch[]
```

### FUNCTION: patch_absorb

Merge a patch into a spec (Phase 2).

```
FUNCTION patch_absorb
  INPUT:
    patch_id: String

  OUTPUT:
    Patch                           -- with status=ABSORBED (returned before deletion)

  WORKFLOW:
    1. Load patch
    2. Verify patch content is already reflected in spec body:
       a. The spec element content should already include the patch changes
          (patches are applied to the spec body when created/activated, NOT at absorption time)
       b. If spec body does NOT reflect the patch: apply patch content now (backward compat)
    3. Increment spec version (patch: 0.1.0 → 0.1.1, minor: 0.1.0 → 0.2.0)
    4. UPDATE Patch: status=ABSORBED, absorbed_at=NOW()
    5. DELETE Patch from spec's Patches section:
       a. Remove the Patch record from the in-memory store
       b. Remove the patch block from the spec's markdown Patches section (if present)
       c. The spec body retains the patch's changes — only the Patch metadata is deleted
    6. git commit (if git enabled) with message: "absorb(PATCH-{id}): {description}"
    7. Return the Patch record (snapshot before deletion, for caller confirmation)

  POSTCONDITIONS:
    - Spec body unchanged (already contained patch content)
    - Spec version bumped
    - Patch record no longer exists in store or spec file
    - nlspec_patch_list for this spec will no longer include this patch
    - nlspec_diff_since will still show the element changes between historical versions
      (diff is computed from spec history, not from patch records)

  THROWS:
    PatchNotFound
    SpecNotFound

  NOTES:
    - Absorption is the END of a patch's lifecycle. After this call, the patch
      is gone. The spec IS the canonical state — no patch metadata remains.
    - The caller (BuildPlanner.commit_unit) is responsible for ensuring all
      affected artifacts are committed BEFORE calling patch_absorb.
    - Historical diffs remain available: nlspec_diff_since("0.1.0") will still
      show what changed between versions, even though the patch record is deleted.
      This is because diff is computed from spec content snapshots, not patch records.
```

### FUNCTION: diff_since

Compute a structured diff between two versions of a spec at element granularity (Phase 2).

```
FUNCTION diff_since
  INPUT:
    namespace: String
    spec_id: String
    since_version: String           -- the version to diff FROM (e.g., "0.1.0")

  OUTPUT:
    StructuredDiff                  -- element-level changes with sub-change classification

  WORKFLOW:
    1. Load current spec version (to_version)
    2. If since_version == to_version: return empty StructuredDiff (no changes)
    3. Load spec snapshot at since_version:
       a. If git enabled: retrieve spec file at the git commit tagged with since_version
       b. If git not enabled: retrieve from version history table in database
    4. Parse both versions into SpecElement lists using parse_spec
    5. Build element index for both versions:
       a. Key = (section, element_type, element_name)
       b. Value = SpecElement with content hash
    6. Compute element-level diff:
       a. Elements in to_version but not in since_version: change_type = ADDED
       b. Elements in since_version but not in to_version: change_type = REMOVED
       c. Elements in both but hash differs: change_type = MODIFIED
       d. Elements in both with same hash: skip (unchanged)
    7. For each MODIFIED element, compute sub_changes:
       a. Parse element content into structural parts:
          - For FUNCTION: extract signature (name, params, return), PRECONDITIONS, POSTCONDITIONS,
            BEHAVIOR, ERRORS, USES, THROWS
          - For RECORD: extract field list, INVARIANTS, USED BY
          - For SCENARIO: extract GIVEN, WHEN, THEN, tags
          - For ENUM: extract variant list
       b. Compare each structural part between versions
       c. Map changed parts to SubChange enum values:
          - Params/return changed → SIGNATURE
          - PRECONDITIONS section differs → PRECONDITIONS
          - POSTCONDITIONS section differs → POSTCONDITIONS
          - Field list differs → DATA_FIELDS
          - INVARIANTS section differs → INVARIANTS
          - ERRORS/THROWS changed → ERROR_CASES
          - BEHAVIOR steps changed → BEHAVIOR
          - GIVEN/WHEN/THEN changed → SCENARIOS
          - USES/THROWS/USED_BY declarations changed → REFERENCES
          - NOTES/PERFORMANCE/description changed → METADATA
    8. For each ElementChange, classify impact:
       a. ADDED elements: impact = ADDITIVE
       b. REMOVED elements: impact = REMOVAL
       c. MODIFIED elements:
          - If sub_changes contains SIGNATURE or DATA_FIELDS (required field added/removed):
            impact = INTERFACE_BREAKING
          - Else if sub_changes contains only PRECONDITIONS, POSTCONDITIONS, BEHAVIOR, ERROR_CASES:
            impact = LOGIC_ONLY
          - Else if sub_changes contains only DATA_FIELDS (optional field added) or METADATA:
            impact = ADDITIVE
          - Else: impact = LOGIC_ONLY (conservative default for ambiguous cases)
    9. Collect absorbed patches between since_version and to_version:
       a. Query patches where status=ABSORBED and version_after > since_version
       b. Set patches_included = matching patch IDs
       c. For each ElementChange, set patch_ids from patches whose sections overlap
    10. Return StructuredDiff

  THROWS:
    SpecNotFound
    VersionNotFound                 -- since_version doesn't exist in history

  LATENCY: < 500ms for specs with < 200 elements

  NOTES:
  - Content hashing uses SHA-256 of the element's raw markdown content (trimmed, normalized whitespace)
  - This function is the bridge between nlspec (what changed in the spec) and l2mify (what to rebuild)
  - The impact classification is conservative: when ambiguous, it chooses the higher-impact classification
    to avoid under-rebuilding
  - Sub-element parsing reuses the same parse_spec infrastructure used for indexing
  - METADATA-only changes (notes, performance targets) still appear in the diff but with ADDITIVE impact,
    signaling that they're informational and don't require artifact changes
```

### FUNCTION: spec_validate

Validate a spec for structural correctness (Phase 2).

```
FUNCTION spec_validate
  INPUT:
    namespace: String
    spec_id: String

  OUTPUT:
    ValidationResult

  CHECKS:
    1. Dangling references: USES/THROWS/IMPORT reference non-existent targets
    2. Orphaned elements: RECORD/CONFIG exists but nothing USES it
    3. Undeclared sections: elements reference non-existent sections
    4. Broken imports: IMPORT statements reference non-existent specs
    5. Re-run gap detection: CRITICAL, WARNING, INFO gaps
    6. Compute is_valid = (errors.len() == 0)

  LATENCY: < 150ms
  ASSUMES: element count < 1000, cross-namespace queries are batched
```

### FUNCTION: build_graph

Build a spec dependency graph (Phase 2).

```
FUNCTION build_graph
  INPUT:
    namespace: String               -- which namespace to analyze

  OUTPUT:
    SubstrateGraph                  -- nodes, edges, substrates

  WORKFLOW:
    1. Load all specs in namespace
    2. Extract DERIVES FROM parent and IMPORTS
    3. Build nodes (one per spec)
    4. Build edges (parent → child, importer → imported)
    5. Identify branching points (specs with 2+ children)
    6. Assign substrates (spatial/temporal/feature based on branching structure)
    7. Cache graph with TTL
    8. Return SubstrateGraph

  LATENCY: < 1s (rebuild only when cache stale)
```

### FUNCTION: build_substrate_graph

Same as build_graph (alias for clarity) (Phase 2).

```
FUNCTION build_substrate_graph
  INPUT:
    namespace: String

  OUTPUT:
    SubstrateGraph
```

### FUNCTION: resolve_substrate_chain

Resolve a single substrate to a linear L1→L4 chain (Phase 2).

```
FUNCTION resolve_substrate_chain
  INPUT:
    namespace: String
    substrate_id: String            -- substrate identifier (e.g., "ios")
    from_layer: u64                 -- starting layer (default 1)
    to_layer: u64                   -- ending layer (default 4)
    include_content: bool           -- include markdown (default false)

  OUTPUT:
    SubstrateChain                  -- ordered specs L1→L4

  WORKFLOW:
    1. Load/rebuild substrate graph (from cache if fresh)
    2. Find Substrate with id=substrate_id
    3. Extract root_spec_id and branches
    4. Build ordered chain from from_layer to to_layer
    5. Collect shared_ancestors
    6. If include_content: load markdown for each entry
    7. Return SubstrateChain

  LATENCY: < 100ms (< 1s if rebuilding graph)
```

### FUNCTION: namespace_manager

Manage namespaces (create, list, delete) (Phase 2).

```
FUNCTION namespace_manager
  INPUT:
    operation: String               -- "create", "list", "delete", "get"
    namespace: String | None        -- required for "create", "delete", "get"

  OUTPUT:
    Namespace | Namespace[]

  WORKFLOW:
    1. If operation="create":
       a. CREATE namespace (INSERT into database)
       b. Return Namespace
    2. If operation="list":
       a. SELECT all namespaces
       b. Return Namespace[]
    3. If operation="get":
       a. SELECT by name
       b. Return Namespace
    4. If operation="delete":
       a. DELETE namespace and all specs (cascading)
       b. Return confirmation
```

### FUNCTION: spec_split

Suggest or execute decomposition of a monolith spec (Phase 2).

```
FUNCTION spec_split
  INPUT:
    namespace: String
    spec_id: String
    strategy: String                -- "cluster" (default), "layer", "manual"
    execute: bool                   -- false=suggest only, true=create files

  OUTPUT:
    SplitSuggestion | Spec[]        -- suggestion if execute=false, created specs if execute=true

  WORKFLOW (suggest mode):
    1. Build reference graph for all elements
    2. Run community detection (clustering algorithm)
    3. Identify clusters of cohesive elements
    4. For each cluster:
       a. Propose new spec_id (based on cluster topic)
       b. Identify sections/elements to move
       c. Calculate cross-cluster references (will become IMPORTs)
    5. Return SplitSuggestion

  WORKFLOW (execute mode):
    1. Generate suggestion
    2. For each cluster:
       a. Create new spec file
       b. Add IMPORT statements for cross-cluster refs
       c. Validate new spec
    3. Update original spec (mark as deprecated or reorganized)
    4. Return new Spec[] objects

  LATENCY: < 500ms (suggest), < 1s (execute)
```

### FUNCTION: detect_drift

Compare spec against code (Phase 2).

```
FUNCTION detect_drift
  INPUT:
    namespace: String
    spec_id: String
    code_surface: CodeSurface       -- agent's code analysis

  OUTPUT:
    DriftReport

  WORKFLOW:
    1. Load spec from database
    2. Extract all ENDPOINT, RECORD, ERROR, CONFIG declarations
    3. Compare against code_surface:
       a. For each endpoint in code: is it in spec? If not: EXTRA_IN_CODE
       b. For each endpoint in spec: is it in code? If not: MISSING_IN_CODE
       c. Similar checks for RECORD fields, ERROR types, etc.
    4. Collect mismatches into DriftReport
    5. Set clean = (drifted.len() == 0)
    6. Return DriftReport

  LATENCY: < 150ms
  NOTE: Drift is advisory. Agents decide: update spec or update code?
```

---

## API — MCP Tools

The system provides 21 MCP tools across two phases: 8 from Phase 1 (bootstrap) and 13 from Phase 2 (extended).

### TOOL: nlspec_init

Initialize a new project with a template spec.

```
TOOL nlspec_init
  INPUT:
    project_dir: String             -- root directory for project
    spec_name: String               -- name for initial spec (e.g., "myservice")
    namespace: String | None        -- namespace (Phase 2; default "default")

  OUTPUT:
    {
      spec_id: String
      path: String
      status: String
      message: String
    }

  DESCRIPTION:
    Creates project directory structure and an initial spec from template.
    - Creates project_dir/specs/ subdirectory
    - Copies bundled template to project_dir/specs/{spec_name}-spec.md
    - Initializes database (.nlspec/index.sqlite or PostgreSQL)
    - Returns Spec object with status "Draft"

  USES: store_load, parse_spec
  THROWS: ProjectAlreadyExists, PermissionDenied, DatabaseError
```

### TOOL: nlspec_get

Retrieve a spec, section, or element.

```
TOOL nlspec_get
  INPUT:
    spec_id: String
    namespace: String | None        -- namespace (Phase 2; default "default")
    section: String | None
    element_id: String | None
    format: String                  -- "json" or "markdown" (default: json)

  OUTPUT:
    {
      spec_id: String
      section: String | None
      element_id: String | None
      content: String or SpecElement or Section or Spec
      format: String
    }

  DESCRIPTION:
    Retrieve by level of specificity:
    - (spec_id) → entire Spec with all sections
    - (spec_id, section) → Section with all elements
    - (spec_id, section, element_id) → single SpecElement

    Format: "json" returns structured objects, "markdown" returns rendered markdown.

  USES: store_get
  THROWS: SpecNotFound, ElementNotFound, SectionNotFound
```

### TOOL: nlspec_list

List all specs or elements with filters.

```
TOOL nlspec_list
  INPUT:
    spec_id: String | None          -- list elements in this spec only
    namespace: String | None        -- filter by namespace (Phase 2)
    element_type: ElementType | None
    tags: String[] | None           -- all must match (AND logic)
    limit: u64                      -- default 100
    offset: u64                     -- default 0

  OUTPUT:
    {
      total: u64
      items: (Spec | SpecElement)[]
      limit: u64
      offset: u64
    }

  DESCRIPTION:
    List specs or elements. If spec_id omitted: list all specs. If spec_id set: list
    elements in that spec with optional element_type and tag filters.

  USES: store_get
  THROWS: SpecNotFound
```

### TOOL: nlspec_search

Full-text and structural search.

```
TOOL nlspec_search
  INPUT:
    query: String | None            -- free-text search
    element_type: ElementType | None
    tags: String[] | None
    references: String | None       -- find elements that USES/THROWS X
    spec_id: String | None
    namespace: String | None        -- search in specific namespace (Phase 2)
    limit: u64                      -- default 50

  OUTPUT:
    {
      results: SearchResult[]
      query: String
      total_matches: u64
    }

  DESCRIPTION:
    Search for elements by text, type, tags, or reference. Supports:
    - Full-text search: "TTL expiration" → returns elements containing those words
    - Reference search: references="Entry" → returns all FUNCTIONs that USES Entry
    - Combined: references="Entry", element_type="FUNCTION" → FUNCTIONs using Entry

    Results ranked by relevance; lower scores filtered by config.search.min_relevance.

  USES: query_search, query_references
  THROWS: InvalidQuery
```

### TOOL: nlspec_create

Create a new element in a spec.

```
TOOL nlspec_create
  INPUT:
    spec_id: String
    namespace: String | None        -- namespace (Phase 2)
    section: String
    element_type: ElementType
    name: String
    content: String
    tags: String[]

  OUTPUT:
    {
      element_id: String
      spec_id: String
      section: String
      name: String
      element_type: ElementType
      created_at: Timestamp
    }

  DESCRIPTION:
    Add a new element to a spec and markdown file. Element name must be unique within
    (spec_id, section, element_type). Content is parsed for references.

  USES: store_create, parse_spec
  THROWS: SpecNotFound, SectionNotFound, DuplicateElement, ParseError
```

### TOOL: nlspec_update

Modify an existing element.

```
TOOL nlspec_update
  INPUT:
    element_id: String
    content: String
    tags: String[] | None

  OUTPUT:
    {
      element_id: String
      updated_at: Timestamp
      content: String
    }

  DESCRIPTION:
    Update an element's content and optionally its tags. Content is re-parsed for
    references. Markdown file is updated atomically.

  USES: store_update, parse_spec
  THROWS: ElementNotFound, ParseError
```

### TOOL: nlspec_delete

Remove an element from the spec and index.

```
TOOL nlspec_delete
  INPUT:
    element_id: String

  OUTPUT:
    {
      element_id: String
      deleted_at: Timestamp
      message: String
    }

  DESCRIPTION:
    Delete an element. This is permanent — the element is removed from both the
    database and markdown file.

  USES: store_delete
  THROWS: ElementNotFound
```

### TOOL: nlspec_query

Advanced query with filter expressions.

```
TOOL nlspec_query
  INPUT:
    filter: String                  -- filter expression (TBD syntax)

  OUTPUT:
    SearchResult[]

  DESCRIPTION:
    (Optional tool, only if defined.) Supports advanced filtering
    with expressions like: element_type="FUNCTION" AND references="Entry" AND tags contains "SMOKE"
```

### TOOL: nlspec_import

Import a spec from a markdown file into a namespace (Phase 2).

```
TOOL nlspec_import
  INPUT:
    namespace: String               -- target namespace
    spec_id: String                 -- identifier for imported spec
    path: String                    -- markdown file path

  OUTPUT:
    {
      spec_id: String
      namespace: String
      element_count: u64
      message: String
    }

  DESCRIPTION:
    Parse and index a spec file into a namespace. Creates namespace if it doesn't exist.
    Re-importing same (namespace, spec_id, path) refreshes the index.

  USES: namespace_manager, parse_spec, store_create
  THROWS: SpecNotFound, ParseError, NamespaceCreateError
```

### TOOL: nlspec_namespaces

List and manage namespaces (Phase 2).

```
TOOL nlspec_namespaces
  INPUT:
    namespace: String | None        -- if set, get details for that namespace

  OUTPUT:
    {
      namespaces: Namespace[]       -- all if no filter
    }

  DESCRIPTION:
    List all namespaces with their specs.

  USES: namespace_manager
```

### TOOL: nlspec_slice

Extract minimal context for a specific problem (Phase 2).

```
TOOL nlspec_slice
  INPUT:
    namespace: String
    spec_id: String
    scenario: u64 | None            -- which scenario to slice for
    function: String | None         -- or which function
    record: String | None           -- or which record
    max_depth: u64                  -- max traversal depth (default 3)

  OUTPUT:
    {
      namespace: String
      spec_id: String
      trigger: String
      elements: SpecElement[]
      specs_touched: String[]
      total_lines: u64
      trace: String
    }

  DESCRIPTION:
    Return the minimal spec context needed to understand and fix a specific scenario,
    function, or record. Traverses references up to max_depth.

  USES: context_slicer
  THROWS: SpecNotFound, ElementNotFound
```

### TOOL: nlspec_patch_create

Create a new patch for a spec (Phase 2).

```
TOOL nlspec_patch_create
  INPUT:
    namespace: String
    spec_id: String
    category: String                -- "A", "B", or "C"
    sections: String[]
    scenarios: u64[]
    description: String
    content: String

  OUTPUT:
    {
      patch_id: String
      status: String
      created_at: Timestamp
    }

  DESCRIPTION:
    Create a patch record for a spec issue. Patch is stored in database and on disk.

  USES: patch_create
  THROWS: SpecNotFound
```

### TOOL: nlspec_patch_list

List patches for a spec (Phase 2).

```
TOOL nlspec_patch_list
  INPUT:
    namespace: String
    spec_id: String
    status: String | None           -- filter by "ACTIVE", "ABSORBED", "REJECTED"

  OUTPUT:
    {
      patches: Patch[]
      total: u64
    }

  DESCRIPTION:
    List all patches for a spec with optional status filter.

  USES: patch_list
  THROWS: SpecNotFound
```

### TOOL: nlspec_patch_absorb

Merge a patch into a spec (Phase 2).

```
TOOL nlspec_patch_absorb
  INPUT:
    patch_id: String

  OUTPUT:
    {
      patch_id: String
      status: String                -- "ABSORBED"
      spec_version: String          -- updated version
    }

  DESCRIPTION:
    Apply a patch to its spec. Patch status becomes "ABSORBED", spec version is bumped.

  USES: patch_absorb
  THROWS: PatchNotFound
```

### TOOL: nlspec_diff_since

Compute a structured diff between a historical version and the current version of a spec (Phase 2).

```
TOOL nlspec_diff_since
  INPUT:
    namespace: String
    spec_id: String
    since_version: String           -- version to diff FROM

  OUTPUT:
    {
      spec_id: String
      from_version: String
      to_version: String
      changes: [
        {
          element_id: String
          element_name: String
          element_type: String      -- "RECORD", "FUNCTION", etc.
          section: String
          change_type: String       -- "MODIFIED", "ADDED", "REMOVED"
          sub_changes: String[]     -- ["SIGNATURE", "BEHAVIOR", etc.]
          impact: String            -- "INTERFACE_BREAKING", "LOGIC_ONLY", "ADDITIVE", "REMOVAL"
          hash_before: String | None
          hash_after: String | None
          patch_ids: String[]
        }
      ]
      patches_included: String[]
      diffed_at: Timestamp
    }

  DESCRIPTION:
    Compute element-level structured diff between since_version and current version.
    Each change includes sub-element classification (what specifically changed within
    a FUNCTION or RECORD) and impact classification (whether callers need to change).
    This is the primary integration point for build planners that need to know precisely
    what changed in a spec to determine the minimal set of artifacts to rebuild.

  USES: diff_since
  THROWS: SpecNotFound, VersionNotFound
```

### TOOL: nlspec_validate

Validate a spec for structural correctness (Phase 2).

```
TOOL nlspec_validate
  INPUT:
    namespace: String
    spec_id: String

  OUTPUT:
    {
      is_valid: bool
      errors: ValidationError[]
      warnings: ValidationWarning[]
      gaps: Gap[]
    }

  DESCRIPTION:
    Comprehensive validation: dangling refs, orphaned elements, broken imports, gaps.
    Returns actionable feedback but never rejects a spec.

  USES: spec_validate
  THROWS: SpecNotFound
```

### TOOL: nlspec_graph

Build and query the spec dependency graph (Phase 2).

```
TOOL nlspec_graph
  INPUT:
    namespace: String
    include_content: bool           -- include markdown (default false)

  OUTPUT:
    {
      nodes: SubstrateNode[]
      edges: SubstrateEdge[]
      substrates: Substrate[]
    }

  DESCRIPTION:
    Return the full dependency graph for a namespace. Useful for visualization
    and understanding how specs relate.

  USES: build_graph
  THROWS: NamespaceNotFound
```

### TOOL: nlspec_split

Suggest or execute spec decomposition (Phase 2).

```
TOOL nlspec_split
  INPUT:
    namespace: String
    spec_id: String
    strategy: String                -- "cluster" (default), "layer", "manual"
    execute: bool                   -- false=suggest, true=execute

  OUTPUT:
    {
      suggestion: SplitSuggestion   -- if execute=false
      created_specs: Spec[]         -- if execute=true
    }

  DESCRIPTION:
    Suggest how to decompose a monolith spec into multiple specs. If execute=true,
    create the new specs with appropriate IMPORT statements.

  USES: spec_split
  THROWS: SpecNotFound
```

### TOOL: nlspec_drift

Detect drift between spec and code (Phase 2).

```
TOOL nlspec_drift
  INPUT:
    namespace: String
    spec_id: String
    code_surface: {
      endpoints: Endpoint[]
      records: Record[]
      errors: Error[]
      configs: Config[]
      artifacts: Artifact[]
    }

  OUTPUT:
    {
      clean: bool
      drifted: DriftItem[]
      checked_at: Timestamp
    }

  DESCRIPTION:
    Compare code against spec. Report mismatches (EXTRA_IN_CODE, MISSING_IN_CODE, MODIFIED).
    Helps keep specs and code aligned.

  USES: detect_drift
  THROWS: SpecNotFound
```

### TOOL: nlspec_layer_stack

Get the layer stack (L1, L2, L3, L4) for a spec (Phase 2).

```
TOOL nlspec_layer_stack
  INPUT:
    namespace: String
    spec_id: String

  OUTPUT:
    {
      stack: LayerStackEntry[]      -- ordered L1→L4
      depth: u64
    }

  DESCRIPTION:
    Return the linear stack of layers from L1 to L4, following DERIVES FROM.

  USES: resolve_substrate_chain
  THROWS: SpecNotFound
```

### TOOL: nlspec_layer_validate

Validate layer constraints for a spec (Phase 2).

```
TOOL nlspec_layer_validate
  INPUT:
    namespace: String
    spec_id: String

  OUTPUT:
    {
      is_valid: bool
      layer: u64
      issues: String[]
      peers: CompositionPeer[]
    }

  DESCRIPTION:
    Validate that spec's layer is correctly declared and its parents/children are
    valid. Report constraint violations.

  USES: LayerValidationReport
  THROWS: SpecNotFound
```

### TOOL: nlspec_substrate_query

Query a specific substrate within a namespace (Phase 2).

```
TOOL nlspec_substrate_query
  INPUT:
    namespace: String
    substrate: String               -- substrate ID (e.g., "ios")
    from_layer: u64                 -- starting layer (default 1)
    to_layer: u64                   -- ending layer (default 4)
    include_content: bool           -- include markdown (default false)

  OUTPUT:
    {
      substrate_id: String
      chain: SubstrateChainEntry[]  -- ordered specs
      shared_ancestors: String[]
    }

  DESCRIPTION:
    Resolve a substrate ID to a linear chain of specs from from_layer to to_layer.
    Useful for navigation and understanding branching.

  USES: resolve_substrate_chain
  THROWS: NamespaceNotFound, SubstrateNotFound
```

### TOOL: nlspec_substrate_graph

Build the full substrate graph for a namespace (Phase 2).

```
TOOL nlspec_substrate_graph
  INPUT:
    namespace: String

  OUTPUT:
    {
      substrates: Substrate[]
      total_specs: u64
      branching_points: u64
    }

  DESCRIPTION:
    Return all substrates (spatial/temporal/feature branches) in a namespace.
    Shows how specs branch and which substrate IDs can be queried.

  USES: build_substrate_graph
  THROWS: NamespaceNotFound
```

---

## Error Model

All errors are NlspecError subclasses.

### ERROR: ParseError

```
ERROR ParseError
  STATUS: 400
  MESSAGE: "Failed to parse markdown: {detail}"
  CONTEXT:
    - spec_id: which spec failed
    - line_number: where the parse failed
    - content_preview: markdown snippet

  CAUSES:
    - Invalid heading syntax
    - Malformed element declaration (missing colon)
    - Invalid reference syntax
```

### ERROR: SpecNotFound

```
ERROR SpecNotFound
  STATUS: 404
  MESSAGE: "Spec not found: {spec_id}"
  CONTEXT:
    - spec_id: the spec that was requested
    - available_specs: list of loaded spec IDs
```

### ERROR: ElementNotFound

```
ERROR ElementNotFound
  STATUS: 404
  MESSAGE: "Element not found: {element_id}"
```

### ERROR: SectionNotFound

```
ERROR SectionNotFound
  STATUS: 404
  MESSAGE: "Section not found in {spec_id}: {section}"
  CONTEXT:
    - spec_id
    - section_name
    - available_sections: list of valid sections
```

### ERROR: DuplicateElement

```
ERROR DuplicateElement
  STATUS: 409
  MESSAGE: "Element already exists: {element_id}"
  CONTEXT:
    - existing_id: the element that conflicts
    - spec_id, section, element_type, name
```

### ERROR: InvalidQuery

```
ERROR InvalidQuery
  STATUS: 400
  MESSAGE: "Invalid query: {reason}"
  EXAMPLES:
    - "Unknown filter key: 'foo'"
    - "limit must be >= 1"
```

### ERROR: DatabaseError

```
ERROR DatabaseError
  STATUS: 500
  MESSAGE: "Database operation failed: {detail}"
  CONTEXT:
    - operation: "INSERT", "SELECT", "UPDATE", etc.
    - underlying_error: from PostgreSQL/SQLite
```

### ERROR: FileNotFound

```
ERROR FileNotFound
  STATUS: 404
  MESSAGE: "File not found: {path}"
```

### ERROR: PermissionDenied

```
ERROR PermissionDenied
  STATUS: 403
  MESSAGE: "Permission denied: {path}"
```

### ERROR: ProjectAlreadyExists

```
ERROR ProjectAlreadyExists
  STATUS: 409
  MESSAGE: "Project already exists at {project_dir}"
```

### ERROR: NamespaceNotFound

```
ERROR NamespaceNotFound
  STATUS: 404
  MESSAGE: "Namespace not found: {namespace}"
  CONTEXT:
    - available_namespaces: String[]
```

### ERROR: BrokenImport

```
ERROR BrokenImport
  STATUS: 400
  MESSAGE: "Broken import in {spec_id}: cannot find {target_spec_id}"
  CONTEXT:
    - source_spec_id
    - target_spec_id
    - available_specs: in namespace
```

### ERROR: PatchNotFound

```
ERROR PatchNotFound
  STATUS: 404
  MESSAGE: "Patch not found: {patch_id}"
```

### ERROR: VersionNotFound

```
ERROR VersionNotFound
  STATUS: 404
  MESSAGE: "Version not found for {spec_id}: {version}"
  CONTEXT:
    - spec_id: the spec whose version history was queried
    - version: the version that was requested
    - available_versions: String[]  -- versions that exist in history
```

### ERROR: SubstrateNotFound

```
ERROR SubstrateNotFound
  STATUS: 404
  MESSAGE: "Substrate not found in {namespace}: {substrate_id}"
  CONTEXT:
    - available_substrates: String[]
```

---

## Contracts

### Contract: Spec Parsing

**What it provides:** `parse_spec` always succeeds in producing a Spec object, even if gaps are identified. No parse failure aborts the process.

**Guarantee:** Every RECORD:, FUNCTION:, SCENARIO:, ERROR:, CONFIG:, ENUM:, ENDPOINT: in markdown becomes a SpecElement. References (USES, THROWS, IMPORT, [SEC:...]) are extracted and stored in SpecElement.references.

**Gap Handling:** Issues are collected in gaps[]. A spec with gaps is still usable; gaps inform consumers that there are issues to address.

---

### Contract: CRUD Atomicity

**What it provides:** `store_create`, `store_update`, `store_delete` are atomic with respect to the markdown file and database.

**Guarantee:** Either both the database and markdown file are updated, or neither. If a failure occurs mid-operation, no partial state is left.

**Rollback:** If database update succeeds but markdown file update fails, the database transaction is rolled back.

---

### Contract: Search Consistency

**What it provides:** `query_search` always sees the latest state after a `store_*` operation completes.

**Guarantee:** FTS index is refreshed immediately after INSERT/UPDATE/DELETE. No stale search results.

---

### Contract: Reference Resolution

**What it provides:** `Reference.resolved` is accurate for the current spec.

**Guarantee:** If a FUNCTION USES Entry, and Entry is defined in the same spec, resolved=true. If Entry is not found, resolved=false. Cross-spec references (IMPORT) are resolved when both specs are in the same namespace or linked.

---

### Contract: Namespace Isolation (Phase 2)

**What it provides:** Specs in different namespaces do not interfere. Queries on namespace A do not see specs from namespace B.

**Guarantee:** All MCP tools accept optional `namespace` parameter. If provided, operations are scoped to that namespace. If omitted, default namespace is used.

---

### Contract: Import Consistency (Phase 2)

**What it provides:** When a spec is imported, all IMPORT statements are validated. Broken imports are reported.

**Guarantee:** If nlspec_validate is called and returns is_valid=true, all IMPORT statements can be resolved.

---

### Contract: Substrate Resolution (Phase 2)

**What it provides:** Substrates are automatically detected and queryable.

**Guarantee:** Given a namespace, nlspec_substrate_graph identifies all spatial, temporal, and feature branches. Each substrate can be queried via nlspec_substrate_query to get a linear L1→L4 chain.

---

## Component Inventory

### Core Modules

| Module | Responsibility |
|--------|-----------------|
| `parser/spec_parser.py` | `parse_spec`, `analyze_gaps` |
| `store/spec_store.py` | `store_load`, `store_get`, `store_create`, `store_update`, `store_delete` |
| `query/query_engine.py` | `query_search`, `query_references` |
| `mcp/server.py` | MCP server setup, tool registration |
| `mcp/tools/*.py` | Individual tool implementations |

### Phase 2 New Modules

| Module | Responsibility |
|--------|-----------------|
| `namespace/namespace_manager.py` | Manage namespaces |
| `slicer/context_slicer.py` | Extract minimal context |
| `patch/patch_manager.py` | Create, list, absorb patches |
| `validator/spec_validator.py` | Validate specs |
| `graph/graph_engine.py` | Build dependency graphs |
| `graph/substrate_resolver.py` | Resolve substrate chains |
| `splitter/spec_splitter.py` | Decompose monolith specs |
| `splitter/clustering.py` | Community detection algorithm |
| `drift/drift_detector.py` | Compare code vs spec |
| `grpc/server.py` | gRPC server setup |
| `grpc/services.py` | gRPC service implementations |

### Storage Modules

| Module | Responsibility |
|--------|-----------------|
| `store/postgres_index.py` | PostgreSQL operations (SQLAlchemy) |
| `store/sqlite_index.py` | SQLite fallback |
| `store/markdown_writer.py` | Atomic markdown file updates |

### Query Modules

| Module | Responsibility |
|--------|-----------------|
| `query/fts_setup.py` | FTS5 / tsvector setup and indexing |
| `query/ranking.py` | Relevance ranking algorithms |

### Configuration

| File | Responsibility |
|------|-----------------|
| `config.py` | Load env vars, merge with config.yaml |
| `.env` | Local development secrets (not committed) |

---

## Appendix: Revision History

```
| Version | Date       | Author                  | Changes                              |
|---------|------------|-------------------------|--------------------------------------|
| 1.0.0   | Mar 2026   | Divyendu Deepak Singh   | Merged Phase 1 + Phase 2 Contracts   |
```
