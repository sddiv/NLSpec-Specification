# nlspec MCP Server Specification

This directory contains the complete specification for the nlspec MCP server, organized in the [4-layer composition model for software](https://sddiv.github.io/blog/2026/03/18/a-4-layer-composition-model-for-software/).

## Overview

nlspec is an MCP server that parses nlspec markdown files into structured, queryable elements and exposes CRUD + search operations as MCP tools. The system spans two phases of capabilities:

- **Phase 1-Bootstrap:** Core CRUD operations, full-text search, 8 MCP tools, single-project mode
- **Phase 2-Extended:** Multi-project namespaces, context slicing, validation, patching, graph operations, substrate detection, decomposition, drift detection, 13 additional tools (21 total)

The specification is merged into a single 4-layer set, removing phase distinctions while preserving all content.

## Layer Structure

All features from both phases are unified into 4 sequential layers:

### Layer 1: Specification (`L1/L1-nlspec-server-spec.md`)

**Type:** Contracts & Interfaces

Defines all data records, functions, tools, errors, and contracts for the complete system (Phase 1 + 2). This is the contract layer that implementation must satisfy.

**Includes:**
- All 30 RECORD types (Spec, SpecElement, Reference, ContextSlice, Patch, ValidationResult, Substrate chains, etc.)
- All 15 core FUNCTION definitions (parse_spec, store operations, query operations, context slicing, validation, patching, graph building, splitting, drift detection)
- All 21 MCP TOOL definitions (8 Phase 1 bootstrap + 13 Phase 2 advanced)
- All 15 error types
- Contracts and guarantees
- Component inventory

### Layer 2: Realization & Workflows (`L2/L2-nlspec-workflows-spec.md`)

**Type:** Data Flows & Scenarios

Describes how Layer 1 contracts are realized through data flows and concrete scenarios.

**Includes:**
- Build order: The sequence for implementing Phase 1 then Phase 2 features
- 10+ data flow diagrams showing how operations move through the system
- 13 scenarios covering all major workflows
- Scenario tiers (SMOKE, section coverage, full round-trip)

### Layer 3: Configuration (`L3/L3-nlspec-config.md`)

**Type:** Deployment & Runtime Configuration

Defines all environment variables, configuration options, file structure, dependencies, and deployment artifacts.

**Includes:**
- CONFIG entries for all subsystems
- Deployment strategies (Phase 1 local-only, Phase 2 cloud-ready)
- Complete file structure with all modules
- Build prerequisites and installation steps
- Runtime dependencies
- Environment variable reference

### Layer 4: Seed Data (`L4/L4-nlspec-seed-spec.md`)

**Type:** Self-Bootstrap & Initialization

Describes how the nlspec system bootstraps itself and manages its own specifications.

**Includes:**
- Build and deploy order (6 stages from implementation to production)
- System namespace (nlspec reserved for system specs)
- Initialization sequence
- Deployment checklist
- Staged deployment options

## Layer Derivation

Layers form a linear stack, each deriving from the previous:

```
L1 (Specification)
   ↓ defines contracts for
L2 (Realization)
   ↓ whose deployment is configured in
L3 (Configuration)
   ↓ which is self-bootstrapped by
L4 (Seed Data)
```

## Reading Guide

### For Architects
Start with **L1** to understand the complete contract. Then read **L2 Build Order** to understand dependencies.

### For Implementers
1. Start with **L1** for the contract
2. Read **L2 Data Flows** to understand operation flow
3. Use **L3** to set up your environment
4. Follow **L2 Build Order** for implementation sequence
5. Use **L4** for deployment

### For Operators
1. Read **L3** for configuration and environment setup
2. Read **L4** for deployment checklist and startup
3. Reference **L3** environment variables when configuring

### For QA/Testing
1. Read **L2 Scenarios** for complete test scenarios (13 total)
2. Use **L3** for test environment setup
3. Run smoke tests for quick validation (Scenarios 1-3, 8-9)

## Unified vs. Phased

These 4 merged files represent a unified specification. The original files were organized as Phase 1 (3 layers) and Phase 2 (4 layers). This merged version combines them:

- L1-nlspec-server-spec.md (Phase 1 L1 + Phase 2 L1)
- L2-nlspec-workflows-spec.md (Phase 1 L2 + Phase 2 L2 + Build Order)
- L3-nlspec-config.md (Phase 1 L3 + Phase 2 L3)
- L4-nlspec-seed-spec.md (Phase 2 L4 only)

### Why Merge?

- **Single source of truth:** All features in one file per layer
- **Clear narrative:** Follow from specification → realization → configuration → seed
- **No phase language:** Specs are unified, not "Phase 1" or "Phase 2"
- **Build order clarity:** L2 explicitly documents implementation sequence
- **Easier navigation:** One file instead of cross-referencing multiple files

## Self-Reference

The nlspec system manages its own specifications. After implementation:

1. Start the server (empty index)
2. Import all 4 layer specs into the `nlspec` namespace
3. Validate: `nlspec_validate({namespace: "nlspec", spec_id: "l1-server"})` etc.
4. If all return `is_valid=true`, implementation is correct

This is **self-hosting**: the system describes itself.

## Questions & Answers

**Q: Do I have to implement Phase 2?**
A: No. Phase 1 is complete and useful on its own. Implement Phase 1 first, then Phase 2 when ready.

**Q: Can I use Phase 1 SQLite?**
A: Yes. Phase 1 defaults to SQLite for local development. Phase 2 recommends PostgreSQL for scale.

**Q: How do I test my implementation?**
A: Use **L2 Scenarios**. Scenarios 1-7 are Phase 1. Scenarios 8-13 are Phase 2.

**Q: What if I find a bug in the spec?**
A: Create a patch: `nlspec_patch_create({namespace: "nlspec", spec_id: "l1-server", ...})`. Fix and absorb.

## Version History

All 4 merged specifications are version 1.0.0, dated March 2026.

## License

CC-BY-4.0 (Creative Commons Attribution 4.0 International)
