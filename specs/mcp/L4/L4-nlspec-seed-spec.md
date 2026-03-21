# nlspec Server — Layer 4: Seed Data

> **Version:** 1.0.0
> **Author:** Divyendu Deepak Singh
> **Date:** March 2026
> **Type:** COMPONENT
> **Language Target:** Python
> **Status:** Draft
> **Layer:** 4-Seed
> **DERIVES FROM:** L3-nlspec-config

---

## Table of Contents

1. [Abstract](#abstract)
2. [Self-Bootstrap Procedure](#self-bootstrap-procedure)
3. [Pre-Loaded System Namespace](#pre-loaded-system-namespace)
4. [Seed Data Initialization](#seed-data-initialization)
5. [Deployment Checklist](#deployment-checklist)

---

## Abstract

Layer 4 defines how the nlspec system bootstraps itself: the build order, self-import procedure, and how the system comes to manage its own specifications.

This is a **short, practical** layer that answers: "How does the complete system start?"

---

## Self-Bootstrap Procedure

### Build and Deploy Order

The nlspec system is self-hosting. Each layer must be implemented before importing specs that define it.

**Stage 1: Implementation (Phase 1)**

1. Implement Phase 1 Layer 1 contracts (parser, store, query, 8 tools)
   - Coding agent reads `L1-nlspec-server-spec.md`
   - Produces working MCP server (stdio only)

2. Implement Phase 1 Layer 2 workflows
   - Coding agent reads `L2-nlspec-workflows-spec.md`
   - Documents how the 8 tools work end-to-end

3. Implement Phase 1 Layer 3 configuration
   - Coding agent reads `L3-nlspec-config.md`
   - Adds config, file structure, build procedures

**Stage 2: Self-Bootstrap Phase 1**

Once Phase 1 is implemented and running:

```bash
# Server is running with empty index
nlspec-server --specs-dir ./specs

# In Claude Code / MCP client:

# Import Layer 1
nlspec_import({
  namespace: "nlspec",
  spec_id: "l1-bootstrap",
  path: "specs/L1/L1-nlspec-server-spec.md"
})

# Import Layer 2
nlspec_import({
  namespace: "nlspec",
  spec_id: "l2-workflows",
  path: "specs/L2/L2-nlspec-workflows-spec.md"
})

# Import Layer 3
nlspec_import({
  namespace: "nlspec",
  spec_id: "l3-config",
  path: "specs/L3/L3-nlspec-config.md"
})
```

After this:
- The system has indexed itself
- All Phase 1 specs are queryable
- The system can update its own specs

**Stage 3: Implementation (Phase 2)**

4. Implement Phase 2 Layer 1 contracts (extends Phase 1 with 13 new tools)
   - Coding agent reads `L1-nlspec-server-spec.md` (merged version with Phase 2 additions)
   - Adds namespaces, validation, slicing, patching, graphs, substrate, drift, split

5. Implement Phase 2 Layer 2 workflows
   - Coding agent reads `L2-nlspec-workflows-spec.md` (merged version with Phase 2 additions)
   - Documents advanced operations

6. Implement Phase 2 Layer 3 configuration
   - Coding agent reads `L3-nlspec-config.md` (merged version with Phase 2 additions)
   - Adds gRPC, PostgreSQL, S3, git integration

**Stage 4: Self-Bootstrap Phase 2**

7. Once Phase 2 is implemented:

```bash
# Server is running with Phase 1 specs indexed

# Import Phase 2 Layer 1 (merged spec)
nlspec_import({
  namespace: "nlspec",
  spec_id: "l1-server",
  path: "specs/mcp/L1/L1-nlspec-server-spec.md"
})

# Import Phase 2 Layer 2 (merged spec)
nlspec_import({
  namespace: "nlspec",
  spec_id: "l2-workflows",
  path: "specs/mcp/L2/L2-nlspec-workflows-spec.md"
})

# Import Phase 2 Layer 3 (merged spec)
nlspec_import({
  namespace: "nlspec",
  spec_id: "l3-config",
  path: "specs/mcp/L3/L3-nlspec-config.md"
})

# Import Phase 2 Layer 4 (this spec)
nlspec_import({
  namespace: "nlspec",
  spec_id: "l4-seed",
  path: "specs/mcp/L4/L4-nlspec-seed-spec.md"
})
```

After this:
- The system manages its own complete specification
- All 8 Phase 1 + 13 Phase 2 tools are active
- Advanced operations are available

**Stage 5: Validate Self**

```bash
nlspec_validate({namespace: "nlspec", spec_id: "l1-bootstrap"})
nlspec_validate({namespace: "nlspec", spec_id: "l2-workflows"})
nlspec_validate({namespace: "nlspec", spec_id: "l3-config"})
nlspec_validate({namespace: "nlspec", spec_id: "l1-server"})
nlspec_validate({namespace: "nlspec", spec_id: "l2-workflows"})
nlspec_validate({namespace: "nlspec", spec_id: "l3-config"})
nlspec_validate({namespace: "nlspec", spec_id: "l4-seed"})
```

All should return `is_valid=true`. If any return errors, those are bugs to fix.

**Stage 6: Use for Real Work**

Once fully bootstrapped and validated:
- Project teams can `nlspec_import` their own project specs
- Agents can slice, validate, patch, and decompose specs
- The system manages itself alongside project specs

---

## Pre-Loaded System Namespace

The `nlspec` namespace is reserved for the system's own specs. When the server starts, this namespace does not exist.

**Creating the nlspec namespace:**

The first `nlspec_import` call creates the namespace. After self-bootstrap:

```
nlspec_namespaces({})
→ Returns:
[
  {
    name: "nlspec",
    specs: ["l1-bootstrap", "l2-workflows", "l3-config", "l1-server", "l2-workflows", "l3-config", "l4-seed"],
    created_at: "2026-03-21T..."
  }
]
```

**Invariants for the nlspec namespace:**

- Reserved for nlspec's own specifications
- Can be queried like any other namespace
- Patches to nlspec specs follow the same workflow as any project spec
- Version numbers track spec evolution
- Changes are git-tracked (if git is enabled)

---

## Seed Data Initialization

### Initial State (Empty Server)

When the server first starts:

1. PostgreSQL (or SQLite in Phase 1) is initialized with empty schema
2. No namespaces exist
3. No specs are indexed
4. MCP/gRPC tools are available but return empty results

**Key point:** The system does NOT require pre-populated specs. It starts empty and is grown by import.

### Initialization Sequence

```
Server Start
  ↓
Config Loaded (from env vars / config.yaml)
  ↓
Database Connection Opened (PostgreSQL or SQLite)
  ↓
Database Schema Created (if not exist)
  ↓
MCP Server Registered (8 Phase 1 tools + 13 Phase 2 tools)
  ↓
gRPC Server Started (if configured, Phase 2+)
  ↓
File Watcher Started (if config.storage.watch=true)
  ↓
Ready: nlspec_import can be called
```

### First Spec Import

The first time `nlspec_import` is called:

```
Agent calls nlspec_import({namespace: "nlspec", spec_id: "l1-bootstrap", path: "specs/L1/L1-nlspec-server-spec.md"})
  ↓
MCP Tool Layer routes to nlspec_import
  ↓
Namespace Manager checks: does "nlspec" namespace exist?
  ↓
  No → creates it
  ↓
Spec Store reads markdown file from disk
  ↓
Spec Parser parses the file
  ↓
SQL INSERT: spec into `specs` table
SQL INSERT: elements into `elements` table
SQL INSERT: references into `references` table
SQL INSERT/UPDATE: FTS index
  ↓
Return: Spec object with element_count
```

After the first import:
- The `nlspec` namespace exists
- 1 spec is indexed
- Agent can call `nlspec_get`, `nlspec_search`, etc. and get results

### Demo Seed Data (Optional)

If a `.nlspec/demo.yaml` file is present at startup, the server can optionally load demo specs:

```yaml
# .nlspec/demo.yaml
demo:
  enabled: false  # set to true to auto-import demo specs
  specs:
    - namespace: demo
      spec_id: kv-store
      path: specs/examples/kv-store-spec.md
    - namespace: demo
      spec_id: auth
      path: specs/examples/auth-spec.md
```

If enabled, the server auto-imports these specs at startup. This is useful for:
- Tutorials (learn nlspec by exploring example specs)
- Testing (pre-load fixtures)
- Demos (show the system working without manual import)

**Default:** demo mode is disabled. Specs are imported explicitly via `nlspec_import`.

---

## Deployment Checklist

Before going live, ensure:

**Phase 1 Only:**
- [ ] Layer 1 is implemented and tested
- [ ] Layer 2 is implemented and tested
- [ ] Layer 3 config is finalized
- [ ] Database (SQLite) is set up
- [ ] Environment variables are configured
- [ ] Server can start without errors: `nlspec-server --specs-dir ./specs`
- [ ] First import succeeds: `nlspec_import({namespace: "nlspec", spec_id: "l1-bootstrap", ...})`
- [ ] Self-validation passes: `nlspec_validate({namespace: "nlspec", spec_id: "l1-bootstrap"})` → `is_valid=true`

**Phase 2 Full System:**
- [ ] Phase 1 is fully bootstrapped and validated
- [ ] Layer 1 extended is implemented and tested
- [ ] Layer 2 extended is implemented and tested
- [ ] Layer 3 extended config is finalized
- [ ] Database (PostgreSQL) is set up and accessible
- [ ] gRPC port (50060) is available and configured
- [ ] Docker image is built (if deploying to container)
- [ ] Docker Compose is configured and tested
- [ ] All environment variables are configured
- [ ] `.env.example` is present and documented
- [ ] All 4 merged layer specs are on disk in correct locations
- [ ] Server can start with gRPC: `nlspec-server --specs-dir ./specs --grpc-port 50060`
- [ ] All 4 merged layers can be imported successfully
- [ ] All 4 merged layers validate as `is_valid=true`
- [ ] MCP is configured in Claude Code, Cursor, or Claude Desktop
- [ ] gRPC is configured for internal tools
- [ ] Agent can call all 21 tools (8 Phase 1 + 13 Phase 2)
- [ ] Team can import project specs: `nlspec_import({namespace: "myproject", ...})`
- [ ] All advanced Phase 2 operations work (slice, validate, patch, split, drift, substrate)

---

## Staged Deployment Options

### Option A: Phase 1 Only (Minimal)

If resources are constrained:

```bash
# Implement only Phase 1 (parser, store, query, 8 tools)
# Skip Phase 2 for now

# Start server
nlspec-server --specs-dir ./specs

# Import Phase 1 specs
nlspec_import({namespace: "nlspec", spec_id: "l1-bootstrap", path: "specs/L1/L1-nlspec-server-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "l2-workflows", path: "specs/L2/L2-nlspec-workflows-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "l3-config", path: "specs/L3/L3-nlspec-config.md"})

# System is now functional:
# - Can parse, search, CRUD specs
# - Cannot slice, validate, patch (those are Phase 2+ features)
# - Can still be used for project specs
```

Later, when ready:
- Implement Phase 2 (add validators, patchers, slicers, etc.)
- Re-import with extended specs
- System gains new capabilities

### Option B: Phase 1 + 2 (Full)

```bash
# Implement both Phase 1 and Phase 2

# Start server with gRPC
nlspec-server --specs-dir ./specs --grpc-port 50060

# Import all 4 merged layers
nlspec_import({namespace: "nlspec", spec_id: "l1-server", path: "specs/mcp/L1/L1-nlspec-server-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "l2-workflows", path: "specs/mcp/L2/L2-nlspec-workflows-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "l3-config", path: "specs/mcp/L3/L3-nlspec-config.md"})
nlspec_import({namespace: "nlspec", spec_id: "l4-seed", path: "specs/mcp/L4/L4-nlspec-seed-spec.md"})

# System now has all 21 tools and can:
# - Manage multiple namespaces
# - Slice context, validate, patch, decompose
# - Detect drift, resolve substrates
# - Integrate via gRPC
```

### Option C: Containerized (Cloud-Ready)

```bash
# Build and run in Docker Compose
docker-compose up -d

# Server is available at:
# - MCP: stdio (via Claude Code configured with .mcp.json)
# - SSE: http://localhost:8080
# - gRPC: localhost:50060

# Import all specs (same as Option B)
```

---

## Rollback & Recovery

If a broken spec is imported (e.g., due to a bug in a patch):

```bash
# Option 1: Delete and re-import
nlspec_delete_namespace({namespace: "myproject"})
nlspec_import({namespace: "myproject", spec_id: "auth", path: "specs/auth-spec.md"})

# Option 2: Use git to revert, then re-import
git checkout specs/auth-spec.md
nlspec_import({namespace: "myproject", spec_id: "auth", path: "specs/auth-spec.md"})

# Option 3: Create a patch to fix the spec (recommended)
nlspec_patch_create({namespace: "myproject", spec_id: "auth", category: "A", ...})
# Fix and absorb the patch
nlspec_patch_absorb({patch_id: "PATCH-NNN"})
```

---

## Appendix A: Version Strategy

Layer specs follow semantic versioning:

- **Major version (1.x.x):** Layer contract changes (breaking)
- **Minor version (x.1.x):** New FUNCTION, RECORD, tool (backward compatible)
- **Patch version (x.x.1):** Bug fixes, clarifications, scenario updates

When a patch is absorbed:
- Spec version is incremented (typically patch: 0.1.0 → 0.1.1)
- Patch file is marked ABSORBED
- Change is git-committed (if git integration is enabled)

---

## Appendix B: Future Expansion (Phase 3+)

When nlspec gains new capabilities (hypothetical Phase 3):

1. Implement Phase 3 spec alongside Phases 1-2
2. Import Phase 3 when ready
3. No impact on existing specs or projects
4. Agents can opt-in to new features as needed

This layered design allows the system to grow without breaking existing systems.

---

## Appendix C: Glossary

| Term | Definition |
|------|-----------|
| **Bootstrap** | Self-hosting: the system manages its own specs |
| **Seed data** | Initial state provided at startup |
| **Layer stack** | The 4-layer spec organization (Specification, Realization, Configuration, Seed) |
| **System namespace** | The `nlspec` namespace reserved for nlspec's own specs |
| **Self-import** | Importing nlspec's own specs into itself after implementation |
| **Self-validate** | Running nlspec_validate on nlspec's own specs to check for bugs |
| **Phase 1** | Bootstrap: minimal system with 8 CRUD tools, single-project mode |
| **Phase 2** | Extended: adds 13 advanced tools, multi-project, namespaces, validation, slicing, patching, graphs, drift, decomposition |
| **Merged specs** | Single-file specs combining Phase 1 + Phase 2 definitions for each layer |

---

## Appendix D: Revision History

```
| Version | Date       | Author                  | Changes                        |
|---------|------------|-------------------------|--------------------------------|
| 1.0.0   | Mar 2026   | Divyendu Deepak Singh   | Layer 4: Seed Data             |
```
