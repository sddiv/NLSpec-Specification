# {PROJECT_NAME} — Natural Language Specification

> **Version:** 0.1.0
> **Author:** {AUTHOR}
> **Date:** {DATE}
> **Language Target:** {Rust | Go | TypeScript | Python | Language-Agnostic}
> **Status:** {Draft | Review | Approved | Implementing | Complete}

---

## How to Use This Spec

This document is a **complete, implementable specification**. Hand it to a coding agent
(Claude Code, Cursor, Codex, etc.) and it should produce a working system with no
additional instructions. Every RECORD, FUNCTION, and SCENARIO is precise enough to
code from. If you find ambiguity, the spec is incomplete — fix the spec, not the code.

**For the coding agent:** Implement every RECORD as a struct/class, every FUNCTION as
a method/function, and validate against every SCENARIO. If a SCENARIO fails, the
implementation is wrong. Do not modify SCENARIOs to match your implementation.

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Problem Statement](#2-problem-statement)
3. [Architecture Overview](#3-architecture-overview)
4. [Data Model](#4-data-model)
5. [Core Functions](#5-core-functions)
6. [API Surface](#6-api-surface)
7. [Error Model](#7-error-model)
8. [Configuration](#8-configuration)
9. [Deployment Artifacts](#9-deployment-artifacts)
10. [Scenarios](#10-scenarios)
11. [Dependencies](#11-dependencies)
12. [File Structure](#12-file-structure)
13. [Maintenance Workflow](#13-maintenance-workflow)
14. [Build and Run](#14-build-and-run)
15. [Boundaries](#15-boundaries)
16. [Dependency Contracts](#16-dependency-contracts)

---

## 1. Abstract

**One paragraph.** What this system does, in plain English. A developer reading only this
paragraph should understand the purpose, scope, and primary value proposition.

```
{PROJECT_NAME} is a {one-line description}.

It does {primary capability}. It does NOT do {explicit non-goals}.

The primary consumer of this system is {who calls it / who uses it}.

The implementation target is {language/platform}. The expected binary/artifact is {what}.
```

**If this spec covers multiple components** (e.g., a system with distinct subsystems
that share a codebase), list them here:

```
COMPONENTS:
  {Component A}: {one-line purpose}
  {Component B}: {one-line purpose}
  {Component C}: {one-line purpose}

Each component gets its own subsections in Sections 4, 5, 6, 7.
For example: Section 4.1 covers Component A's RECORDs, Section 4.2 covers
Component B's RECORDs. This keeps the spec navigable — the agent can read
only the component it's working on.
```

---

## 2. Problem Statement

**What exists today and why it's insufficient.** Be specific about the pain point.
Include measurable deficiencies (latency, cost, complexity, missing capability).

```
### Current State
{What exists today. Be specific.}

### Deficiency
{What's wrong with it. Include numbers where possible.}

### Target State
{What this system achieves. Include target metrics.}

### Key Insight
{The one architectural or conceptual insight that makes this system possible/better.}
```

---

## 3. Architecture Overview

**ASCII diagram of the system.** Show every major component, how they connect, and
what crosses system boundaries. This diagram is the reference for all subsequent sections.

```
+--------------------------------------------------+
|                  {SYSTEM_NAME}                    |
|                                                    |
|  +------------------+  +---------------------+   |
|  |  Component A     |  |  Component B        |   |
|  |  - responsibility|  |  - responsibility   |   |
|  |  - responsibility|  |  - responsibility   |   |
|  +--------+---------+  +---------+-----------+   |
|           |                      |                |
|           v                      v                |
|  +------------------+  +---------------------+   |
|  |  Component C     |  |  External System    |   |
|  +------------------+  +---------------------+   |
+--------------------------------------------------+
```

### 3.1 Component Inventory

For each box in the diagram:

```
### Component: {NAME}
- **Responsibility:** {What it does, one sentence}
- **Owns:** {What state/data it manages}
- **Calls:** {What other components it invokes}
- **Called by:** {What invokes it}
- **Concurrency:** {Single-threaded | Multi-threaded | Async | Actor}
- **Lifecycle:** {Created when, destroyed when}
```

### 3.2 Data Flow

Describe the primary data flows through the system. Use numbered steps.

```
### Flow: {NAME} (e.g., "Query Execution")
1. {Caller} sends {what} to {Component A}
2. {Component A} does {what} and calls {Component B}
3. {Component B} returns {what}
4. {Component A} returns {what} to {Caller}

Latency budget: {total}ms
- Step 1: {x}ms
- Step 2: {y}ms
- Step 3: {z}ms
```

---

## 4. Data Model

**Every data structure in the system.** Use RECORD for structs, ENUM for enumerations,
ALIAS for type aliases. These are language-agnostic — implement as structs, classes,
or whatever your language provides.

### Conventions

```
RECORD Name:
  field_name : Type            -- description
  field_name : Type | None     -- optional field
  field_name : List<Type>      -- list/array/vec
  field_name : Map<Key, Value> -- hashmap/dictionary
  field_name : Type = default  -- field with default value

ENUM Name:
  VARIANT_A   -- description
  VARIANT_B   -- description

ALIAS Name = UnderlyingType    -- type alias
```

### 4.1 Core Records

```
RECORD {EntityName}:
  id          : String              -- unique identifier, UUID v4
  created_at  : Timestamp           -- creation time, UTC
  {field}     : {Type}              -- {description}
  {field}     : {Type} | None       -- {description}, optional
  {field}     : List<{Type}>        -- {description}

  USED BY: function_a, function_b   -- functions that read/write this record (for context slicing)
```

**Invariants:**
- {field} must always satisfy {condition}
- {field} is immutable after creation
- If {field_a} is set, {field_b} must also be set

### 4.2 Enumerations

```
ENUM {EnumName}:
  VALUE_A    -- {when this value is used}
  VALUE_B    -- {when this value is used}
  VALUE_C    -- {when this value is used}
```

### 4.3 Type Aliases

```
ALIAS EntityId = String         -- UUID v4 format
ALIAS Timestamp = i64           -- Unix milliseconds, UTC
ALIAS Duration = i64            -- Milliseconds
```

---

## 5. Core Functions

**Every function the system exposes or uses internally.** Each function is specified with:
inputs, outputs, preconditions, postconditions, error conditions, and behavioral notes.

### Conventions

```
FUNCTION function_name(param: Type, param: Type) -> ReturnType
  USES: RecordA, RecordB       -- RECORDs this function reads/writes (for context slicing)
  THROWS: ErrorA, ErrorB       -- errors this function can produce

  PRECONDITIONS:
  - {condition that must be true before calling}

  POSTCONDITIONS:
  - {condition that is guaranteed true after successful return}

  BEHAVIOR:
  1. {Step 1 — what happens}
  2. {Step 2 — what happens}
  3. {Step 3 — what happens}

  ERRORS:
  - ErrorA: when {condition}
  - ErrorB: when {condition}

  PERFORMANCE:
  - Target latency: {x}ms (p50), {y}ms (p99)
  - Memory allocation: {description}

  NOTES:
  - {Any additional behavioral notes}
  - {Thread safety guarantees}
  - {Idempotency guarantees}
```

### 5.1 {Function Group Name}

```
FUNCTION {name}({params}) -> {return}
  ...
```

---

## 6. API Surface

**The external interface.** HTTP endpoints, gRPC services, CLI commands, or library API —
whatever the caller sees. Each endpoint is fully specified.

### Conventions

```
ENDPOINT {METHOD} {path}
  DESCRIPTION: {what it does}

  REQUEST:
    Headers:
      Authorization: Bearer {token}
      Content-Type: application/json
    Body:
      {RECORD reference or inline schema}

  RESPONSE 200:
    Body:
      {RECORD reference or inline schema}

  RESPONSE 400:
    Body:
      error : String  -- human-readable error message
      code  : String  -- machine-readable error code

  RESPONSE 500:
    Body:
      error : String
      code  : "INTERNAL_ERROR"

  EXAMPLE:
    Request:
      POST /api/v1/search
      {"query": "customer churn", "limit": 10}
    Response:
      {"results": [...], "total": 42, "latency_ms": 3}
```

### 6.1 {API Group}

```
ENDPOINT ...
```

---

## 7. Error Model

**Complete error taxonomy.** Every error the system can produce, organized hierarchically.
The coding agent must implement all of these.

```
{ProjectName}Error
  +-- ConfigError              -- system misconfiguration
  |     +-- MissingConfig      -- required config key absent
  |     +-- InvalidConfig      -- config value out of range or wrong type
  +-- {DomainError}            -- errors from core domain logic
  |     +-- {SpecificError}    -- {when this happens}
  |     +-- {SpecificError}    -- {when this happens}
  +-- IOError                  -- filesystem, network, hardware
  |     +-- NetworkError       -- connection failed, timeout
  |     +-- StorageError       -- disk full, permission denied
  +-- InternalError            -- bugs, invariant violations (should never happen)
```

**Error handling rules:**
- All errors must include a human-readable message and a machine-readable code
- Errors must propagate with full context (no swallowing errors silently)
- {DomainError} variants must be recoverable — the caller can retry or take action
- InternalError means a bug — log, alert, and fail loudly
- Never panic/crash on {DomainError} — only on InternalError

---

## 8. Configuration

**Every knob the system exposes.** Environment variables, config files, CLI flags.
Each config value has a type, default, and description.

```
CONFIG {config_group}:
  {key}:
    type: {Type}
    default: {value}
    env: {ENVIRONMENT_VARIABLE_NAME}
    description: {what it controls}
    validation: {constraints, e.g., "must be > 0", "must be valid URL"}

  {key}:
    type: {Type}
    required: true
    env: {ENVIRONMENT_VARIABLE_NAME}
    description: {what it controls}
```

### 8.1 {Config Group}

```
CONFIG server:
  port:
    type: u16
    default: 8080
    env: MYSERVICE_PORT
    description: HTTP server listen port
    validation: 1024-65535

  gpu_memory_limit_mb:
    type: u64
    default: 10240
    env: MYSERVICE_GPU_MEMORY_MB
    description: Maximum GPU memory for catalog index
    validation: must be > 512
```

---

## 9. Deployment Artifacts

**Infrastructure-as-code, manifests, and deployment definitions.** Not every spec needs
this section — skip it for pure libraries. Include it for anything that gets deployed
(services, infrastructure, controllers).

### 9.1 Container / Image

```
IMAGE {name}:
  base: {base image, e.g., "rust:1.76-slim", "ubuntu:24.04"}
  ports: [{port}:{protocol}]
  volumes: [{host_path}:{container_path}:{mode}]
  env: [{VARIABLE}: {description}]
  healthcheck: {command or endpoint}
  resource_limits:
    cpu: {cores}
    memory: {MB}
    gpu: {type and count, if applicable}
```

### 9.2 Kubernetes Manifests

```
MANIFEST {name}:
  kind: {Deployment | StatefulSet | DaemonSet | Service | ConfigMap | etc.}
  namespace: {namespace}
  purpose: {what this manifest creates and why}
  spec_highlights:
    - {key configuration choices and why}
    - {resource limits}
    - {affinity/anti-affinity rules}
    - {volume mounts}

  Full YAML: (inline or reference to file in deploy/ directory)
```

### 9.3 Infrastructure Dependencies

```
INFRA {name}:
  type: {GPU | storage | network | certificate | DNS | etc.}
  required: {true | false}
  specification: {what exactly is needed}
  provisioning: {how it gets created — manual, Terraform, Helm, etc.}
  verification: {how to confirm it's ready}
```

### 9.4 Deployment Order

```
DEPLOY ORDER:
  1. {what gets deployed first and why}
  2. {what depends on step 1}
  3. {what depends on step 2}

  Verification between steps:
  - After step 1: {check command and expected output}
  - After step 2: {check command and expected output}
```

### 9.5 CI/CD Pipelines

**Define when and how scenarios run automatically.** The Quality Spec (or Scenarios
section) defines WHAT to test. This section defines WHEN those tests run and what
happens on failure. The agent generates the actual pipeline files (GitHub Actions,
GitLab CI, etc.) from these definitions.

```
PIPELINE ci:
  trigger: {push to any branch | pull request | merge to main}
  steps:
    1. {Build step}
    2. {Run SMOKE scenarios — must pass, < 30s}
    3. {Run AFFECTED scenarios — determined by FILE_TO_SECTION_MAP}
  failure: {block merge | alert | etc.}

PIPELINE nightly:
  trigger: {cron schedule, e.g., "02:00 UTC daily"}
  steps:
    1. {Build step}
    2. {Run FULL scenario suite — all tiers}
    3. {Run performance benchmarks}
    4. {Generate coverage report}
  failure: {alert team, do not block}

PIPELINE release:
  trigger: {tag matching v* | manual}
  steps:
    1. {Build step}
    2. {Run FULL scenario suite}
    3. {Run security scan}
    4. {Build container image}
    5. {Push to registry}
    6. {Deploy to staging}
    7. {Run SMOKE against staging}
  failure: {block release}
```

### 9.6 File-to-Section Mapping

**Maps source files to spec sections for automatic AFFECTED scenario determination.**
When a file changes, the CI pipeline uses this map to identify which scenarios to run.

```
FILE_TO_SECTION_MAP:
  {src/path/to/module/*}   → [{SEC:x.x}]
  {src/path/to/api/*}      → [{SEC:x.x}]
  {src/path/to/errors.*}   → [{SEC:x}]
  {src/path/to/config.*}   → [{SEC:x}]
  {deploy/*}               → [{SEC:9}]
```

---

## 10. Scenarios

**End-to-end validation scenarios.** These are NOT unit tests — they are behavioral
specifications. Each scenario describes a complete user journey with expected outcomes.
The coding agent must make ALL scenarios pass. Scenarios are the "holdout set" —
they validate the system works correctly end-to-end.

### Conventions

```
SCENARIO {number}: {name}
  GIVEN:
  - {precondition — system state before the scenario}
  - {precondition}

  WHEN:
  - {action — what the user/caller does}
  - {action}

  THEN:
  - {expected outcome — what must be true after}
  - {expected outcome}
  - {performance constraint — must complete within X ms}

  NOTES:
  - {Edge cases to watch for}
  - {Why this scenario matters}
```

### 9.1 Happy Path Scenarios

```
SCENARIO 1: {Basic successful operation}
  GIVEN:
  - ...
  WHEN:
  - ...
  THEN:
  - ...
```

### 9.2 Error Scenarios

```
SCENARIO 10: {What happens when X fails}
  GIVEN:
  - ...
  WHEN:
  - ...
  THEN:
  - Error {ErrorType} is returned
  - System state is unchanged (rollback)
  - ...
```

### 9.3 Performance Scenarios

```
SCENARIO 20: {Latency under load}
  GIVEN:
  - {system loaded with N records}
  WHEN:
  - {M concurrent requests of type X}
  THEN:
  - p50 latency < {x}ms
  - p99 latency < {y}ms
  - No errors
```

### 9.4 Edge Case Scenarios

```
SCENARIO 30: {Boundary condition}
  GIVEN:
  - {unusual or extreme state}
  WHEN:
  - ...
  THEN:
  - {system handles gracefully}
```

---

## 11. Dependencies

**External systems and libraries this project requires.**

```
DEPENDENCY {name}:
  purpose: {why we need it}
  version: {minimum version}
  required: {true | false — can the system function without it?}
  interface: {how we interact with it — API, library, CLI, socket}
  fallback: {what happens if it's unavailable}
```

### 10.1 Runtime Dependencies

```
DEPENDENCY {name}:
  ...
```

### 10.2 Build Dependencies

```
DEPENDENCY {name}:
  ...
```

---

## 12. File Structure

**The expected directory layout.** The coding agent must produce this structure.

```
{project_name}/
├── Cargo.toml (or package.json, go.mod, etc.)
├── README.md
├── SPEC.md                    -- this file
├── src/
│   ├── main.rs               -- entry point
│   ├── lib.rs                 -- library root
│   ├── config/
│   │   └── mod.rs             -- configuration loading
│   ├── models/
│   │   ├── mod.rs             -- re-exports
│   │   ├── {entity}.rs        -- RECORD definitions
│   │   └── errors.rs          -- error types
│   ├── {component_a}/
│   │   ├── mod.rs
│   │   └── ...
│   ├── {component_b}/
│   │   ├── mod.rs
│   │   └── ...
│   └── api/
│       ├── mod.rs
│       ├── routes.rs          -- endpoint definitions
│       └── handlers.rs        -- request handlers
├── tests/
│   ├── scenarios/
│   │   ├── scenario_01.rs     -- SCENARIO 1
│   │   ├── scenario_02.rs     -- SCENARIO 2
│   │   └── ...
│   └── integration/
│       └── ...
└── config/
    └── default.toml           -- default configuration
```

---

## 13. Maintenance Workflow

**How to handle bugs, changes, and iteration after initial implementation.**

The spec is the source of truth, not the code. Code is a derived artifact of the spec.
When something is wrong, determine which category the issue falls into and follow
the corresponding workflow.

### 13.1 Bug Categories

```
CATEGORY A — Spec Deficiency:
  The spec was ambiguous, incomplete, or wrong.
  The agent implemented exactly what the spec said, but that was incorrect.

  Workflow:
  1. Fix the spec (update the relevant RECORD, FUNCTION, or BEHAVIOR)
  2. Add a SCENARIO that would have caught this bug
  3. Bump spec version (e.g., 0.1.0 → 0.1.1)
  4. Feed the FULL UPDATED SPEC to the coding agent
  5. Agent re-implements against the corrected spec
  6. New SCENARIO must pass

CATEGORY B — Implementation Error:
  The spec was clear and correct. The agent made a mistake.
  
  Workflow:
  1. Do NOT modify the original spec
  2. Write a PATCH SPEC (see format below) referencing the section + scenario that fails
  3. Feed the agent: ORIGINAL SPEC + PATCH SPEC
  4. Agent fixes the implementation
  5. Existing SCENARIO that was failing now passes
  6. Fold the patch back into the main spec at next version bump (if it added clarity)

CATEGORY C — Missing Capability:
  The spec didn't cover this case at all. New feature or edge case discovered.

  Workflow:
  1. Add new RECORDs, FUNCTIONs, and/or SCENARIOs to the spec
  2. Mark new additions with <!-- ADDED v0.1.1 --> for traceability
  3. Bump spec version
  4. Feed the FULL UPDATED SPEC to the coding agent
  5. New SCENARIOs must pass, all existing SCENARIOs must still pass
```

### 13.2 Patch Spec Format

A patch spec is a small, focused document used for Category B fixes. It is temporary
and gets absorbed into the main spec at the next version bump.

```markdown
# PATCH: {short description}

> **Patches:** {SPEC_NAME} v{version}
> **Section:** {section number and name}
> **Failing Scenario:** {scenario number}
> **Date:** {date}

## Problem
{What is going wrong. Be specific. Include the error or incorrect behavior.}

## Root Cause
{Why the agent's implementation is wrong. Reference the spec section that is clear
about the correct behavior.}

## Fix
{Precise description of what the agent must change. Reference specific FUNCTIONs
or RECORDs.}

## Verification
{The existing SCENARIO number that must now pass, and/or a new SCENARIO to add.}

SCENARIO P{number}: {patch scenario name}
  GIVEN:
  - {state}
  WHEN:
  - {action that triggers the bug}
  THEN:
  - {correct behavior after fix}
```

### 13.3 What the Agent Receives — Modular Context, Not Full Spec

**Never feed the full spec for a bug fix.** The full spec is for initial implementation
and major version bumps ONLY. For patches and fixes, feed the minimum context needed.

The spec is designed to be sliceable by section number. Each FUNCTION references which
RECORDs it uses. Each SCENARIO references which FUNCTIONs it tests. This dependency
chain determines the context slice.

```
CONTEXT SLICE for a bug fix:
  1. The PATCH SPEC or updated SECTION (the change itself)
  2. The failing SCENARIO(s) (what must now pass)
  3. The RECORDs referenced by the affected FUNCTION (data structures it touches)
  4. The ERROR MODEL entries relevant to the function
  5. NOTHING ELSE

Typical slice size: 200-500 lines (not 3,000-7,000)
```

```
STAGE: Initial Implementation
  Agent receives: FULL SPEC.md
  Agent produces: complete codebase
  Validation: ALL SCENARIOs pass
  When: once, at project start

STAGE: Bug Fix (any category)
  Agent receives: CONTEXT SLICE (affected sections + failing scenarios + referenced RECORDs)
  Agent produces: targeted fix (specific files, not full codebase rewrite)
  Validation: AFFECTED SCENARIOs pass + SMOKE SCENARIOs pass
  When: each bug fix, potentially many per day

STAGE: Nightly / Version Bump Regression
  Agent receives: FULL SPEC.md (or nothing — just runs the full test suite)
  Agent produces: nothing new (just validation) OR clean-up refactors
  Validation: ALL SCENARIOs pass
  When: nightly batch job, or at each version bump

STAGE: Consolidation (absorb patches)
  Agent receives: FULL SPEC.md (new version with patches absorbed)
  Agent produces: verified codebase (may refactor to align with consolidated spec)
  Validation: ALL SCENARIOs pass
  When: periodically, e.g., weekly or at minor version bumps
```

### 13.4 Scenario Tiers — Not All Scenarios Run Every Time

SCENARIOs are tagged with tiers that determine when they run:

```
SCENARIO TIERS:

  SMOKE (runs on every change):
    - 3-5 scenarios that exercise the critical path end-to-end
    - If any SMOKE scenario fails, something fundamental is broken
    - Must complete in < 30 seconds total
    - Tagged: [SMOKE]

  AFFECTED (runs on relevant changes):
    - Scenarios that test the specific FUNCTION or RECORD being changed
    - Determined by dependency tracing from the changed section
    - Tagged with the section they validate: [SEC:5.3] [SEC:4.1]

  FULL (runs nightly / at version bumps):
    - ALL scenarios, including performance and edge cases
    - The complete regression suite
    - May take minutes to hours
    - Tagged: [FULL] (or untagged — all scenarios run at FULL tier)

Execution rule:
  On each patch:  run SMOKE + AFFECTED scenarios
  Nightly:        run FULL scenario suite
  Version bump:   run FULL scenario suite, then consolidate

Cost model:
  Each patch:     ~200-500 input tokens (context slice) + ~30s validation
  Nightly:        ~3,000-7,000 input tokens (full spec) + full validation
  This is how 3 engineers produce hundreds of patches per day without cost overruns.
```

### 13.5 Scenario Dependency Tags

Each SCENARIO declares which spec sections it validates. This enables automatic
determination of which scenarios to run when a section changes.

```
SCENARIO 7: Query returns results within latency budget  [SEC:5.3] [SEC:4.1] [SMOKE]
  GIVEN:
  - Catalog loaded with 1000 tables
  WHEN:
  - POST /search {"query": "customer churn"}
  THEN:
  - Response contains ranked results
  - Latency < 10ms
```

When Section 5.3 is patched, the agent runs:
- All [SMOKE] scenarios (always)
- All [SEC:5.3] scenarios (affected by the change)
- NOT scenarios tagged [SEC:6.2] or [SEC:8.1] (unrelated)

### 13.6 Spec Versioning Rules

```
Version format: MAJOR.MINOR.PATCH

  PATCH bump (0.1.0 → 0.1.1):
    - Bug fixes (Category A or C with small additions)
    - Patch absorption
    - SCENARIO additions
    - No breaking changes to RECORDs or API

  MINOR bump (0.1.0 → 0.2.0):
    - New FUNCTIONs or ENDPOINTS
    - New RECORDs (additive)
    - New configuration options
    - Backward-compatible changes

  MAJOR bump (0.1.0 → 1.0.0):
    - Breaking changes to RECORDs or API
    - Architectural changes
    - Removal of FUNCTIONs or ENDPOINTS
    - The agent should re-implement from scratch
```

### 13.7 Scenario as Regression Guard

Every bug fix (any category) MUST add or identify a SCENARIO that prevents regression.
SCENARIOs are append-only — you never remove a passing SCENARIO. The SCENARIO count
monotonically increases. This is the equivalent of StrongDM's "holdout set" — the
scenarios exist outside the agent's ability to modify them.

```
Rule: If a bug was found and fixed, there MUST be a SCENARIO that would catch
it if it reappeared. If no such SCENARIO exists, the fix is incomplete.
```

---

## 14. Build and Run

**How to build, test, and run the system.** Exact commands.

```
### Prerequisites
- {language} {version}
- {tool} {version}
- {system dependency}

### Build
$ {build command}

### Test
$ {test command}             # unit tests
$ {test command}             # scenario tests
$ {test command}             # all tests

### Run
$ {run command}

### Verify
$ {curl or CLI command to verify the system is running}
Expected output: {what you should see}
```

---

## 15. Boundaries

**What this system explicitly does NOT do.** Prevents scope creep during implementation.

```
### This System Does:
- {capability}
- {capability}

### This System Does NOT:
- {non-goal} — {why not, and what system handles this instead}
- {non-goal} — {why not}

### Future Extensions (not in this spec):
- {feature that might be added later but is explicitly out of scope now}
```

---

## 16. Dependency Contracts

**What this spec exports to consumers and expects from dependencies.** When spec A
declares `IMPORT spec B`, B's exports are automatically seeded as rules in A's runtime.
This section is machine-readable — the seed resolver uses it to produce deterministic
initialization state.

If this spec has no imports and is not imported by others, state:
`EXPORTS: none — this spec is standalone.`

### 16.1 Exports

Exports are constraints, rules, or data that this spec provides to any spec that
imports it. They are instantiated automatically — the consuming spec does not need
to reference them explicitly.

```
EXPORT {ExportName}:
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  target      : {which component/layer in the consumer receives this}
  condition   : {when this export applies — "ALWAYS" if unconditional}
  value       : {the constraint, rule, or data being exported}
  override    : NEVER | WITH_JUSTIFICATION | UNRESTRICTED
  source_ref  : [SEC:x.x]  -- which section of THIS spec defines the basis
```

**Export types:**
- `CONSTRAINT` — a limit that restricts behavior (e.g., max memory, timeout ceiling)
- `INVARIANT` — a property that must always hold at runtime, verified continuously
- `SEED_DATA` — initial state to populate at deploy time (e.g., default config values)
- `POLICY` — a behavioral default (e.g., retry strategy, logging level)

**Override levels:**
- `NEVER` — cannot be overridden by any consumer. Use for safety invariants, physical
  limits, and security boundaries. Two conflicting NEVER exports cause a build failure.
- `WITH_JUSTIFICATION` — can be overridden if the consumer documents why.
- `UNRESTRICTED` — consumer can freely override.

### 16.2 Expects

Expects are contracts this spec requires from its dependencies. If a dependency
does not export a matching contract, the build fails with a clear error.

```
EXPECTS {ExpectName}:
  from        : {spec_id or "ANY_DEPENDENCY"}
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  description : {what this spec needs and why}
  required    : true | false
  fallback    : {default value if required=false and no dependency provides it}
```

### 16.3 Conflict Resolution

When multiple dependencies export constraints that affect the same target:

1. `override=NEVER` exports always win. If two NEVER exports conflict, the build fails.
2. More restrictive constraint wins (lower limit, stricter policy).
3. Spec closer to the dependency root (fewer hops) wins.
4. If still ambiguous, the build fails with a clear error.

### 16.4 Notes for Spec Authors

- Every spec that other specs depend on SHOULD define its EXPORTS explicitly.
- If your spec has no exports, state: `EXPORTS: none — this spec is a leaf.`
- The seed resolver walks the full dependency graph and collects all exports before
  any code is generated. Conflicts are caught at build time, not runtime.
- For details on the resolution algorithm, see `specs/seed-resolver-spec.md`.

---

## Appendix A: Glossary

```
{Term}: {Definition as used in this spec}
{Term}: {Definition}
```

## Appendix B: Related Specs and Cross-Spec References

When this spec references RECORDs, FUNCTIONs, or ENUMs defined in another spec,
use the IMPORT declaration. This tells the coding agent where to find the definition
without duplicating it.

```
IMPORT {RecordName} FROM {spec-name}-spec.md Section {x.x}
IMPORT {FunctionName} FROM {spec-name}-spec.md Section {x.x}
IMPORT {EnumName} FROM {spec-name}-spec.md Section {x.x}
```

Example:
```
IMPORT ConstraintMask FROM cra-spec.md Section 4.3
IMPORT KernelDefinition FROM cra-spec.md Section 4.5
IMPORT QueryResponse FROM search-service-spec.md Section 4.2
```

The coding agent must read the imported definition from the referenced spec file.
If the referenced spec is not on disk, ask the human for it.

**Related specs for this project:**
```
{Spec Name} ({filename}): {relationship — calls, is called by, shares RECORDs, etc.}
{Spec Name} ({filename}): {relationship}
```

## Appendix C: Revision History

```
| Version | Date       | Author | Changes                    |
|---------|------------|--------|----------------------------|
| 0.1.0   | {date}     | {name} | Initial draft              |
```
