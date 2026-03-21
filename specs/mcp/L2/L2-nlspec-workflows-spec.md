# nlspec Server — Layer 2: Realization & Workflows

> **Version:** 1.0.0
> **Author:** Divyendu Deepak Singh
> **Date:** March 2026
> **Type:** COMPONENT
> **Language Target:** Python
> **Status:** Draft
> **Layer:** 2-Realization
> **DERIVES FROM:** L1-nlspec-server-spec

---

## Table of Contents

1. [Abstract](#abstract)
2. [Build Order](#build-order)
3. [Architecture: Data Flows](#architecture-data-flows)
4. [Scenarios](#scenarios)
5. [Maintenance Workflow](#maintenance-workflow)
6. [Build and Run](#build-and-run)

---

## Abstract

Layer 2 describes how the Layer 1 contracts are realized. This includes:

- **Data flows:** How operations move through the system
- **Scenarios:** Concrete sequences of operations that demonstrate the contracts
- **Behavioral patterns:** When operations happen, in what order, with what side effects
- **Bootstrap workflows:** Simple CRUD + search scenarios (Phase 1)
- **Advanced workflows:** Slicing, validation, patching, substrate queries (Phase 2)
- **MCP routing:** How MCP tools call core functions
- **gRPC routing:** How internal services call functions (Phase 2)

**This layer imports all definitions from Layer 1 without redefinition.**

---

## Build Order

When implementing the nlspec system, follow this sequence to ensure proper dependencies:

### Phase 1 Bootstrap Implementation Order

1. **Spec Parser (`parser/spec_parser.py`)**
   - `parse_spec()` function
   - `analyze_gaps()` function
   - Element type detection and reference extraction
   - Markdown rendering

2. **Spec Store (`store/spec_store.py`)**
   - `store_load()` function
   - `store_get()` function
   - `store_create()` function
   - `store_update()` function
   - `store_delete()` function
   - Database schema creation

3. **Query Engine (`query/query_engine.py`)**
   - `query_search()` function (FTS setup)
   - `query_references()` function
   - FTS index population

4. **MCP Tool Layer (`mcp/tools/*.py`)**
   - All 8 Phase 1 tools: init, get, list, search, create, update, delete, query
   - MCP server registration

### Phase 2 Extended Implementation Order

5. **Namespace Manager (`namespace/namespace_manager.py`)**
   - Namespace CRUD operations
   - Integration with store_load/get/create

6. **Context Slicer (`slicer/context_slicer.py`)**
   - `context_slicer()` function
   - Reference traversal (outgoing)
   - Cross-namespace imports

7. **Spec Validator (`validator/spec_validator.py`)**
   - `spec_validate()` function
   - Dangling reference checks
   - Orphaned element checks
   - Broken import detection
   - Gap analysis integration

8. **Patch Manager (`patch/patch_manager.py`)**
   - `patch_create()` function
   - `patch_list()` function
   - `patch_absorb()` function
   - Patch file I/O

9. **Graph Engine (`graph/graph_engine.py`)**
   - `build_graph()` function
   - `build_substrate_graph()` function
   - Branching point detection

10. **Substrate Resolver (`graph/substrate_resolver.py`)**
    - `resolve_substrate_chain()` function
    - Layer stack construction

11. **Spec Splitter (`splitter/spec_splitter.py`)**
    - `spec_split()` function (suggest mode)
    - Community detection (`clustering.py`)
    - Cluster identification

12. **Spec Splitter Execute (`splitter/spec_splitter.py`)**
    - `spec_split()` function (execute mode)
    - New spec file creation
    - IMPORT statement generation

13. **Drift Detector (`drift/drift_detector.py`)**
    - `detect_drift()` function
    - Code surface comparison

14. **MCP Phase 2 Tools (`mcp/tools/*.py`)**
    - All 13 Phase 2 tools: import, namespaces, slice, patch (3x), validate, graph, split, drift, layer (2x), substrate (2x)

15. **gRPC Service Layer (`grpc/server.py`, `grpc/services.py`)** (Optional)
    - gRPC server setup
    - Service implementations

**Key Insight:** Parser → Store → Query → Tools (Phase 1) → Namespace → Slicer → Validator → Patcher → Graph → Splitter → Drift → Tools (Phase 2).

Start with parser/store/query as the foundation. Add MCP tools once core is stable. Then add Phase 2 advanced features in order.

---

## Architecture: Data Flows

### Flow: Agent reads a SECTION (Phase 1)

```
Agent
  |
  | nlspec_get({spec_id: "kv-store", section: "Functions.1"})
  |
  v
MCP Tool Layer
  |
  | route to Core Engine
  |
  v
Spec Store (store_get)
  |
  | query PostgreSQL/SQLite
  |
  v
Database
  |
  | SELECT * FROM elements WHERE spec_id=? AND section=?
  |
  v
Spec Store
  |
  | construct Section with SpecElement[] list
  |
  v
Core Engine
  |
  | return Section
  |
  v
MCP Tool Layer
  |
  | serialize to JSON (if format="json") or Markdown (if format="markdown")
  |
  v
Agent
  |
  | receives structured FUNCTION definitions

Latency budget: < 50ms
Assumptions: Database is on same machine, query is indexed on spec_id + section
```

### Flow: Agent searches for elements by text (Phase 1)

```
Agent
  |
  | nlspec_search({query: "TTL expiration", element_type: "FUNCTION"})
  |
  v
MCP Tool Layer
  |
  | route to Core Engine
  |
  v
Query Engine
  |
  | execute PostgreSQL/SQLite FTS query
  |
  v
Database
  |
  | SELECT * FROM elements WHERE content @@ tsquery(?) LIMIT ?
  |
  v
Query Engine
  |
  | rank by relevance using similarity()
  | filter by element_type
  |
  v
Core Engine
  |
  | return SearchResult[] with match_reason and relevance
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives ranked search results

Latency budget: < 50ms
Assumptions: FTS index exists and is warm, relevance ranking is in-memory
```

### Flow: Agent searches by reference (Phase 1)

```
Agent
  |
  | nlspec_search({references: "Entry", element_type: "FUNCTION"})
  |
  v
MCP Tool Layer
  |
  | route to Core Engine
  |
  v
Query Engine
  |
  | execute structural (reference-based) query
  |
  v
Database
  |
  | SELECT * FROM elements WHERE element_id IN (
  |   SELECT source_element_id FROM references
  |   WHERE target_name="Entry" AND reference_type IN ("USES", "THROWS")
  | )
  |
  v
Query Engine
  |
  | filter by element_type="FUNCTION"
  |
  v
Core Engine
  |
  | return SearchResult[] with match_reason="USES Entry"
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives FUNCTIONs that use Entry

Latency budget: < 30ms
Assumptions: reference table is indexed on (target_name, reference_type)
```

### Flow: Agent creates a new SCENARIO (Phase 1)

```
Agent
  |
  | nlspec_create({spec_id: "kv-store", section: "Scenarios",
  |   element_type: "SCENARIO", name: "SCENARIO 50",
  |   content: "...", tags: ["SEC:Functions.1", "SMOKE"]})
  |
  v
MCP Tool Layer
  |
  | validate input (name not empty, spec_id exists, section exists)
  |
  v
Spec Store
  |
  | check: name is unique in (spec_id, section, type)
  | query Database: SELECT COUNT(*) FROM elements WHERE spec_id=? AND section=? AND name=?
  |
  v
Spec Store (parse content)
  |
  | call parse_spec on just the element content
  | extract references (USES, THROWS, SEC tags)
  |
  v
Spec Store (insert)
  |
  | INSERT INTO elements (...) VALUES (...)
  | INSERT INTO references for each USES/THROWS/SEC
  |
  v
Database
  |
  | confirm insert
  |
  v
Spec Store (markdown update)
  |
  | read spec's markdown file from disk
  | find section by name (## Scenarios or similar)
  | append new element markdown to end of section
  | write to temp file
  | atomically rename temp -> original
  |
  v
Filesystem
  |
  | confirm file exists
  |
  v
Spec Store
  |
  | return SpecElement with assigned ID
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives created element with ID

Latency budget: < 100ms
Assumptions: filesystem is local, write is atomic, database insert is fast
Side effects: markdown file is modified, git sees change (if git enabled)
```

### Flow: Agent updates an element (Phase 1)

```
Agent
  |
  | nlspec_update({element_id: "kv-store:DataModel.1:record:Entry",
  |   content: "... updated content ...", tags: ["NEW_TAG"]})
  |
  v
MCP Tool Layer
  |
  | validate input
  |
  v
Spec Store
  |
  | query: load current element
  | validate: element exists, content is parseable
  |
  v
Spec Store (parse content)
  |
  | reparse updated content for references
  |
  v
Spec Store (update database)
  |
  | UPDATE elements SET content=?, updated_at=NOW() WHERE id=?
  | DELETE FROM references WHERE source_element_id=?
  | INSERT new references
  |
  v
Database
  |
  | confirm update
  |
  v
Spec Store (markdown update)
  |
  | read markdown file
  | find element block by name and type
  | replace content atomically
  |
  v
Filesystem
  |
  | confirm write
  |
  v
Spec Store
  |
  | return updated SpecElement
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives updated element

Latency budget: < 100ms
Side effects: markdown file and database are modified
```

### Flow: Agent deletes an element (Phase 1)

```
Agent
  |
  | nlspec_delete({element_id: "kv-store:Scenarios:scenario:SCENARIO 50"})
  |
  v
MCP Tool Layer
  |
  | validate input
  |
  v
Spec Store
  |
  | query: load current element
  | validate: element exists
  |
  v
Spec Store (delete database)
  |
  | DELETE FROM elements WHERE id=?
  | DELETE FROM references WHERE source_element_id=?
  |
  v
Database
  |
  | confirm delete
  |
  v
Spec Store (markdown update)
  |
  | read markdown file
  | find element block by name and type
  | remove that block atomically
  |
  v
Filesystem
  |
  | confirm write
  |
  v
Spec Store
  |
  | return confirmation
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives deleted confirmation

Latency budget: < 100ms
Side effects: markdown file and database are modified, element is gone permanently
```

### Flow: Agent initializes a new project (Phase 1)

```
Agent
  |
  | nlspec_init({project_dir: "/tmp/test-project", spec_name: "myservice"})
  |
  v
MCP Tool Layer
  |
  | validate input: project_dir path is valid, not already a project
  |
  v
Core Engine (initialization)
  |
  | create directory structure:
  |   /tmp/test-project/
  |   /tmp/test-project/specs/
  |   /tmp/test-project/.nlspec/
  |
  v
Core Engine (database)
  |
  | initialize PostgreSQL or SQLite:
  |   - create schema (specs, elements, references tables)
  |   - create FTS index
  |
  v
Core Engine (template)
  |
  | copy bundled template to /tmp/test-project/specs/myservice-spec.md
  |
  v
Spec Store (parse)
  |
  | parse the template markdown
  | extract sections, elements, references
  |
  v
Spec Store (index)
  |
  | INSERT into database
  |
  v
Database
  |
  | confirm inserts
  |
  v
Core Engine
  |
  | return Spec with id="myservice", status="Draft"
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives initialized Spec

Latency budget: < 500ms (includes file I/O and database setup)
Side effects: filesystem and database are modified, new project created
```

### Flow: Agent imports a spec with namespaces (Phase 2)

```
Agent
  |
  | nlspec_import({namespace: "myproject", spec_id: "auth", path: "specs/auth-spec.md"})
  |
  v
MCP Tool Layer
  |
  | validate: path exists and is readable
  | validate: spec_id is valid format
  |
  v
Namespace Manager
  |
  | check if namespace "myproject" exists
  | if not: create namespace
  |
  v
Database
  |
  | INSERT INTO namespaces (name) VALUES (?) ON CONFLICT DO NOTHING
  |
  v
Spec Store
  |
  | read markdown from path
  |
  v
Spec Parser
  |
  | parse_spec(markdown, spec_id, namespace)
  | extract header, sections, elements, references, IMPORT statements
  |
  v
Spec Store (insert)
  |
  | INSERT INTO specs (namespace, spec_id, ...) VALUES (...)
  | INSERT INTO elements for each SpecElement
  | INSERT INTO references for each Reference
  |
  v
Database
  |
  | confirm inserts
  | FTS index is automatically updated
  |
  v
Spec Store
  |
  | return Spec with element_count
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives imported Spec

Latency budget: < 200ms
Assumptions: markdown file is < 5MB, database bulk insert batches elements
Side effects: database index is updated, FTS index is refreshed
Note: Original markdown file is NOT copied; only indexed. Path is stored.
```

### Flow: Agent slices context for a bug fix (Phase 2)

```
Agent
  |
  | nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  |
  v
MCP Tool Layer
  |
  | route to Core Engine
  |
  v
Context Slicer
  |
  | identify SCENARIO 7 element in auth spec
  |
  v
Spec Store
  |
  | query database for element_id matching "SCENARIO 7"
  |
  v
Context Slicer (trace outgoing references)
  |
  | depth-first traversal, max_depth=3
  | from SCENARIO 7:
  |   - extract SEC tags: [SEC:Functions.3]
  |   - find element Functions.3 (FUNCTION validate_token)
  |   - FUNCTION has USES: TokenRecord
  |   - find RECORD TokenRecord
  |   - RECORD TokenRecord has USED_BY: validate_token
  |   - FUNCTION has THROWS: AuthError
  |   - find ENUM AuthError
  | collect all: SCENARIO 7, FUNCTION validate_token, RECORD TokenRecord, ENUM AuthError
  |
  v
Spec Store
  |
  | check for IMPORT statements in any collected element
  | if found: recursively slice from imported spec
  |
  v
Context Slicer (cross-namespace traversal)
  |
  | if FUNCTION validate_token has:
  |   "IMPORT UserRecord FROM myproject/users DataModel.1"
  |   - query database for myproject/users:DataModel.1:record:UserRecord
  |   - add to slice
  |   - add myproject/users to specs_touched
  |
  v
Context Slicer (assemble result)
  |
  | order elements by section, then position
  | estimate total_lines by counting rendered markdown
  | build trace: human-readable dependency chain
  |
  v
Core Engine
  |
  | return ContextSlice
  |
  v
MCP Tool Layer
  |
  | render as markdown (if format="markdown") or JSON
  |
  v
Agent
  |
  | receives minimal context sufficient for the fix

Latency budget: < 100ms
Assumptions: max_depth is small (2-3), cross-namespace imports are few
Side effects: none (read-only)
```

### Flow: Agent validates a spec (Phase 2)

```
Agent
  |
  | nlspec_validate({namespace: "myproject", spec_id: "auth"})
  |
  v
MCP Tool Layer
  |
  | route to Core Engine
  |
  v
Spec Validator
  |
  | load Spec from database (namespace + spec_id)
  |
  v
Validator (dangling reference check)
  |
  | for each FUNCTION that USES X:
  |   query database: does X exist in same spec or imported specs?
  |   if not found: add error "dangling_reference"
  |
  v
Validator (orphaned element check)
  |
  | for each RECORD:
  |   query database: does any FUNCTION USES this RECORD?
  |   if not found: add warning "orphaned"
  |
  v
Validator (cross-spec import check)
  |
  | for each IMPORT statement:
  |   extract target_namespace, target_spec_id, target_section, element_name
  |   query database for target_namespace/target_spec_id/element
  |   if not found: add error "broken_import"
  |
  v
Validator (re-run gap detection)
  |
  | call analyze_gaps
  | include CRITICAL, WARNING, INFO gaps
  |
  v
Validator (compute is_valid)
  |
  | is_valid = (errors.len() == 0)
  |
  v
Core Engine
  |
  | return ValidationResult with errors, warnings, gaps, is_valid
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives ValidationResult with actionable feedback

Latency budget: < 150ms
Assumptions: element count is < 1000, cross-namespace queries are batched
Side effects: none (read-only)
Note: Validation never rejects a spec; it reports findings.
```

### Flow: Agent detects drift (Phase 2)

```
Agent (reads code)
  |
  | extracts code_surface: {endpoints, records, errors, configs, artifacts}
  | endpoints: [GET /auth/status, POST /auth/login, DELETE /auth/purge]
  | records: [{name: "Token", fields: [{name: "id"}, {name: "expires"}]}]
  |
  v
Agent calls nlspec_drift({namespace: "myproject", spec_id: "auth", code_surface})
  |
  v
MCP Tool Layer
  |
  | route to Drift Detector
  |
  v
Drift Detector (load spec)
  |
  | query database for all ENDPOINT elements in spec
  | query for all RECORD and field definitions
  | query for all ERROR definitions
  |
  v
Drift Detector (compare endpoints)
  |
  | spec declares: POST /auth/login, GET /auth/status
  | code has: POST /auth/login, GET /auth/status, DELETE /auth/purge
  | mismatch: DELETE /auth/purge is EXTRA_IN_CODE
  |
  v
Drift Detector (compare records)
  |
  | spec declares: RECORD Token with fields [id, expires, user_id]
  | code has: RECORD Token with fields [id, expires]
  | mismatch: user_id is MISSING_IN_CODE
  |
  v
Drift Detector (assemble DriftReport)
  |
  | clean=false (drift found)
  | drifted=[
  |   {category: "API", element: "DELETE /auth/purge", severity: "EXTRA_IN_CODE"},
  |   {category: "DATA_MODEL", element: "Token.user_id", severity: "MISSING_IN_CODE"}
  | ]
  |
  v
Core Engine
  |
  | return DriftReport
  |
  v
MCP Tool Layer
  |
  | serialize
  |
  v
Agent
  |
  | receives drift report
  | decides: update spec or update code?

Latency budget: < 150ms
Assumptions: code_surface is pre-populated, spec element count < 500
Side effects: none (read-only)
Note: Drift is advisory, not prescriptive.
```

### Flow: Agent resolves a substrate chain (Phase 2)

```
Agent
  |
  | nlspec_substrate_query({namespace: "payments", substrate: "ios"})
  |
  v
MCP Tool Layer
  |
  | route to Substrate Engine
  |
  v
Substrate Engine (load or rebuild graph)
  |
  | check cache: is substrate_graph cached for namespace?
  | if cache fresh: use it
  | else: rebuild by scanning all specs
  |
  v
Substrate Engine (build graph if needed)
  |
  | for each spec in namespace:
  |   if has Layer declaration: extract layer number, DERIVES FROM parent
  | build directed graph: parent → children
  | identify branching points: specs with 2+ children
  | for each branching point: trace L1→L4 chains
  | assign substrate IDs (spatial/temporal/feature based on structure)
  |
  v
Database
  |
  | SELECT * FROM specs WHERE namespace=? AND (layer IS NOT NULL OR derives_from IS NOT NULL)
  |
  v
Substrate Engine (cache graph)
  |
  | store in in-memory cache with TTL=300s
  |
  v
Substrate Engine (resolve chain)
  |
  | find Substrate with id="ios"
  | extract stack: [L1-contracts, L2-ios, L3-ios-prod, L4-ios-us]
  | filter to from_layer:to_layer range (default 1:4)
  |
  v
Core Engine
  |
  | return SubstrateChain with ordered specs and shared_ancestors
  |
  v
MCP Tool Layer
  |
  | serialize (optionally include_content=true means include markdown)
  |
  v
Agent
  |
  | receives linear L1→L4 chain for iOS substrate
  | can now navigate the 3D topology as a simple 1D list

Latency budget: < 100ms (< 1s if rebuilding graph)
Assumptions: max 100 specs per namespace, branching depth is 4
Side effects: none (read-only); graph may be rebuilt if cache is stale
Caching: SubstrateGraph is cached in memory with TTL (default 300s)
```

---

## Scenarios

### Scenarios.1 Bootstrap Phase (Phase 1)

#### SCENARIO 1: Init project and load spec

```
SCENARIO 1: Init project and load spec  [SEC:Functions.1] [SEC:API.1] [SMOKE]
  GIVEN:
  - Empty directory at /tmp/test-project
  WHEN:
  - Call nlspec_init({project_dir: "/tmp/test-project", spec_name: "myservice"})
  THEN:
  - /tmp/test-project/specs/myservice-spec.md exists
  - /tmp/test-project/.nlspec/index exists (PostgreSQL or SQLite)
  - Returned spec has id "myservice", status "Draft"
  - Workflow: validate input → create dirs → copy template → parse → index → return
```

#### SCENARIO 2: Parse kv-store spec and get section

```
SCENARIO 2: Parse kv-store spec and get section  [SEC:Functions.2] [SEC:API.2] [SMOKE]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_get({spec_id: "kv-store", section: "Functions.1"})
  THEN:
  - Returns Functions.1 with 5 FUNCTIONs: storage_get, storage_set, storage_delete, storage_list, storage_stats
  - Each FUNCTION has element_type "FUNCTION"
  - storage_get has references: USES Entry, THROWS KeyNotFound, THROWS KeyExpired
  - Workflow: route to Spec Store → query Database → construct Section → return JSON/markdown
```

#### SCENARIO 3: Search by reference

```
SCENARIO 3: Search by reference  [SEC:Functions.3] [SEC:API.3] [SMOKE]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_search({references: "Entry", element_type: "FUNCTION"})
  THEN:
  - Returns storage_get, storage_set, storage_delete, storage_list
  - Does NOT return storage_stats (it USES StoreStats, not Entry)
  - Each result has match_reason containing "USES Entry"
  - Workflow: Query Engine → find USES references → filter by element_type → return
```

#### SCENARIO 4: Create a new scenario

```
SCENARIO 4: Create a new scenario  [SEC:Functions.2] [SEC:API.4]
  GIVEN:
  - Project with kv-store-spec.md loaded
  - Scenarios section exists
  WHEN:
  - Call nlspec_create({spec_id: "kv-store", section: "Scenarios", element_type: "SCENARIO",
    name: "SCENARIO 50: Empty list", content: "...", tags: ["SEC:Functions.1", "SMOKE"]})
  - Call nlspec_list({spec_id: "kv-store", element_type: "SCENARIO"})
  THEN:
  - Create returns element with id "kv-store:Scenarios:scenario:SCENARIO 50"
  - List includes SCENARIO 50
  - kv-store-spec.md file contains new scenario text
  - Workflow: validate → check unique → parse content → insert database → update markdown atomically → return
```

#### SCENARIO 5: Update an element

```
SCENARIO 5: Update an element  [SEC:Functions.2] [SEC:API.4]
  GIVEN:
  - Entry RECORD exists in kv-store spec
  WHEN:
  - Call nlspec_update({element_id: "kv-store:DataModel.1:record:Entry", content: "..."})
  THEN:
  - Returned element has updated content
  - kv-store-spec.md file reflects the change
  - Database index is updated
  - Workflow: query current → validate → reparse content → update database → update markdown → return
```

#### SCENARIO 6: List with filters

```
SCENARIO 6: List with filters  [SEC:Functions.2] [SEC:API.2]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_list({spec_id: "kv-store", element_type: "SCENARIO", tags: ["SMOKE"]})
  THEN:
  - Returns only scenarios tagged [SMOKE]
  - Workflow: Query Engine → filter by spec_id, element_type, tags → return
```

#### SCENARIO 7: Text search across specs

```
SCENARIO 7: Text search across specs  [SEC:Functions.3] [SEC:API.3]
  GIVEN:
  - Project with kv-store-spec.md loaded
  WHEN:
  - Call nlspec_search({query: "TTL expiration"})
  THEN:
  - Returns SCENARIO 5 (mentions TTL expiration)
  - Returns FUNCTION storage_get (mentions expired entries)
  - Results are ranked by relevance
  - Workflow: Database FTS query → rank by similarity → return SearchResult[]
```

### Scenarios.2 Phase 2 Advanced Operations

#### SCENARIO 8: System imports its own specs

```
SCENARIO 8: System imports its own specs  [SEC:API.5] [SMOKE]
  GIVEN: nlspec MCP server is running with no specs loaded
  WHEN:
    Agent calls nlspec_import({namespace: "nlspec", spec_id: "l1-bootstrap", path: "specs/L1/L1-nlspec-server-spec.md"})
    Agent calls nlspec_import({namespace: "nlspec", spec_id: "l2-workflows", path: "specs/L2/L2-nlspec-workflows-spec.md"})
  THEN:
  - nlspec_namespaces returns [{name: "nlspec", specs: ["l1-bootstrap", "l2-workflows"]}]
  - nlspec_get({namespace: "nlspec", spec_id: "l1-bootstrap", section: "DataModel.1"}) returns Spec record
  - nlspec_search({namespace: "nlspec", query: "ContextSlice"}) finds the record in extended specs (if imported)
  - Workflow: read file → parse → index → update namespace → return
```

#### SCENARIO 9: Slice for a scenario

```
SCENARIO 9: Slice for a scenario  [SEC:Functions.4] [SEC:API.6]
  GIVEN: A spec "myproject/auth" with:
    - RECORD TokenRecord
    - FUNCTION validate_token (USES: TokenRecord, THROWS: AuthError)
    - ERROR AuthError definition
    - SCENARIO 7 tagged [SEC:Functions.3]
  WHEN: Agent calls nlspec_slice({namespace: "myproject", spec_id: "auth", scenario: 7})
  THEN:
  - slice.trigger is "SCENARIO 7"
  - slice.elements contains: SCENARIO 7, FUNCTION validate_token, RECORD TokenRecord, AuthError
  - slice.specs_touched is ["myproject/auth"]
  - Workflow: find element → trace outgoing refs → collect elements → order by section → return ContextSlice
```

#### SCENARIO 10: Patch lifecycle

```
SCENARIO 10: Patch lifecycle  [SEC:Functions.5] [SEC:API.7]
  GIVEN: "myproject/auth" spec is imported, version "0.1.0"
  WHEN:
    Agent calls nlspec_patch_create({
      namespace: "myproject", spec_id: "auth",
      category: "B", sections: ["Functions.3"], scenarios: [7],
      description: "validate_token does not check expiry",
      content: "FUNCTION validate_token: add step 3: check token.expires_at > now()"
    })
  THEN: Returns patch with id "PATCH-001", status "ACTIVE"

  WHEN: Agent calls nlspec_patch_list({namespace: "myproject", spec_id: "auth"})
  THEN: Returns [PATCH-001] with status ACTIVE

  WHEN: Agent calls nlspec_patch_absorb({patch_id: "PATCH-001"})
  THEN:
  - Patch status is "ABSORBED"
  - Spec version is "0.1.1"
  - Workflow: create → list → absorb (store_update on spec) → return
```

#### SCENARIO 11: Detect dangling reference

```
SCENARIO 11: Detect dangling reference  [SEC:Functions.6] [SEC:API.8]
  GIVEN: A spec where FUNCTION foo USES: BarRecord, but no RECORD BarRecord exists
  WHEN: Agent calls nlspec_validate
  THEN: result.is_valid is false
  AND: errors contains {error_type: "dangling_reference", message contains "BarRecord"}
  - Workflow: load spec → for each USES → query database → collect missing → return error
```

#### SCENARIO 12: Detect spatial substrates from DERIVES FROM graph

```
SCENARIO 12: Detect spatial substrates from DERIVES FROM graph  [SEC:Functions.11] [SEC:API.13]
  GIVEN: Namespace "payments" has:
    - L1 spec "l1-contracts"
    - L2 spec "l2-web" (DERIVES FROM: l1-contracts)
    - L2 spec "l2-ios" (DERIVES FROM: l1-contracts)
    - L3 spec "l3-web-prod" (DERIVES FROM: l2-web)
  WHEN: Agent calls nlspec_substrate_graph({namespace: "payments"})
  THEN:
  - graph.substrates has 2 entries (web, ios)
  - Each substrate.type is "spatial"
  - Workflow: scan specs → extract DERIVES FROM → identify branching → assign substrates → return SubstrateGraph
```

#### SCENARIO 13: Resolve linear substrate chain for an agent

```
SCENARIO 13: Resolve linear substrate chain for an agent  [SEC:Functions.11] [SEC:API.13]
  GIVEN: SCENARIO 12 namespace is set up
  WHEN: Agent calls nlspec_substrate_query({namespace: "payments", substrate: "web"})
  THEN:
  - chain.substrate_id is "web"
  - chain.chain has 3 entries: l1-contracts, l2-web, l3-web-prod
  - chain.shared_ancestors includes l1-contracts
  - Workflow: load/rebuild substrate graph → find substrate → extract chain → return SubstrateChain
```

---

## Maintenance Workflow

### Bug Categories

| Category | Spec was... | Action |
|----------|------------|--------|
| A: Spec Deficiency | Wrong/ambiguous | Fix spec, add scenario, version bump |
| B: Implementation Error | Correct | Write patch, agent fixes code |
| C: Missing Capability | Incomplete | Add to spec, add scenarios, version bump |

### Self-Maintenance

Patches to the nlspec specs follow the standard workflow:

1. Agent identifies issue: "nlspec_search doesn't filter by tags correctly"
2. Agent calls `nlspec_patch_create({...})`
3. Issue is tracked as PATCH-NNN in patches/nlspec/{spec_id}/PATCH-NNN.md
4. Agent fixes implementation and spec
5. Agent calls `nlspec_patch_absorb({patch_id: "PATCH-NNN"})`
6. Spec version is bumped, patch is marked ABSORBED

### Scenario Tiers

- **[SMOKE]:** Quick sanity checks (SCENARIOS 1, 2, 3, 8, 9)
- **[SEC:x.x]:** Section coverage (each SCENARIO is tagged with sections it tests)
- **[FULL]:** Round-trip and persistence checks

---

## Build and Run

### Prerequisites

- Python 3.11+
- PostgreSQL 13+ (Phase 2; SQLite for Phase 1)
- uv (recommended) or pip

### Install

```bash
uv pip install -e ".[dev]"
```

### Test

```bash
pytest                                    # all tests
pytest tests/smoke/                       # smoke only
pytest -k "scenario_01"                   # specific scenario
```

### Run as MCP Server (Phase 1, stdio)

```bash
nlspec-server --specs-dir ./specs
```

### Run as MCP Server (Phase 2, with PostgreSQL)

```bash
export DATABASE_URL="postgresql://user:pass@localhost:5432/nlspec"
nlspec-server --specs-dir ./specs
```

### Bootstrap Sequence (Phase 1)

```bash
# 1. Start the server
nlspec-server --specs-dir ./specs

# 2. In Claude Code / MCP client:
nlspec_init({project_dir: ".", spec_name: "myservice"})

# 3. Verify functionality
nlspec_list({})
nlspec_search({query: "FUNCTION"})
nlspec_get({spec_id: "myservice", section: "Functions.1"})
```

### Self-Bootstrap Sequence (Phase 1 + 2)

```bash
# 1. Start server
nlspec-server --specs-dir ./specs

# 2. In Claude Code / MCP client, import all specs:
nlspec_import({namespace: "nlspec", spec_id: "l1-bootstrap", path: "specs/L1/L1-nlspec-server-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "l2-workflows", path: "specs/L2/L2-nlspec-workflows-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "l3-config", path: "specs/L3/L3-nlspec-config.md"})

# 3. Verify (Phase 1 only):
nlspec_validate({namespace: "nlspec", spec_id: "l1-bootstrap"})

# Both should return is_valid=true
```

---

## Appendix: Revision History

```
| Version | Date       | Author                  | Changes                              |
|---------|------------|-------------------------|--------------------------------------|
| 1.0.0   | Mar 2026   | Divyendu Deepak Singh   | Merged Phase 1 + Phase 2 Workflows   |
```
