# nlspec Bootstrap — Natural Language Specification

> **Version:** 0.1.0
> **Author:** Divyendu Deepak Singh
> **Date:** February 2026
> **Language Target:** TypeScript
> **Status:** Draft

---

## How to Use This Spec

This document is a **complete, implementable specification**. Hand it to a coding agent
(Claude Code, Cursor, Codex, etc.) and it should produce a working system with no
additional instructions. Every RECORD, FUNCTION, and SCENARIO is precise enough to
code from. If you find ambiguity, the spec is incomplete — fix the spec, not the code.

**Bootstrap context:** This is a minimal extraction of the full nlspec spec.
It implements only the Spec Parser, Spec Store, Query Engine, and MCP Server with
CRUD + search tools. Once running, the full nlspec spec becomes the first project
managed by this server. Additional capabilities (context slicer, patch manager,
validation, graph) are added later as specs managed by the running system.

**Build order:**
1. Implement this bootstrap spec → produces a working MCP server
2. Load the full nlspec spec into the running server
3. Use the server to manage development of remaining components
4. The system builds itself

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

nlspec-bootstrap is the core MCP server that parses nlspec markdown files into
structured, queryable elements and exposes CRUD + search operations as MCP tools.
An AI coding agent connects via MCP and can read, create, update, delete, and search
spec elements without parsing raw markdown itself.

This is the first of two specs that define the nlspec system:
- **This spec (bootstrap)** — parser, store, query engine, 8 CRUD tools
- **specs/mcp-server-spec.md** — imports from this spec, adds context slicing,
  patch management, validation, graph operations, namespaces, and `nlspec_import`

Build order: implement this spec first, then implement mcp-server-spec on top of it.
Once both are running, `nlspec_import` both specs into the running system — the
system then manages itself.

The expected artifact is an npm package `@nlspec/server` that runs as an MCP server
via stdio transport.

---

## 2. Problem Statement

### Current State
AI coding agents receive specs as raw markdown. The agent must manually parse section
headers, follow cross-references, and re-read files on every interaction. There is no
structured access. "Find all FUNCTIONs that reference Entry" requires the agent to
grep or scan the entire file.

### Deficiency
No structured CRUD. No search. No indexing. Every agent interaction re-parses the
same markdown. Cross-spec references (`IMPORT X FROM other-spec.md Section 4.2`)
require the agent to open files and navigate manually.

### Target State
An MCP server that parses nlspec markdown into indexed, queryable elements. The agent
calls `nlspec_get` to read a FUNCTION, `nlspec_search` to find elements by name,
type, or reference, and `nlspec_create` to add a new SCENARIO — all via native MCP
tool calls.

### Key Insight
nlspec markdown has implicit structure: RECORDs reference FUNCTIONs via USED BY,
FUNCTIONs declare USES and THROWS, SCENARIOs tag sections with [SEC:x.x]. Parsing
this structure into an indexed store turns a flat document into a queryable database
while keeping human-readable markdown as the source of truth.

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
|  +-------------------+  +-------------------+             |
|  |  MCP Tool Layer   |  |  CLI Adapter      |             |
|  |  8 tools          |  |  (same functions) |             |
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
|            |  | Query Engine   |  | Text + structural      |
|            |  +----------------+  | search                 |
|            |                      |                        |
|            +----------+-----------+                        |
|                       |                                    |
|            +----------v-----------+                        |
|            |  Persistence         |                        |
|            |  - .md files (truth) |                        |
|            |  - SQLite (index)    |                        |
|            +----------------------+                        |
+-----------------------------------------------------------+
```

### 3.1 Component Inventory

### Component: MCP Server
- **Responsibility:** Expose nlspec operations as MCP tools
- **Owns:** MCP transport (stdio), tool registration, request routing
- **Calls:** Core Engine
- **Called by:** MCP clients
- **Lifecycle:** Started as subprocess by MCP client, runs for session duration

### Component: Spec Parser
- **Responsibility:** Parse nlspec markdown into structured SpecElement tree
- **Owns:** Parsing rules, element type detection, reference extraction
- **Calls:** Nothing (pure function)
- **Called by:** Spec Store (on ingest)
- **Lifecycle:** Stateless, invoked per parse

### Component: Spec Store
- **Responsibility:** CRUD operations on SpecElements, persistence to markdown + SQLite
- **Owns:** The SpecElement collection, SQLite index, markdown files
- **Calls:** Spec Parser (to parse on load), filesystem
- **Called by:** Core Engine
- **Lifecycle:** Initialized on server start, persists across requests

### Component: Query Engine
- **Responsibility:** Search across SpecElements by type, tag, text, reference
- **Owns:** Query logic, FTS5 index
- **Calls:** Spec Store (to read elements)
- **Called by:** Core Engine
- **Lifecycle:** Stateless per query

### 3.2 Data Flows

### Flow: Agent reads a FUNCTION
1. Agent calls `nlspec_get({spec_id: "kv-store", section: "5.1"})`
2. MCP Server routes to Core Engine
3. Core Engine calls Spec Store → SQLite lookup → returns Section 5.1
4. Returns structured JSON with all FUNCTIONs in that section

Latency budget: < 50ms

### Flow: Agent searches for elements
1. Agent calls `nlspec_search({query: "Entry", element_type: "FUNCTION"})`
2. MCP Server routes to Core Engine
3. Core Engine calls Query Engine → FTS5 + structural search
4. Returns ranked results

Latency budget: < 50ms

### Flow: Agent creates a new SCENARIO
1. Agent calls `nlspec_create({spec_id: "kv-store", section: "10", element_type: "SCENARIO", ...})`
2. Core Engine validates, inserts into SQLite index
3. Core Engine appends to markdown file atomically (write temp, rename)
4. Returns created element

Latency budget: < 100ms

---

## 4. Data Model

### 4.1 Core Records

```
RECORD Spec:
  id           : String              -- unique identifier, kebab-case (e.g., "kv-store")
  name         : String              -- display name (e.g., "KV Store")
  version      : String              -- semver (e.g., "0.1.0")
  tags         : List<String>        -- freeform tags (e.g., ["software"], ["quality", "security"])
  path         : String              -- filesystem path to the markdown file
  sections     : List<Section>       -- ordered list of top-level sections
  metadata     : SpecMetadata        -- author, date, language, status
  imports      : List<SpecImport>    -- specs this spec imports from (many-to-many)
  imported_by  : List<SpecImport>    -- specs that import this one (computed)
  created_at   : Timestamp
  updated_at   : Timestamp

  USED BY: store_load, store_get, mcp_init, mcp_list
```

```
RECORD SpecImport:
  from_spec    : String              -- spec that declares the import
  to_spec      : String              -- spec being imported
  elements     : List<String>        -- which elements are imported (empty = entire API)
  relationship : String              -- freeform label (e.g., "uses API", "tests", "deploys on")

  USED BY: store_load, mcp_search
```

**Notes on spec relationships:**
- Specs form a directed graph, not a tree. Any spec can import from any other.
- Cycles are allowed but warned about. There is no fixed hierarchy or layer count.
- The 5-layer pattern from NLSPEC-SYSTEM.md is one example taxonomy, not a constraint.
- `imports` and `imported_by` are symmetric.

**Invariants:**
- id is unique across the store
- version is valid semver
- path points to an existing .md file
- sections are ordered by section number

```
RECORD Section:
  id           : String              -- "{spec_id}:{section_number}" (e.g., "kv-store:5.1")
  spec_id      : String              -- parent spec
  number       : String              -- section number (e.g., "5.1", "10")
  title        : String              -- section title
  elements     : List<SpecElement>   -- parsed elements in this section
  raw_markdown : String              -- original markdown of this section

  USED BY: store_get_section, mcp_get
```

```
RECORD SpecElement:
  id           : String              -- "{spec_id}:{section}:{type}:{name}"
                                     -- e.g., "kv-store:4.1:record:Entry"
  spec_id      : String              -- parent spec
  section      : String              -- section number this element lives in
  element_type : ElementType         -- what kind of element
  name         : String              -- element name (e.g., "Entry", "storage_get")
  content      : String              -- the element's parsed content as text
  tags         : List<String>        -- [SEC:x.x], [SMOKE], [FULL], custom tags
  references   : List<Reference>     -- outgoing: USES, THROWS, IMPORT
  referenced_by: List<Reference>     -- incoming: USED BY (computed)
  raw_markdown : String              -- original markdown of this element

  USED BY: store_crud, query_search, mcp_get, mcp_create, mcp_update, mcp_delete
```

**Invariants:**
- id is globally unique
- references and referenced_by are symmetric
- raw_markdown is in sync with content

```
RECORD Reference:
  from_element : String              -- source element ID
  to_element   : String              -- target element name or ID
  ref_type     : ReferenceType       -- USES, THROWS, IMPORT, TAGGED, USED_BY
  target_spec  : String | None       -- if cross-spec import
  target_section: String | None      -- if cross-spec import

  USED BY: query_references, store_create, store_delete
```

```
RECORD SpecMetadata:
  author       : String
  date         : String
  language     : String
  status       : String              -- Draft | Review | Approved | Implementing | Complete

  USED BY: store_load, mcp_init
```

```
RECORD SearchResult:
  element      : SpecElement
  score        : f64                 -- 0.0 - 1.0
  match_reason : String              -- why this matched

  USED BY: query_search, mcp_search
```

### 4.2 Enumerations

```
ENUM ElementType:
  RECORD
  FUNCTION
  ENDPOINT
  SCENARIO
  ENUM
  ALIAS
  CONFIG
  IMAGE
  MANIFEST
  INFRA
  PIPELINE                           -- CI/CD pipeline definition
  TOPOLOGY
  CONTRACT
  FAILURE_MODE
  IMPORT
  PROSE                              -- unstructured text
```

```
ENUM ReferenceType:
  USES                               -- function uses a record
  USED_BY                            -- record is used by a function (computed)
  THROWS                             -- function throws an error
  IMPORTS                            -- cross-spec import
  TAGGED                             -- scenario tags a section [SEC:x.x]
```

### 4.3 Type Aliases

```
ALIAS Timestamp = i64               -- Unix milliseconds, UTC
ALIAS ElementId = String            -- "{spec_id}:{section}:{type}:{name}"
ALIAS SpecId = String               -- kebab-case identifier
ALIAS SectionNumber = String        -- e.g., "5.1", "10"
```

---

## 5. Core Functions

### 5.1 Spec Parser

```
FUNCTION parse_spec(markdown: String, spec_id: SpecId) -> Spec
  USES: Spec, Section, SpecElement, Reference, SpecMetadata
  THROWS: ParseError

  PRECONDITIONS:
  - markdown is valid UTF-8

  POSTCONDITIONS:
  - Returned Spec contains all sections and elements found in the markdown
  - Every element definition is a SpecElement
  - References (USES, THROWS, IMPORT, [SEC:] tags) are extracted
  - raw_markdown is preserved on every element

  BEHAVIOR:
  1. Parse header block (Version, Author, Date, Language, Status) into SpecMetadata
  2. Split markdown by "## {N}." section headers into Sections
  3. For each section, detect and parse elements by pattern matching:
     a. "RECORD {Name}:" → RECORD
     b. "FUNCTION {name}(" → FUNCTION
     c. "SCENARIO {N}:" → SCENARIO
     d. "ENUM {Name}:" → ENUM
     e. "ALIAS {Name} =" → ALIAS
     f. "CONFIG {name}:" → CONFIG
     g. "ENDPOINT {METHOD} {path}" → ENDPOINT
     h. "IMAGE {name}:" → IMAGE
     i. "MANIFEST {name}:" → MANIFEST
     j. "INFRA {name}:" → INFRA
     k. "PIPELINE {name}:" → PIPELINE
     l. "TOPOLOGY:" → TOPOLOGY
     m. "CONTRACT:" → CONTRACT
     n. "FAILURE:" → FAILURE_MODE
     o. "IMPORT {Name} FROM" → IMPORT
     p. Everything else → PROSE
  4. For each element, extract references:
     a. "USES: A, B, C" → Reference(USES, A), Reference(USES, B), Reference(USES, C)
     b. "THROWS: X, Y" → Reference(THROWS, X), Reference(THROWS, Y)
     c. "USED BY: f1, f2" → Reference(USED_BY, f1), Reference(USED_BY, f2)
     d. "[SEC:5.3]" in tags → Reference(TAGGED, "5.3")
     e. "IMPORT X FROM spec.md Section 4.2" → Reference(IMPORTS, X, target_spec, target_section)
  5. Assign element IDs: "{spec_id}:{section}:{type}:{name}"
  6. Build and return the Spec

  ERRORS:
  - ParseError: malformed element definition. Include line number and context.
    Malformed elements become PROSE, do not reject the entire file.

  PERFORMANCE:
  - Target: < 100ms for 3000 lines
  - Memory: proportional to file size

  NOTES:
  - Element detection patterns match at start of line within a code fence block
  - Parser is lenient: unknown patterns become PROSE
  - raw_markdown is preserved exactly for round-tripping
  - Parser does NOT validate that references resolve — that's validation (future)
```

```
FUNCTION render_elements(elements: List<SpecElement>) -> String
  USES: SpecElement
  THROWS: (none)

  BEHAVIOR:
  1. For each element, output its raw_markdown
  2. Concatenate with appropriate section headers and separator lines
  3. Return the rendered markdown

  NOTES:
  - Inverse of parsing: elements → readable markdown
  - Must produce markdown that re-parses to identical elements (round-trip)
```

### 5.2 Spec Store

```
FUNCTION store_load(project_dir: String) -> List<Spec>
  USES: Spec
  THROWS: IOError, ParseError

  BEHAVIOR:
  1. Scan project_dir/specs/ recursively for *.md files
  2. Also scan project_dir root for *-spec.md files
  3. Parse each file with parse_spec (derive spec_id from filename: "kv-store-spec.md" → "kv-store")
  4. Build SQLite index of all elements and references
  5. Compute referenced_by (inverse of references) across all specs
  6. Return list of loaded specs

  NOTES:
  - SQLite index is a cache, markdown files are source of truth
  - If index exists and is newer than all .md files, skip re-parsing
  - If any .md file is newer, re-parse only that file
  - spec_id derivation: strip "-spec.md" suffix, kebab-case the remainder
```

```
FUNCTION store_get(element_id: ElementId) -> SpecElement
  USES: SpecElement
  THROWS: NotFound

  BEHAVIOR:
  1. Look up element_id in SQLite
  2. If not found, throw NotFound
  3. Return SpecElement

  PERFORMANCE: < 5ms
```

```
FUNCTION store_get_section(spec_id: SpecId, section: SectionNumber) -> Section
  USES: Section
  THROWS: NotFound

  BEHAVIOR:
  1. Look up spec_id + section in SQLite
  2. Return Section with all its elements

  PERFORMANCE: < 10ms
```

```
FUNCTION store_create(element: SpecElement) -> SpecElement
  USES: SpecElement
  THROWS: AlreadyExists, InvalidElement, IOError

  BEHAVIOR:
  1. Validate: element has required fields, section exists in the spec
  2. Check element ID doesn't already exist
  3. Insert into SQLite index
  4. Append to markdown file at end of the target section (before next "## " header)
  5. Write atomically (temp file + rename)
  6. Update referenced_by for any elements this one references
  7. Return created element

  NOTES:
  - Markdown append: find the line "## {next_section_number}." or EOF,
    insert the element's rendered markdown before that line
```

```
FUNCTION store_update(element_id: ElementId, content: String | None, tags: List<String> | None, name: String | None) -> SpecElement
  USES: SpecElement
  THROWS: NotFound, IOError

  BEHAVIOR:
  1. Look up existing element
  2. Apply updates (only provided fields)
  3. Regenerate raw_markdown from updated content
  4. Update SQLite index
  5. In markdown file: find old raw_markdown block, replace with new
  6. Write atomically
  7. Recompute references from new content
  8. Update referenced_by
  9. Return updated element

  NOTES:
  - Finding the old block: match the element's raw_markdown in the file
  - If raw_markdown doesn't match (file was externally edited), re-parse the file first
```

```
FUNCTION store_delete(element_id: ElementId, force: bool) -> SpecElement
  USES: SpecElement
  THROWS: NotFound, ReferenceConflict, IOError

  BEHAVIOR:
  1. Look up element
  2. If force is false and element has referenced_by entries, throw ReferenceConflict
  3. Remove from SQLite index
  4. Remove from markdown file (find raw_markdown block, delete it)
  5. Write atomically
  6. Clean up referenced_by on elements this one referenced
  7. Return deleted element
```

```
FUNCTION store_list(spec_id: SpecId | None, element_type: ElementType | None, section: SectionNumber | None, tags: List<String> | None) -> List<SpecElement>
  USES: SpecElement
  THROWS: (none)

  BEHAVIOR:
  1. Query SQLite with optional filters (all are AND)
  2. Return matching elements ordered by section number, then position
  
  NOTES:
  - All parameters optional. No filters = return everything.
  - Tags filter is exact match: tags=["SMOKE"] returns elements tagged [SMOKE]
```

### 5.3 Query Engine

```
FUNCTION query_search(query: String, spec_id: SpecId | None, element_type: ElementType | None, tags: List<String> | None, references: String | None, limit: u64) -> List<SearchResult>
  USES: SearchResult, SpecElement
  THROWS: (none)

  BEHAVIOR:
  1. If query is non-empty, run FTS5 full-text search over element names + content + raw_markdown
  2. Apply filters: spec_id, element_type, tags (all AND)
  3. If references is set, find elements whose references contain that name
     (e.g., references="Entry" → elements that USES Entry)
  4. Rank by relevance (FTS5 rank + exact name match boost)
  5. Return top `limit` results

  PERFORMANCE:
  - Target: < 50ms for a project with 10 specs, 1000 elements

  NOTES:
  - Empty query with only filters is valid (equivalent to store_list but ranked)
  - references filter checks both element.references[].to_element and
    element.referenced_by[].from_element
```

```
FUNCTION query_references(element_id: ElementId, direction: "outgoing" | "incoming" | "both") -> List<Reference>
  USES: Reference
  THROWS: NotFound

  BEHAVIOR:
  1. Look up element
  2. Return references (outgoing), referenced_by (incoming), or both
```

---

## 6. API Surface — MCP Tools

Eight MCP tools. Each takes JSON parameters, returns JSON results.

**Namespace support:** All tools accept an optional `namespace` parameter (String).
When provided, operations are scoped to that namespace. When omitted, the "default"
namespace is used. The namespace system is defined in specs/mcp-server-spec.md.
Bootstrap tools work without namespaces — they become namespace-aware when the
MCP server spec is also implemented.

### 6.1 Project Management

```
MCP TOOL: nlspec_init
  DESCRIPTION: Initialize a new nlspec project or add a spec to an existing project

  PARAMETERS:
    project_dir  : String              -- path to project root
    spec_name    : String              -- spec identifier (kebab-case)
    language     : String | None       -- target language (default: "TypeScript")

  RETURNS:
    spec         : Spec
    path         : String              -- path to created file

  BEHAVIOR:
  1. Create specs/ directory if not exists
  2. Copy nlspec template to specs/{spec_name}-spec.md with placeholders replaced
  3. Create .nlspec/ directory and empty SQLite index
  4. Parse the new file and load into store
  5. Return spec record and file path
```

### 6.2 Read Operations

```
MCP TOOL: nlspec_get
  DESCRIPTION: Read a specific element or section from a spec

  PARAMETERS:
    element_id   : ElementId | None    -- direct element lookup
    spec_id      : SpecId | None       -- spec to look in (required with section)
    section      : SectionNumber | None -- section number
    format       : "json" | "markdown" -- default: "markdown"

  RETURNS:
    element      : SpecElement | None  -- if element_id given
    section_data : Section | None      -- if section given (renamed to avoid collision)
    markdown     : String | None       -- if format is "markdown"

  EXAMPLES:
    nlspec_get({spec_id: "kv-store", section: "5.1"})
    → Returns Section 5.1 with all FUNCTIONs, rendered as markdown

    nlspec_get({element_id: "kv-store:4.1:record:Entry"})
    → Returns the Entry RECORD definition

    nlspec_get({spec_id: "kv-store", section: "5.1", format: "json"})
    → Returns Section 5.1 as structured JSON with all element fields
```

```
MCP TOOL: nlspec_list
  DESCRIPTION: List specs, sections, or elements with filters

  PARAMETERS:
    spec_id      : SpecId | None
    element_type : ElementType | None
    section      : SectionNumber | None
    tags         : List<String> | None
    limit        : u64                 -- default: 100

  RETURNS:
    items        : List<SpecElement>
    total        : u64

  EXAMPLES:
    nlspec_list({})
    → Returns all elements across all specs (up to 100)

    nlspec_list({spec_id: "kv-store", element_type: "SCENARIO", tags: ["SMOKE"]})
    → Returns SMOKE-tagged scenarios in kv-store spec

    nlspec_list({element_type: "RECORD"})
    → Returns all RECORDs across all specs
```

### 6.3 Search

```
MCP TOOL: nlspec_search
  DESCRIPTION: Text and structural search across specs

  PARAMETERS:
    query        : String              -- text to search for
    spec_id      : SpecId | None
    element_type : ElementType | None
    tags         : List<String> | None
    references   : String | None       -- find elements referencing this name
    limit        : u64                 -- default: 20

  RETURNS:
    results      : List<SearchResult>

  EXAMPLES:
    nlspec_search({query: "TTL expiration"})
    → Returns elements mentioning TTL expiration, ranked by relevance

    nlspec_search({references: "Entry", element_type: "FUNCTION"})
    → Returns all FUNCTIONs that USES Entry

    nlspec_search({tags: ["SEC:5.1"], element_type: "SCENARIO"})
    → Returns all SCENARIOs that test Section 5.1
```

### 6.4 Write Operations

```
MCP TOOL: nlspec_create
  DESCRIPTION: Add a new element to a spec

  PARAMETERS:
    spec_id      : SpecId
    section      : SectionNumber
    element_type : ElementType
    name         : String
    content      : String              -- element content in nlspec markdown format
    tags         : List<String> | None

  RETURNS:
    element      : SpecElement         -- created element with assigned ID

  EXAMPLE:
    nlspec_create({
      spec_id: "kv-store",
      section: "10",
      element_type: "SCENARIO",
      name: "SCENARIO 50: Empty store list",
      content: "SCENARIO 50: Empty store list  [SEC:5.1] [SMOKE]\n  GIVEN:\n  - Store is empty\n  WHEN:\n  - GET /kv?limit=100\n  THEN:\n  - Returns 200 with keys: [], count: 0",
      tags: ["SEC:5.1", "SMOKE"]
    })
```

```
MCP TOOL: nlspec_update
  DESCRIPTION: Modify an existing element

  PARAMETERS:
    element_id   : ElementId
    content      : String | None
    tags         : List<String> | None
    name         : String | None

  RETURNS:
    element      : SpecElement
```

```
MCP TOOL: nlspec_delete
  DESCRIPTION: Remove an element from a spec

  PARAMETERS:
    element_id   : ElementId
    force        : bool                -- delete even if referenced (default: false)

  RETURNS:
    element      : SpecElement         -- the deleted element
    broken_refs  : List<Reference>     -- dangling refs if force=true
```

---

## 7. Error Model

```
NlspecError
  +-- ParseError                -- markdown parsing failure
  |     +-- MalformedElement    -- element definition is wrong
  |     +-- InvalidHeader       -- spec header is malformed
  +-- StoreError
  |     +-- NotFound            -- element or spec doesn't exist
  |     +-- AlreadyExists       -- element ID taken
  |     +-- InvalidElement      -- missing required fields
  |     +-- ReferenceConflict   -- can't delete referenced element
  +-- IOError                   -- filesystem failure
  +-- ConfigError               -- invalid configuration
```

**Handling rules:**
- ParseError: malformed elements become PROSE, log warning, continue parsing
- NotFound: return error to caller, no crash
- AlreadyExists: return error, agent can choose a different name
- ReferenceConflict: return error with list of referencing elements
- IOError: return error, never silently lose data
- All errors include a human-readable message

---

## 8. Configuration

```
CONFIG server:
  transport:
    type: String
    default: "stdio"
    env: NLSPEC_TRANSPORT
    description: MCP transport type
    validation: "stdio" | "sse"

  project_dir:
    type: String
    default: "."
    env: NLSPEC_PROJECT_DIR
    description: Root directory of the project
    validation: must exist

CONFIG store:
  index_path:
    type: String
    default: ".nlspec/index.sqlite"
    env: NLSPEC_INDEX_PATH
    description: SQLite index path relative to project_dir

  auto_reindex:
    type: bool
    default: true
    env: NLSPEC_AUTO_REINDEX
    description: Re-parse when markdown files change

CONFIG search:
  fts_enabled:
    type: bool
    default: true
    env: NLSPEC_FTS
    description: Enable FTS5 full-text search
```

---

## 9. Deployment Artifacts

```
Distribution: npm package @nlspec/server
Installation: npm install -g @nlspec/server  OR  npx @nlspec/server

MCP client configuration (claude_desktop_config.json):
{
  "mcpServers": {
    "nlspec": {
      "command": "npx",
      "args": ["@nlspec/server", "--project-dir", "/path/to/project"]
    }
  }
}

Claude Code configuration (.mcp.json in project root):
{
  "mcpServers": {
    "nlspec": {
      "command": "npx",
      "args": ["@nlspec/server"]
    }
  }
}
```

---

## 10. Scenarios

### 10.1 Happy Path

```
SCENARIO 1: Init project and load spec  [SEC:5.2] [SEC:6.1] [SMOKE]
  GIVEN:
  - Empty directory at /tmp/test-project
  WHEN:
  - Call nlspec_init({project_dir: "/tmp/test-project", spec_name: "myservice"})
  THEN:
  - /tmp/test-project/specs/myservice-spec.md exists
  - /tmp/test-project/.nlspec/index.sqlite exists
  - Returned spec has id "myservice", status "Draft"

SCENARIO 2: Parse kv-store spec and get section  [SEC:5.1] [SEC:5.2] [SEC:6.2] [SMOKE]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_get({spec_id: "kv-store", section: "5.1"})
  THEN:
  - Returns Section 5.1 with 5 FUNCTIONs: storage_get, storage_set, storage_delete,
    storage_list, storage_stats
  - Each FUNCTION has element_type "FUNCTION"
  - storage_get has references: USES Entry, THROWS KeyNotFound, THROWS KeyExpired

SCENARIO 3: Search by reference  [SEC:5.3] [SEC:6.3] [SMOKE]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_search({references: "Entry", element_type: "FUNCTION"})
  THEN:
  - Returns storage_get, storage_set, storage_delete, storage_list
  - Does NOT return storage_stats (it USES StoreStats, not Entry)
  - Each result has match_reason containing "USES Entry"

SCENARIO 4: Create a new scenario  [SEC:5.2] [SEC:6.4]
  GIVEN:
  - Project with kv-store-spec.md loaded
  - Section 10 exists with existing scenarios
  WHEN:
  - Call nlspec_create({spec_id: "kv-store", section: "10", element_type: "SCENARIO",
    name: "SCENARIO 50: Empty list returns empty", content: "SCENARIO 50: ...", tags: ["SEC:5.1"]})
  - Call nlspec_list({spec_id: "kv-store", element_type: "SCENARIO"})
  THEN:
  - Create returns element with id "kv-store:10:scenario:SCENARIO 50"
  - List includes SCENARIO 50 among the results
  - kv-store-spec.md file contains the new scenario text in Section 10

SCENARIO 5: Update an element  [SEC:5.2] [SEC:6.4]
  GIVEN:
  - Project with kv-store-spec.md loaded
  - Entry RECORD exists in Section 4.1
  WHEN:
  - Call nlspec_update({element_id: "kv-store:4.1:record:Entry",
    content: "RECORD Entry:\n  key: String\n  value: String\n  version: u64\n  new_field: bool\n\n  USED BY: storage_get, storage_set"})
  THEN:
  - Returned element has updated content including new_field
  - kv-store-spec.md file reflects the change
  - SQLite index is updated

SCENARIO 6: List with filters  [SEC:5.2] [SEC:6.2]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_list({spec_id: "kv-store", element_type: "SCENARIO", tags: ["SMOKE"]})
  THEN:
  - Returns only scenarios tagged [SMOKE]
  - Does not return scenarios without [SMOKE] tag
  - Each result has tags containing "SMOKE"

SCENARIO 7: Text search across specs  [SEC:5.3] [SEC:6.3]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_search({query: "TTL expiration"})
  THEN:
  - Returns SCENARIO 5 (mentions TTL expiration)
  - Returns FUNCTION storage_get (mentions expired entries)
  - Results are ranked by relevance (scenario with exact match scores higher)

SCENARIO 8: Get references for an element  [SEC:5.3] [SEC:6.2]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_search({references: "Entry", element_type: "FUNCTION"})
  THEN:
  - Results include storage_get, storage_set, storage_delete, storage_list
  - Each has a reference with ref_type USES and to_element "Entry"
```

### 10.2 Error Scenarios

```
SCENARIO 10: Get nonexistent element  [SEC:5.2] [SEC:7]
  GIVEN:
  - Project loaded
  WHEN:
  - Call nlspec_get({element_id: "kv-store:99:record:Nope"})
  THEN:
  - Returns NotFound error with the requested element_id

SCENARIO 11: Create duplicate  [SEC:5.2] [SEC:7]
  GIVEN:
  - Entry RECORD exists
  WHEN:
  - Call nlspec_create({spec_id: "kv-store", section: "4.1", element_type: "RECORD",
    name: "Entry", content: "..."})
  THEN:
  - Returns AlreadyExists error

SCENARIO 12: Delete referenced element  [SEC:5.2] [SEC:7]
  GIVEN:
  - Entry RECORD is referenced by storage_get, storage_set, etc.
  WHEN:
  - Call nlspec_delete({element_id: "kv-store:4.1:record:Entry", force: false})
  THEN:
  - Returns ReferenceConflict error
  - Error includes list of referencing element IDs

SCENARIO 13: Delete with force  [SEC:5.2] [SEC:7]
  GIVEN:
  - Entry RECORD is referenced
  WHEN:
  - Call nlspec_delete({element_id: "kv-store:4.1:record:Entry", force: true})
  THEN:
  - Element is deleted
  - broken_refs lists all now-dangling references
  - Markdown file no longer contains Entry RECORD

SCENARIO 14: Parse malformed element  [SEC:5.1] [SEC:7]
  GIVEN:
  - Markdown file with "RECORD BadDef" (missing colon)
  WHEN:
  - File is loaded
  THEN:
  - No crash
  - The malformed line is captured as PROSE
  - All other valid elements are parsed correctly
  - Warning is logged
```

### 10.3 Round-Trip Scenarios

```
SCENARIO 20: Parse then render produces identical markdown  [SEC:5.1] [FULL]
  GIVEN:
  - kv-store-spec.md as input
  WHEN:
  - Parse with parse_spec
  - Render all elements with render_elements
  THEN:
  - Output is byte-for-byte identical to input

  NOTES:
  - Critical test. If parsing loses content, store_update corrupts specs.
  - Round-trip failures are blocking bugs.
```

### 10.4 Persistence Scenarios

```
SCENARIO 30: Changes survive restart  [SEC:5.2] [FULL]
  GIVEN:
  - Project with kv-store-spec.md loaded
  - A new SCENARIO 50 was created via nlspec_create
  WHEN:
  - Server is stopped
  - Server is restarted on the same project_dir
  - Call nlspec_get({element_id: "kv-store:10:scenario:SCENARIO 50"})
  THEN:
  - SCENARIO 50 exists (was persisted to markdown + re-indexed on restart)

SCENARIO 31: External edit triggers re-index  [SEC:5.2] [FULL]
  GIVEN:
  - Project loaded, auto_reindex is true
  - Human manually edits kv-store-spec.md to add a new RECORD
  WHEN:
  - Agent calls nlspec_list on the spec
  THEN:
  - The manually-added RECORD appears in results
  - SQLite index was re-built from the modified markdown
```

---

## 11. Dependencies

```
DEPENDENCY @modelcontextprotocol/sdk:
  purpose: MCP server SDK (tool registration, stdio/SSE transport)
  version: latest
  required: true
  interface: library (TypeScript)
  fallback: none

DEPENDENCY better-sqlite3:
  purpose: SQLite for element index and FTS5 full-text search
  version: "11.0+"
  required: true
  interface: library
  fallback: none

DEPENDENCY chokidar:
  purpose: File system watching for auto_reindex
  version: "3.0+"
  required: false (only if auto_reindex is true)
  interface: library
  fallback: poll-based checking on each request
```

---

## 12. File Structure

```
nlspec/
├── package.json
├── tsconfig.json
├── README.md
├── specs/
│   ├── bootstrap-spec.md            -- this file
│   └── mcp-server-spec.md           -- extensions (imports from this spec)
├── CLAUDE.md
├── src/
│   ├── index.ts                     -- MCP server entry point
│   ├── cli.ts                       -- CLI entry point
│   ├── config.ts                    -- configuration loading
│   ├── models/
│   │   ├── types.ts                 -- Spec, Section, SpecElement, Reference, etc.
│   │   └── errors.ts               -- NlspecError hierarchy
│   ├── parser/
│   │   ├── spec-parser.ts           -- parse_spec
│   │   ├── element-patterns.ts      -- regex patterns for element detection
│   │   └── renderer.ts             -- render_elements
│   ├── store/
│   │   ├── spec-store.ts            -- store_load, CRUD operations
│   │   ├── sqlite-index.ts          -- SQLite schema, index operations
│   │   └── markdown-writer.ts       -- atomic markdown file updates
│   ├── query/
│   │   ├── query-engine.ts          -- query_search, query_references
│   │   └── fts-setup.ts             -- FTS5 table creation and population
│   └── mcp/
│       ├── server.ts                -- MCP server setup
│       └── tools/
│           ├── init.ts              -- nlspec_init
│           ├── get.ts               -- nlspec_get
│           ├── list.ts              -- nlspec_list
│           ├── search.ts            -- nlspec_search
│           ├── create.ts            -- nlspec_create
│           ├── update.ts            -- nlspec_update
│           └── delete.ts            -- nlspec_delete
├── tests/
│   ├── fixtures/
│   │   └── kv-store-spec.md         -- test fixture
│   ├── scenarios/
│   │   ├── scenario_01_init.test.ts
│   │   ├── scenario_02_parse.test.ts
│   │   ├── scenario_03_search.test.ts
│   │   ├── scenario_04_create.test.ts
│   │   ├── scenario_10_errors.test.ts
│   │   ├── scenario_20_roundtrip.test.ts
│   │   └── scenario_30_persist.test.ts
│   └── smoke/
│       └── smoke.test.ts            -- SCENARIOS 1, 2, 3
└── templates/
    └── nlspec-template.md            -- bundled template for nlspec_init
```

---

## 13. Maintenance Workflow

Standard nlspec maintenance process. Bug categories A/B/C, patch specs,
scenario tiers [SMOKE]/[SEC:x.x]/[FULL].

**Bootstrap lifecycle:**
1. This spec is implemented by hand (agent reads spec, produces code)
2. Once running, this spec is loaded into the running server as the first project
3. Future changes to this spec go through the server: nlspec_create, nlspec_update
4. Patches to this spec are managed by nlspec_patch_create (once that tool exists)
5. The system manages its own evolution

---

## 14. Build and Run

```
### Prerequisites
- Node.js 20+
- npm

### Build
$ npm install
$ npm run build

### Test
$ npm test                          # all tests
$ npm run test:smoke                # smoke only (SCENARIOS 1, 2, 3)

### Run as MCP Server
$ npx @nlspec/server --project-dir /path/to/project

### Run as CLI
$ npx nlspec init --name myservice
$ npx nlspec list --spec myservice --type FUNCTION
$ npx nlspec search "Entry" --type FUNCTION
$ npx nlspec get --spec myservice --section 5.1

### Verify MCP Connection
1. Add to claude_desktop_config.json:
   {"mcpServers": {"nlspec": {"command": "npx", "args": ["@nlspec/server", "--project-dir", "."]}}}
2. In Claude Desktop: "List all specs" → agent calls nlspec_list → returns results
```

---

## 15. Boundaries

```
### This System Does:
- Parse nlspec markdown into structured, indexed elements
- CRUD on elements (RECORD, FUNCTION, SCENARIO, etc.)
- Text and structural search across specs
- Reference tracking (USES, THROWS, TAGGED, IMPORTS)
- Persist to markdown (source of truth) + SQLite (index cache)
- Expose all operations as MCP tools
- Expose all operations as CLI commands
- Round-trip markdown faithfully (parse → edit → render → identical)

### This System Does NOT:
- Context slicing — FUTURE: nlspec_slice tool (traces dependency chains)
- Patch management — FUTURE: nlspec_patch_* tools (lifecycle management)
- Spec validation — FUTURE: nlspec_validate tool (dangling refs, orphans)
- Graph queries — FUTURE: nlspec_graph tool (dependency visualization)
- Code generation — agent's job, not nlspec's
- Test execution — agent's job using test runners
- Deployment — agent's job using platform tools
- Git integration — use git directly
- Semantic/AI search — search is FTS5 text + structural, not embedding-based
- Role-based filtering — convention in CLAUDE.md, not enforced
- Multi-user conflict resolution — single writer assumption

### Future Extensions (added as specs managed by this server):
- nlspec_slice: context slicing via dependency tracing
- nlspec_patch_create/list/absorb: patch lifecycle management
- nlspec_validate: structural validation (dangling refs, orphans, missing sections)
- nlspec_graph: dependency graph queries
- nlspec_describe: MCP App — interactive spec overview rendered inline in conversation
  (architecture diagram, component inventory, scenario coverage). Uses MCP Apps
  extension (ext-apps, ui:// resources with sandboxed HTML/JS).
- nlspec_visualize: MCP App — interactive diagrams (dependency graph, layer map,
  scenario heatmap, CI/CD flow). Uses D3.js in ui:// resources. Bidirectional:
  click a node to trigger nlspec_get and show the element definition.
- nlspec_diff: spec version comparison
- Semantic search via embeddings (future)
- Role-based context scoping (auto-enforced, not just convention)
```

---

## Appendix A: Glossary

```
nlspec:          Natural Language Specification — markdown document following the 15-section template
SpecElement:     Parsed, structured unit (RECORD, FUNCTION, SCENARIO, etc.)
MCP:             Model Context Protocol — agents use this to call nlspec tools
Bootstrap:       This spec is implemented first, then manages everything including itself
Round-trip:      parse → edit → render produces identical markdown
Index:           SQLite cache of parsed elements, rebuilt from markdown when stale
Source of truth: The .md files. SQLite is always rebuildable from markdown.
```

## Appendix B: Related Specs and Cross-Spec References

```
nlspec-spec.md: The full nlspec spec. This bootstrap is a subset.
  Relationship: bootstrap implements core; full spec adds slicer, patches, validation, graph.
  Once bootstrap is running, the full nlspec spec is loaded and developed incrementally.

NLSPEC-TEMPLATE.md: The template format this spec follows and that nlspec_init produces.
NLSPEC-SYSTEM.md: The 5-layer architecture that nlspec supports.
CLAUDE.md: Agent instructions referencing nlspec MCP tools.
kv-store-spec.md: Example spec used as test fixture.
```

## Appendix C: Revision History

```
| Version | Date       | Author                  | Changes                              |
|---------|------------|-------------------------|--------------------------------------|
| 0.1.0   | Feb 2026   | Divyendu Deepak Singh   | Initial bootstrap extraction         |
```
