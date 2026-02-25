# {PROJECT_NAME} — Natural Language Specification

> **Version:** 0.1.0
> **Author:** {AUTHOR}
> **Date:** {DATE}
> **Type:** {SYSTEM | PATTERN | ASSET}
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

### Spec Types

**SYSTEM** (default) — Produces a running system. All recommended sections apply. The agent
implements this as a deployable service, library, or application.

**PATTERN** — Defines a reusable architectural blueprint. The agent does not deploy
this standalone — other SYSTEM specs reference it via `USES PATTERN:`. Most sections
apply except Deployment (typically N/A unless the pattern includes infrastructure
templates). The Scenarios section validates the pattern's constraints and interface
contracts, not a running system.

**ASSET** — Catalogs static resources and defines the style guide for how agents must
use them. Assets can come from two sources:

- **Workspace assets** (images, stylesheets, fonts, config templates, certificates)
  that already exist in the workspace file structure. The agent does NOT generate
  these — the ASSET spec tells the agent where they are and how to use them.

- **External assets** (icon libraries, font packages, design token packs, UI component
  libraries) that must be installed before use. The ASSET spec declares a DEPENDENCY
  for installation, then catalogs the installed assets the same way as workspace assets.
  After install, the agent reads the actual installed contents to build the catalog.

For simple asset catalogs, only Abstract, Architecture, FileStructure, Boundaries,
and Contracts sections are needed. For complex design systems with style guides, use
all sections that apply. The Functions section becomes the style guide — the rules
and tokens the agent follows when writing code that references these assets.

**When to use an ASSET spec vs inline DEPENDENCY USAGE RULES:**
- Simple case (one icon library, 3-5 rules) → DEPENDENCY with `interface: asset-pack`
  and inline USAGE RULES. No separate ASSET spec needed.
- Complex case (design system, brand guidelines, many rules/tokens, multiple asset
  packs) → DEPENDENCY for install + separate ASSET spec for catalog and style guide.
  The ASSET spec references the DEPENDENCY via `REQUIRES DEPENDENCY:`.

---

## Section Declaration

**Specs may declare their sections at the top.** When present, the SECTIONS block
is the contract — it tells the agent what sections exist, what they're called, and
what each contains. This enables strict mode: the parser validates completeness
and the agent can check for structural gaps.

**When no SECTIONS block is present**, the parser operates in loose mode: it discovers
sections from `## ` headers in document order. The spec still works — all parsing,
slicing, and implementation proceed normally. The agent just can't validate
completeness against a declaration, and gap detection is less precise.

**Either way, the parser never rejects a spec.** Missing sections are reported as
gaps, not errors. A spec with just an Abstract and some FUNCTIONs is parseable and
implementable — the agent will flag what's missing and proceed with what it has.

**Recommended sections for TYPE: SYSTEM:**

```
SECTIONS:
  Abstract        — what this spec defines
  Problem         — current state, deficiency, target state, key insight
  Architecture    — components, data flows, patterns, assets
  DataModel       — RECORDs, ENUMs, ALIASes with invariants
  Functions       — every function with behavior, pre/post conditions, errors
  API             — external interface (endpoints, CLI, MCP tools)
  Errors          — error taxonomy with codes, severity, handling
  Config          — configuration knobs with types, defaults, validation
  Deployment      — containers, manifests, pipelines, CI/CD
  Scenarios       — behavioral validation (the holdout set)
  Dependencies    — external systems, libraries, and asset packs with pinned versions
  FileStructure   — expected directory layout
  Maintenance     — bug workflow, patches, scenario tiers, versioning
  BuildAndRun     — exact commands to build, test, run, verify, and artifact manifest
  Boundaries      — explicit non-goals
  Contracts       — exports, expects, conflict resolution
```

**Recommended sections for TYPE: PATTERN:**

```
SECTIONS:
  Abstract        — what problem this pattern solves, when to use it
  Problem         — the architectural problem and why existing approaches fail
  Architecture    — the pattern's structure and components
  Interface       — what the consuming spec interacts with
  Constraints     — rules and invariants the pattern enforces
  Scenarios       — constraint validation (not end-to-end system tests)
  Boundaries      — what this pattern does not cover
  Contracts       — architectural constraints exported to consumers
```

**Recommended sections for TYPE: ASSET:**

```
SECTIONS:
  Abstract        — what assets this catalogs (workspace and/or external) and how agents use them
  AssetCatalog    — file paths, formats, dimensions, variants (workspace + installed external)
  LocaleCatalog   — locale files, fallback chains, string classifications
  StyleGuide      — RULEs and TOKENs (design constraints, typography, spacing)
  Boundaries      — what this asset does not cover
  Contracts       — design constraints exported to consumers
```

**Rules:**
- Section names are stable identifiers. Renaming a section requires updating all
  `[SEC:]` tags that reference it.
- Subsections use `{SectionName}.{number}`: `Functions.1`, `Architecture.2`.
- Tags reference sections by name: `[SEC:Functions.1]`, `[SEC:Scenarios]`.
- Custom sections are allowed. Add them to the SECTIONS block. The agent treats
  them like any other section.
- The SECTIONS block, when present, is the first thing the agent reads. It
  determines what to look for and how to slice context.
- Without a SECTIONS block, the agent discovers sections from `## ` headers.
  Cross-references and slicing still work — the agent just can't detect
  missing sections against a declaration.

---

## Abstract

**One paragraph.** What this spec defines, in plain English. A developer reading only
this paragraph should understand the purpose, scope, and primary value proposition.

**For SYSTEM specs:**

```
{PROJECT_NAME} is a {one-line description}.

It does {primary capability}. It does NOT do {explicit non-goals}.

The primary consumer of this system is {who calls it / who uses it}.

The implementation target is {language/platform}. The expected binary/artifact is {what}.
```

**For PATTERN specs:** Describe the architectural problem this pattern solves.

```
{PATTERN_NAME} is a reusable architectural pattern for {what problem it solves}.

Use this pattern when {conditions}. Do NOT use it when {anti-conditions}.

This pattern is consumed by SYSTEM specs via "USES PATTERN: {name}".
```

**For ASSET specs:** Describe what resources exist and how agents should use them.

```
{ASSET_NAME} catalogs {what resources} located in {workspace paths}.

This spec defines the style guide and usage constraints for these assets.
Agents reference this spec to know which assets to use, at what sizes, in
what contexts, and which design rules to follow.

This asset is consumed by SYSTEM specs via "USES ASSET: {name}".
```

**If this spec covers multiple components** (e.g., a system with distinct subsystems
that share a codebase), list them here:

```
COMPONENTS:
  {Component A}: {one-line purpose}
  {Component B}: {one-line purpose}
  {Component C}: {one-line purpose}

Each component gets its own subsections in DataModel, Functions, API, and Errors.
For example: DataModel.1 covers Component A's RECORDs, DataModel.2 covers
Component B's RECORDs. This keeps the spec navigable — the agent can read
only the component it's working on.
```

---

## Problem

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

## Architecture

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

### Architecture.1 Component Inventory

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

### Architecture.2 Data Flow

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

### Architecture.3 Patterns and Assets

**Declare which architectural patterns and assets this system uses.** For prior art
patterns, the agent uses its training knowledge — just name them. For novel patterns
(full PATTERN specs), the agent reads the referenced spec. For assets, the agent
reads the referenced asset spec or manifest.

```
USES PATTERN: {PatternName}
  applied_to  : {which component or layer}
  customization: {any project-specific adjustments, or "none"}

USES PATTERN: {NovelPatternName} FROM {pattern-spec.md}
  applied_to  : {which component or layer}
  customization: {any project-specific adjustments, or "none"}

USES ASSET: {AssetName} FROM {asset-spec.md}
  applied_to  : {which component or layer}
  customization: {any project-specific adjustments, or "none"}
```

**Prior art patterns** (agent already knows these — just declare usage):
```
USES PATTERN: Repository
  applied_to  : data access layer
  customization: "All repositories are async, return Result types"

USES PATTERN: CircuitBreaker
  applied_to  : external service calls
  customization: "5 failure threshold, 30s recovery window"
```

**Novel patterns** (agent reads the full spec for implementation details):
```
USES PATTERN: ContextStack FROM context-stack-pattern.md
  applied_to  : runtime state management
  customization: "Base frame includes hardware manifest constraints"
```

**Assets** (agent reads the asset spec for file locations and usage rules):
```
USES ASSET: DesignSystem FROM design-system-spec.md
  applied_to  : all UI components
  customization: "Dark mode as default theme"
```

**Rules for the agent:**
- For `USES PATTERN:` without a FROM, use your training knowledge of the named pattern.
  Implement it according to the language idioms in Abstract section. Apply the customization.
- For `USES PATTERN: X FROM spec.md`, read the referenced spec (TYPE: PATTERN).
  Implement according to that spec.
- For `USES ASSET: X FROM spec.md`, read the referenced spec (TYPE: ASSET).
  Assets already exist in the workspace at the paths listed in the spec's FileStructure section.
  Follow the style guide in the spec's Functions section. Honor all design constraints in
  the spec's Contracts section EXPORTS.
- Do NOT invent patterns or assets not declared here. If the implementation needs a
  pattern not listed, STOP and report: "Component X would benefit from {pattern}.
  Should I add it to Architecture.3?"

---

## DataModel

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

### DataModel.1 Core Records

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

### DataModel.2 Enumerations

```
ENUM {EnumName}:
  VALUE_A    -- {when this value is used}
  VALUE_B    -- {when this value is used}
  VALUE_C    -- {when this value is used}
```

### DataModel.3 Type Aliases

```
ALIAS EntityId = String         -- UUID v4 format
ALIAS Timestamp = i64           -- Unix milliseconds, UTC
ALIAS Duration = i64            -- Milliseconds
```

---

## Functions

**Every function the system exposes or uses internally.** Each function is specified with:
inputs, outputs, preconditions, postconditions, error conditions, and behavioral notes.

**For ASSET specs:** Functions section becomes the **style guide** — the design rules and
tokens the agent must follow when writing code that references these assets. Use
RULE blocks instead of FUNCTION blocks:

```
RULE {rule_name}:
  applies_to  : {what this rule governs — e.g., "all buttons", "page layout", "typography"}
  constraint  : {the rule itself — e.g., "minimum touch target is 44x44px"}
  rationale   : {why this rule exists}
  exceptions  : {when it's OK to deviate, or "none"}

TOKEN {token_name}:
  category    : {color | spacing | typography | breakpoint | shadow | border}
  value       : {the value — e.g., "#1a73e8", "16px", "Inter 400"}
  usage       : {when to use this token}
  css_var     : {CSS custom property name, if applicable — e.g., "--color-primary"}
```

Example style guide:
```
RULE minimum-touch-target:
  applies_to  : all interactive elements (buttons, links, form controls)
  constraint  : minimum 44x44px touch target area
  rationale   : WCAG 2.5.5 accessibility requirement
  exceptions  : inline text links within paragraphs

RULE color-contrast:
  applies_to  : all text on backgrounds
  constraint  : minimum 4.5:1 contrast ratio (AA), 7:1 for large text (AAA)
  rationale   : WCAG 2.1 Level AA compliance
  exceptions  : decorative text, disabled states

TOKEN color-primary:
  category    : color
  value       : #1a73e8
  usage       : primary actions, links, active states
  css_var     : --color-primary

TOKEN spacing-unit:
  category    : spacing
  value       : 8px
  usage       : base spacing unit. All spacing is a multiple of this value.
  css_var     : --spacing-unit

TOKEN font-body:
  category    : typography
  value       : Inter, 16px/1.5, 400 weight
  usage       : all body text
  css_var     : --font-body

TOKEN breakpoint-mobile:
  category    : breakpoint
  value       : 768px
  usage       : below this width, switch to mobile layout
  css_var     : --breakpoint-mobile
```

Localization rules (if the ASSET spec includes locale strings):
```
RULE no-hardcoded-strings:
  applies_to  : all display text in UI components
  constraint  : every user-visible string must come from a locale file via a
                string key. Never hardcode text in templates or components.
  rationale   : enables localization without code changes
  exceptions  : none

RULE fallback-chain:
  applies_to  : string resolution at runtime
  constraint  : resolve strings as: exact locale → language locale → source locale.
                Example: fr-CA → fr → en.
  rationale   : partial translations should still render, not break
  exceptions  : none

RULE curated-string-integrity:
  applies_to  : strings classified as human_curated in the LOCALE CATALOG
  constraint  : never auto-translate. If missing in target locale, display the
                fallback locale version. Flag as a translation gap in build output.
  rationale   : brand voice, legal text, and cultural nuance require human review
  exceptions  : none
```

The agent reads these RULEs and TOKENs before generating any UI code. When a
SYSTEM spec declares `USES ASSET: DesignSystem`, the agent must honor every RULE
and use every TOKEN from that ASSET spec's Functions section.

### Conventions (for SYSTEM specs)

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

### Functions.1 {Function Group Name}

```
FUNCTION {name}({params}) -> {return}
  ...
```

---

## API

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

### API.1 {API Group}

```
ENDPOINT ...
```

---

## Errors

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

## Config

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

### Config.1 {Config Group}

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

## Deployment

**Infrastructure-as-code, manifests, and deployment definitions.** Not every spec needs
this section — skip it for pure libraries and PATTERN specs. Include it for anything
that gets deployed (services, infrastructure, controllers). ASSET specs include this
only if assets need to be deployed to a CDN or artifact registry.

### Deployment.1 Container / Image

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

### Deployment.2 Kubernetes Manifests

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

### Deployment.3 Infrastructure Dependencies

```
INFRA {name}:
  type: {GPU | storage | network | certificate | DNS | etc.}
  required: {true | false}
  specification: {what exactly is needed}
  provisioning: {how it gets created — manual, Terraform, Helm, etc.}
  verification: {how to confirm it's ready}
```

### Deployment.4 Deployment Order

```
DEPLOY ORDER:
  1. {what gets deployed first and why}
  2. {what depends on step 1}
  3. {what depends on step 2}

  Verification between steps:
  - After step 1: {check command and expected output}
  - After step 2: {check command and expected output}
```

### Deployment.5 CI/CD Pipelines

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

### Deployment.6 File-to-Section Mapping

**Maps source files to spec sections for automatic AFFECTED scenario determination.**
When a file changes, the CI pipeline uses this map to identify which scenarios to run.

```
FILE_TO_SECTION_MAP:
  {src/path/to/module/*}   → [{SEC:x.x}]
  {src/path/to/api/*}      → [{SEC:x.x}]
  {src/path/to/errors.*}   → [{SEC:x}]
  {src/path/to/config.*}   → [{SEC:x}]
  {deploy/*}               → [{SEC:Deployment}]
```

---

## Scenarios

**End-to-end validation scenarios.** These are NOT unit tests — they are behavioral
specifications. Each scenario describes a complete user journey with expected outcomes.
The coding agent must make ALL scenarios pass. Scenarios are the "holdout set" —
they validate the system works correctly end-to-end.

**For PATTERN specs:** Scenarios validate the pattern's constraints and interface
contracts, not a running system. For example, a Repository pattern scenario might
verify that all data access goes through the repository interface, not directly to
the database.

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

### Scenarios.1 Happy Path Scenarios

```
SCENARIO 1: {Basic successful operation}
  GIVEN:
  - ...
  WHEN:
  - ...
  THEN:
  - ...
```

### Scenarios.2 Error Scenarios

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

### Scenarios.3 Performance Scenarios

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

### Scenarios.4 Edge Case Scenarios

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

## Dependencies

**External systems, libraries, and asset packs this project requires.** Every dependency
the agent needs to install, fetch, or pull before building. The agent honors these
declarations during IMPLEMENT — it does not substitute alternatives, upgrade versions,
or skip platform constraints without human approval.

```
DEPENDENCY {name}:
  category    : {runtime | build | external-asset}
  purpose     : {why we need it}
  version     : {exact version, range, or "latest" — see pinning rules below}
  source      : {where to get it — npm, PyPI, crates.io, Docker Hub, CDN, etc.}
  install     : {exact command to install}
  required    : {true | false — can the system function without it?}
  interface   : {library | CLI | API | base-image | asset-pack | socket}
  platform    : {architecture/OS constraint — e.g., "linux/arm64", "any"}
  verify      : {command or check to verify the install succeeded and matches the spec}
  fallback    : {what happens if it's unavailable}
```

### Version Pinning Rules

```
RULE: When `version` specifies an exact version (e.g., "3.1.0", "v2.4"),
  the agent installs EXACTLY that version. No upgrades, no "compatible" substitutes.

RULE: When `version` specifies a range (e.g., "^3.1.0", ">=2.4 <3.0"),
  the agent resolves within that range and records the resolved version in a lockfile.

RULE: When `version` says "latest", the agent installs the latest available
  at implementation time and records the resolved version in the lockfile or
  build output. The spec author accepts that this may change between builds.

RULE: When `platform` is specified, the agent MUST verify the dependency supports
  that platform. If building a Docker image, the base image must support the declared
  architecture. Do NOT silently fall back to a different platform.

RULE: The agent NEVER substitutes a different library for the one declared.
  "Use React instead of Preact" is a spec change, not an implementation choice.
```

### Verify After Install — Critical Anti-Hallucination Rule

**The agent's training knowledge about a dependency's API, config format, or asset
names may be wrong for the pinned version.** This is the most common source of
subtle bugs in agent-generated code: the agent installs version X but writes code
using version Y's API because that's what its training data contains.

```
RULE: After installing any dependency, the agent MUST verify its generated code
  against the ACTUAL installed artifact — not against training knowledge.

RULE: The `verify` field in the DEPENDENCY declaration tells the agent HOW to check.
  If no `verify` field exists, the agent uses these defaults by interface type:

  library:
    1. After install, read the installed package's type definitions, API surface,
       or module exports (e.g., node_modules/{name}/index.d.ts, or the package's
       __init__.py, or cargo doc output)
    2. Compare every import and function call in the generated code against the
       actual exports of the installed version
    3. If a function/class/type used in code doesn't exist in the installed version,
       fix the code BEFORE running tests — don't wait for test failures

  CLI:
    1. Run `{name} --version` or `{name} version` to confirm the version matches
    2. Run `{name} --help` or read the man page to verify subcommands and flags
       used in the generated scripts/config actually exist
    3. If the spec generates config files for the CLI tool, verify the config
       format matches the installed version's expected schema

  base-image:
    1. Pull the image and inspect: `docker inspect {image}:{tag}`
    2. Verify the declared platform matches: check Architecture field
    3. If the Dockerfile uses features from the base image (installed packages,
       entrypoint, exposed ports), verify they exist in the actual image

  asset-pack:
    1. After install, list the actual available assets
       (e.g., ls node_modules/lucide-react/dist/esm/icons/)
    2. Compare every asset reference in the generated code against the actual
       available assets
    3. If an icon/font/component name used in code doesn't exist in the installed
       pack, fix the reference BEFORE running tests

  API:
    1. If the dependency provides a health or version endpoint, check it
    2. If the dependency provides an OpenAPI/GraphQL schema, fetch it and
       verify the generated client code matches the actual schema

RULE: If the `verify` check fails, classify as IMPL_BUG (not SPEC_GAP).
  The spec correctly declared the dependency and version. The agent's code was
  wrong for that version. Fix the code to match the installed reality.

RULE: If the installed version's API is fundamentally different from what the
  spec's FUNCTIONs expect (e.g., the spec says "call airflow.trigger_dag()"
  but the installed version has no such function), classify as SPEC_GAP.
  The spec needs updating to match the dependency's actual API. File a patch.
```

### Dependencies.1 Runtime Dependencies

Dependencies needed at runtime. The built artifact requires these to run.

Example:
```
DEPENDENCY airflow:
  category    : runtime
  purpose     : Workflow orchestration engine
  version     : "3.1.0"
  source      : PyPI
  install     : pip install apache-airflow==3.1.0
  required    : true
  interface   : library
  platform    : any
  verify      : "python -c 'import airflow; print(airflow.__version__)' outputs '3.1.0'
                 AND python -c 'from airflow.sdk import DAG' succeeds (v3 uses sdk module)"
  fallback    : none — core dependency

DEPENDENCY postgres:
  category    : runtime
  purpose     : Primary database
  version     : "16"
  source      : Docker Hub
  install     : docker pull postgres:16
  required    : true
  interface   : base-image
  platform    : linux/arm64
  verify      : "docker inspect postgres:16 shows Architecture=arm64"
  fallback    : none — required for persistence
```

### Dependencies.2 Build Dependencies

Dependencies needed to build but not at runtime.

Example:
```
DEPENDENCY rust:
  category    : build
  purpose     : Compiler toolchain
  version     : "1.75"
  source      : rustup
  install     : rustup install 1.75.0
  required    : true
  interface   : CLI
  platform    : linux/arm64
  verify      : "rustc --version outputs 'rustc 1.75.x'
                 AND rustc --print target-list | grep aarch64-unknown-linux"
  fallback    : none
```

### Dependencies.3 External Asset Packs

Assets that are NOT in the workspace and must be fetched before use. Icon libraries,
font packages, design token packs, UI component libraries. The agent installs them
during IMPLEMENT and uses them according to the rules declared here or in a
referenced ASSET spec.

Example:
```
DEPENDENCY lucide-react:
  category    : external-asset
  purpose     : Icon library — all UI icons must come from this pack
  version     : "0.263.1"
  source      : npm
  install     : npm install lucide-react@0.263.1
  required    : true
  interface   : asset-pack
  platform    : any
  verify      : "After install, run: ls node_modules/lucide-react/dist/esm/icons/
                 Then verify every icon import in the codebase exists in that directory.
                 Common renames across versions: Edit2→Pencil, Trash2→Trash"
  fallback    : none — do NOT use inline SVGs for icons that exist in this library

  USAGE RULES:
  - Import by name: `import { Camera, Trash, Search } from 'lucide-react'`
  - Do NOT use icons from any other library (no FontAwesome, no Heroicons)
  - Do NOT inline SVGs for icons that exist in lucide-react
  - If an icon doesn't exist in lucide-react, ask the human — do NOT substitute
  - See brand-assets-spec.md for icon sizing and color rules (if applicable)

DEPENDENCY inter-font:
  category    : external-asset
  purpose     : Primary UI typeface
  version     : "4.0"
  source      : Google Fonts
  install     : "Add to HTML: <link href='https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700' rel='stylesheet'>"
  required    : true
  interface   : asset-pack
  platform    : any
  verify      : "Font loads in browser — check Network tab for fonts.googleapis.com 200 response"
  fallback    : "font-family: system-ui, -apple-system, sans-serif"

  USAGE RULES:
  - Use Inter for all body text and UI elements
  - Weights: 400 (body), 500 (labels), 600 (headings), 700 (emphasis)
  - Do NOT use other fonts unless the ASSET spec explicitly declares them
```

---

## FileStructure

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

**For ASSET specs:** FileStructure section becomes the asset catalog — mapping every
asset with its metadata. Assets come from two sources:

- **Workspace assets** — files that already exist on disk. The spec catalogs them.
- **External assets** — assets from an installed DEPENDENCY (icon pack, font package,
  component library). The ASSET spec declares which DEPENDENCY provides the assets,
  and the agent reads the installed contents to verify the catalog.

```
ASSET CATALOG:

  --- Workspace assets (already on disk) ---

  ASSET {name}:
    source      : workspace
    path        : {workspace-relative path, e.g., "assets/logo-dark.svg"}
    format      : {file format: SVG, PNG, CSS, WOFF2, JSON, etc.}
    description : {what this asset is and when to use it}
    dimensions  : {width x height if image — e.g., "200x50"}
    variants    : {list of size/context variants, if applicable}

  --- External assets (installed from a DEPENDENCY) ---

  ASSET {name}:
    source      : dependency:{dependency_name}
    path        : {path within installed package, e.g., "lucide-react/dist/esm/icons/"}
    format      : {format — e.g., "React component", "SVG", "WOFF2"}
    description : {what this asset pack provides}
    includes    : {which assets from the pack to use — "all" or a curated list}
    excludes    : {which assets to NOT use, if applicable}
```

**REQUIRES DEPENDENCY declaration:**

When an ASSET spec depends on an external package, declare it at the top of the
FileStructure section. This tells the agent to install the dependency before
reading the catalog.

```
REQUIRES DEPENDENCY: {name} FROM {system-spec.md}
```

This references a DEPENDENCY element declared in the consuming SYSTEM spec's
Dependencies section. The ASSET spec doesn't own the install — the SYSTEM spec does.
The ASSET spec just says "I need this installed to catalog its contents."

Example — mixed workspace and external assets:
```
REQUIRES DEPENDENCY: lucide-react FROM app-spec.md
REQUIRES DEPENDENCY: inter-font FROM app-spec.md

ASSET CATALOG:

  --- Workspace assets ---

  ASSET logo-dark:
    source      : workspace
    path        : assets/images/logo-dark.svg
    format      : SVG
    description : Primary logo for dark backgrounds. Use in nav bar and login page.
    dimensions  : "200x50 (native), scales to container width"
    variants    :
      - favicon: assets/images/favicon-32.png (32x32, PNG)
      - og-image: assets/images/og-logo.png (1200x630, PNG)

  ASSET base-styles:
    source      : workspace
    path        : styles/base.css
    format      : CSS
    description : Global reset, typography, spacing scale, color variables.
                  Import this in every page. Do NOT override CSS custom properties
                  defined here — extend them in component-level styles.

  ASSET design-tokens:
    source      : workspace
    path        : styles/tokens.json
    format      : JSON
    description : Machine-readable design tokens (colors, spacing, typography,
                  breakpoints). Generated from the style guide. The agent reads
                  this file when producing any UI code.

  --- External assets (from installed dependencies) ---

  ASSET icons:
    source      : dependency:lucide-react
    path        : node_modules/lucide-react/dist/esm/icons/
    format      : React component (ESM)
    description : UI icon library. All icons in the application come from this pack.
    includes    : all — agent may use any icon in the pack
    excludes    : none

  ASSET primary-font:
    source      : dependency:inter-font
    path        : (loaded via Google Fonts CDN — no local path)
    format      : WOFF2 (remote)
    description : Primary UI typeface. Loaded via <link> tag in HTML head.
    includes    : "weights 400, 500, 600, 700"
    excludes    : "weights 100-300, 800-900 — not in the brand guidelines"
```

**Rules for external asset catalogs:**

```
RULE: The agent MUST install the required DEPENDENCY before reading the catalog.
  Do NOT catalog assets that haven't been installed yet.

RULE: After installation, the agent verifies the catalog against the actual
  installed contents. If the catalog lists an asset that doesn't exist in the
  installed package, classify as SPEC_GAP (catalog is wrong) or IMPL_BUG
  (wrong version installed).

RULE: If `includes` is "all", the agent may use any asset in the installed pack.
  If `includes` is a curated list, the agent uses ONLY those assets and treats
  any other asset in the pack as if it doesn't exist.

RULE: The `excludes` field overrides `includes: all`. If an asset is excluded,
  the agent MUST NOT use it even though it's available.

RULE: For external assets, the agent reads the ACTUAL installed contents to
  discover asset names — not training knowledge. Icon names, component names,
  and font weights can change between versions.
```

### Localization Strings

Locale files are assets. They live in the workspace and are cataloged like any
other asset. The key distinction is that some strings are agent-translatable
(the agent can generate them for new locales) while others are human-curated
(the agent must use them verbatim from the locale file).

```
LOCALE CATALOG:
  source_locale : {the authoritative locale — e.g., "en"}
  fallback_chain: {resolution order — e.g., "fr-CA → fr → en"}
  format        : {file format — e.g., "JSON (flat key-value)", "JSON (nested)", "YAML"}
  interpolation : {variable syntax — e.g., "{variable}", "{{variable}}", "%{variable}"}

  LOCALE {locale_code}:
    path        : {workspace path, e.g., "locales/en.json"}
    status      : {complete | partial | stub}
    coverage    : {percentage of source_locale keys present}

  LOCALE {locale_code}:
    path        : {workspace path}
    ...
```

```
STRING CLASSIFICATION:

  STRING_CLASS agent_translatable:
    description : Straightforward UI labels, button text, form fields, error messages.
                  The agent can generate these for new locales from the source strings.
    keys        : {list of key patterns, e.g., "nav.*", "button.*", "form.label.*"}
    review      : optional

  STRING_CLASS human_curated:
    description : Brand copy, legal text, culturally sensitive content, idioms, humor,
                  marketing slogans. The agent must use these VERBATIM from the locale
                  file. If a key is missing in a locale, fall back — do NOT auto-translate.
    keys        : {list of key patterns, e.g., "legal.*", "marketing.*", "onboarding.welcome"}
    review      : required

  STRING_CLASS interpolated:
    description : Strings with variables. The agent must preserve all placeholders and
                  handle pluralization per locale rules.
    keys        : {list of key patterns, e.g., "messages.item_count", "messages.time_ago"}
    pluralization: {rule reference — e.g., "CLDR plural rules per locale"}
    review      : required for human_curated, optional for agent_translatable
```

Example:
```
LOCALE CATALOG:
  source_locale : en
  fallback_chain: "regional → language → en" (e.g., fr-CA → fr → en)
  format        : JSON (flat key-value)
  interpolation : {variable}

  LOCALE en:
    path        : locales/en.json
    status      : complete
    coverage    : 100%

  LOCALE es:
    path        : locales/es.json
    status      : complete
    coverage    : 100%

  LOCALE ja:
    path        : locales/ja.json
    status      : partial
    coverage    : 85%

STRING CLASSIFICATION:

  STRING_CLASS agent_translatable:
    description : Standard UI labels and form text
    keys        : "nav.*", "button.*", "form.label.*", "form.placeholder.*",
                  "error.validation.*", "table.header.*"
    review      : optional

  STRING_CLASS human_curated:
    description : Brand voice, legal, and culturally nuanced content
    keys        : "legal.*", "marketing.*", "onboarding.welcome",
                  "onboarding.value_prop", "error.friendly.*"
    review      : required

  STRING_CLASS interpolated:
    description : Strings with dynamic values
    keys        : "messages.item_count", "messages.time_ago", "messages.greeting"
    pluralization: CLDR plural rules (one/few/many/other per locale)
    review      : required for human_curated keys, optional otherwise
```

**Rules for the agent:**
- Always load strings from locale files. Never hardcode display text.
- For `agent_translatable` strings missing in a target locale, the agent MAY
  generate a translation from the source locale string.
- For `human_curated` strings missing in a target locale, the agent MUST fall
  back through the fallback chain. Never auto-translate these.
- For `interpolated` strings, preserve all `{variable}` placeholders. Handle
  pluralization according to the locale's CLDR plural rules.
- When adding new UI strings during implementation, add the key to the source
  locale file, classify it, and note missing translations for other locales.

---

## Maintenance

**How to handle bugs, changes, and iteration after initial implementation.**

The spec is the source of truth, not the code. Code is a derived artifact of the spec.
When something is wrong, determine which category the issue falls into and follow
the corresponding workflow.

### Maintenance.1 Bug Categories

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

### Maintenance.2 Patch Spec Format

A patch spec is a small, focused document used for Category B fixes. It is temporary
and gets absorbed into the main spec at the next version bump.

```markdown
# PATCH: {short description}

> **Patches:** {SPEC_NAME} v{version}
> **Section:** {section name and name}
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

### Maintenance.3 What the Agent Receives — Modular Context, Not Full Spec

**Never feed the full spec for a bug fix.** The full spec is for initial implementation
and major version bumps ONLY. For patches and fixes, feed the minimum context needed.

The spec is designed to be sliceable by section name. Each FUNCTION references which
RECORDs it uses. Each SCENARIO references which FUNCTIONs it tests. This dependency
chain determines the context slice.

```
CONTEXT SLICE for a bug fix:
  1. The PATCH SPEC or updated SECTION (the change itself)
  2. The failing SCENARIO(s) (what must now pass)
  3. The RECORDs referenced by the affected FUNCTION (data structures it touches)
  4. The ERROR MODEL entries relevant to the function
  5. Any PATTERN constraints that apply to the affected component (from Architecture.3)
  6. Any ASSET constraints that apply (RULEs and TOKENs from referenced ASSET spec Functions section)
  7. Any seed rules from the seed manifest that constrain this area
  8. NOTHING ELSE

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

### Maintenance.4 Scenario Tiers — Not All Scenarios Run Every Time

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
    - Tagged with the section they validate: [SEC:Functions.3] [SEC:DataModel.1]

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

### Maintenance.5 Scenario Dependency Tags

Each SCENARIO declares which spec sections it validates. This enables automatic
determination of which scenarios to run when a section changes.

```
SCENARIO 7: Query returns results within latency budget  [SEC:Functions.3] [SEC:DataModel.1] [SMOKE]
  GIVEN:
  - Catalog loaded with 1000 tables
  WHEN:
  - POST /search {"query": "customer churn"}
  THEN:
  - Response contains ranked results
  - Latency < 10ms
```

When Functions section.3 is patched, the agent runs:
- All [SMOKE] scenarios (always)
- All [SEC:Functions.3] scenarios (affected by the change)
- NOT scenarios tagged [SEC:API.2] or [SEC:Config.1] (unrelated)

### Maintenance.6 Spec Versioning Rules

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

### Maintenance.7 Scenario as Regression Guard

Every bug fix (any category) MUST add or identify a SCENARIO that prevents regression.
SCENARIOs are append-only — you never remove a passing SCENARIO. The SCENARIO count
monotonically increases. This is the equivalent of StrongDM's "holdout set" — the
scenarios exist outside the agent's ability to modify them.

```
Rule: If a bug was found and fixed, there MUST be a SCENARIO that would catch
it if it reappeared. If no such SCENARIO exists, the fix is incomplete.
```

---

## BuildAndRun

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

### Artifacts

**What a successful build produces.** Machine-checkable output contract. The agent
verifies these after IMPLEMENT and VALIDATE. Downstream specs can depend on these
artifacts existing and matching their declared properties.

```
ARTIFACT {name}:
  type        : BINARY | CONTAINER | LIBRARY | CONFIG | STATIC | BUNDLE
  path        : {where the artifact lives after build — relative to project root}
  produced_by : {which build command creates this}
  checkable   : {how to verify it exists and is correct}
  properties:
    {key}     : {value — e.g., exposes_port: 8080, binary_name: "kv-server"}
```

Example:
```
ARTIFACT server-binary:
  type        : BINARY
  path        : target/release/kv-server
  produced_by : cargo build --release
  checkable   : file exists AND file is executable AND ./kv-server --version prints version
  properties:
    binary_name : kv-server
    min_size    : 1MB

ARTIFACT docker-image:
  type        : CONTAINER
  path        : docker.io/{org}/kv-server:{version}
  produced_by : docker build -t kv-server .
  checkable   : docker inspect kv-server:{version} succeeds
  properties:
    exposes_port : 8080
    base_image   : rust:1.75-slim

ARTIFACT api-schema:
  type        : STATIC
  path        : docs/openapi.yaml
  produced_by : cargo run -- --export-schema > docs/openapi.yaml
  checkable   : file exists AND valid OpenAPI 3.x
  properties:
    format     : openapi-3.1
```

**Rules:**
- Every SYSTEM spec should declare at least one ARTIFACT (the primary deliverable).
- PATTERN and ASSET specs typically produce LIBRARY artifacts.
- Artifacts are verified after IMPLEMENT (do they exist?) and after VALIDATE
  (do they still exist and pass checks after all scenarios run?).
- Downstream specs can reference artifacts in their EXPECTS declarations:
  `EXPECTS ServerBinary FROM kv-store-spec.md` means the dependency must produce
  that artifact before this spec can be implemented.
```

---

## Boundaries

**What this spec explicitly does NOT cover.** Prevents scope creep during implementation.

```
### This Spec Does:
- {capability}
- {capability}

### This Spec Does NOT:
- {non-goal} — {why not, and what handles this instead}
- {non-goal} — {why not}

### Future Extensions (not in this spec):
- {feature that might be added later but is explicitly out of scope now}
```

---

## Contracts

**What this spec exports to consumers and expects from dependencies.** When spec A
declares `IMPORT spec B`, B's exports are automatically seeded as rules in A's runtime.
This section is machine-readable — the seed resolver uses it to produce deterministic
initialization state.

If this spec has no imports and is not imported by others, state:
`EXPORTS: none — this spec is standalone.`

### Contracts.1 Exports

```
EXPORT {ExportName}:
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  target      : {which component/layer in the consumer receives this}
  condition   : {when this export applies — "ALWAYS" if unconditional}
  value       : {the constraint, rule, or data being exported}
  override    : NEVER | WITH_JUSTIFICATION | UNRESTRICTED
  source_ref  : [SEC:SectionName.x]  -- which section of THIS spec defines the basis
```

**Export types:**
- `CONSTRAINT` — a limit that restricts behavior
- `INVARIANT` — a property that must always hold at runtime
- `SEED_DATA` — initial state to populate at deploy time
- `POLICY` — a behavioral default

**Override levels:**
- `NEVER` — cannot be overridden. Two conflicting NEVER exports cause a build failure.
- `WITH_JUSTIFICATION` — can be overridden if the consumer documents why.
- `UNRESTRICTED` — consumer can freely override.

**For PATTERN specs:** Exports are architectural constraints enforced at implementation
time. Example: a Repository pattern exports "all data access must go through a
repository interface." The agent must follow this when implementing any SYSTEM spec
that declares `USES PATTERN: Repository`.

**For ASSET specs:** Exports are design constraints from the style guide (Functions section
RULEs and TOKENs). Example: a design system exports "minimum touch target is 44px",
"primary color is #1a73e8." These are enforced when generating UI code in any SYSTEM
spec that declares `USES ASSET:`. The agent reads the ASSET spec's Functions section (style
guide) and FileStructure section (asset catalog) to know what files exist and how to use them.

### Contracts.2 Expects

```
EXPECTS {ExpectName}:
  from        : {spec_id or "ANY_DEPENDENCY"}
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  description : {what this spec needs and why}
  required    : true | false
  fallback    : {default value if required=false and no dependency provides it}
```

### Contracts.3 Conflict Resolution

When multiple dependencies export constraints that affect the same target:

1. `override=NEVER` exports always win. If two NEVER exports conflict, the build fails.
2. More restrictive constraint wins (lower limit, stricter policy).
3. Spec closer to the dependency root (fewer hops) wins.
4. If still ambiguous, the build fails with a clear error.

### Contracts.4 Notes for Spec Authors

- Every spec that other specs depend on SHOULD define its EXPORTS explicitly.
- If your spec has no exports, state: `EXPORTS: none — this spec is a leaf.`
- The seed resolver walks the full dependency graph and collects all exports before
  any code is generated. Conflicts are caught at build time, not runtime.

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
IMPORT ConstraintMask FROM cra-spec.md DataModel.3
IMPORT KernelDefinition FROM cra-spec.md DataModel.5
IMPORT QueryResponse FROM search-service-spec.md DataModel.2
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
