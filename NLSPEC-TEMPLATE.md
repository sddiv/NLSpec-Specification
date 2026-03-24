# {PROJECT_NAME} — Natural Language Specification

> **Version:** 0.1.0
> **Author:** {AUTHOR}
> **Date:** {DATE}
> **Type:** {SYSTEM | PATTERN | ASSET}
> **Language Target:** {Rust | Go | TypeScript | Python | Language-Agnostic}
> **Status:** {Draft | Review | Approved | Implementing | Complete}
> **Layer:** {1-Specification | 2-Realization | 3-Configuration | 4-UserProfile | —}

The **Layer** field is optional. Omit it (or set to `—`) for specs that are not part of a
layered composition. When present, it declares where this spec sits in the 4-layer
composition model:

| Layer | Name | What It Defines |
|-------|------|-----------------|
| 1 | Specification | Contracts, interfaces, invariants, pluggable boundaries. Defines the possibility space. |
| 2 | Realization | Makes Layer 1 concrete for a specific platform or technology. Derives from an L1 spec. |
| 3 | Configuration | Tunes a Layer 2 realization for a specific domain or deployment. Derives from an L2 spec. |
| 4 | User Profile | Preferences, overrides, personalization constraints. Derives from an L3 spec. |

Each layer derives from the layer below it and adds its own constraints. A spec at any
layer uses the same template — the layer declaration affects how agents interpret
derivation, contracts, and constraint flow, not which sections are available.

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
  SystemManifest  — platform composition, hardware, kernels, environment configs
  Deployment      — containers, manifests, pipelines, CI/CD
  Scenarios       — behavioral validation (the holdout set)
  Dependencies    — external systems, libraries, and asset packs with pinned versions
  FileStructure   — expected directory layout
  Maintenance     — bug workflow, patches, scenario tiers, versioning
  BuildAndRun     — exact commands to build, test, run, verify, and artifact manifest
  AgentPipeline   — AI agent execution protocol (provision → build → deploy → validate)
  Boundaries      — explicit non-goals
  Contracts       — exports, expects, conflict resolution
  LayerContext    — (optional) layer composition: derivation, stack, constraint flow
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
  LayerContext    — (optional) layer composition: which layers this pattern applies to
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
  LayerContext    — (optional) layer composition: which layers these assets serve
```

**Rules:**
- Section names are stable identifiers. Renaming a section requires updating all
  `[SEC:]` tags that reference it.
- Subsections use `{SectionName}.{number}`: `Functions.1`, `Architecture.2`.
- Tags reference sections by name: `[SEC:Functions.1]`, `[SEC:Scenarios]`.
- Cross-layer references use `[Ln:{spec-id}:{Section}.{subsection}]` to point to
  elements in specs at other layers. Example: `[L1:product-spec:Functions.3]`.
  See LayerContext.5 for full syntax and rules.
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

## SystemManifest

**Platform composition, hardware requirements, kernel declarations, and environment
configurations for systems that deploy ON a platform.** Skip this section for standalone
systems that don't run on a platform (e.g., a library or CLI tool). Include it when the
system deploys as an application on an infrastructure platform (e.g., a CRA application
deploying on the CRA platform, a microservice on Kubernetes, or an edge application on
specific hardware).

The SystemManifest bridges the gap between "what this system does" (the rest of the spec)
and "what it needs to run" (hardware, platform services, runtime configuration). For
systems that use an external manifest file (TOML, YAML), this section declares the
manifest location and summarizes its contents. For simpler systems, declare inline.

### When to Use an External Manifest File

Use an external manifest file when:
- The system deploys in multiple environments (dev, staging, production)
- Hardware and kernel declarations are complex enough to warrant machine-parseable format
- Multiple specs share a manifest template

Use inline SystemManifest when:
- The system has a single deployment target
- Hardware requirements are simple (e.g., "any Linux x86_64, 2GB RAM")

### External Manifest Reference

```
SYSTEM MANIFEST: {manifest-filename.toml}
  template    : {path to manifest template}
  scope       : deployment and infrastructure configuration
```

The agent reads the referenced manifest file for hardware, kernel, and environment
details. All `SystemManifest.X` references in Deployment, BuildAndRun, and other
sections refer to the corresponding sections of that manifest instance.

### Inline SystemManifest

For simpler systems, declare the manifest inline:

```
SYSTEM MANIFEST (inline):

  COMPOSITION:
    profile       : {composition profile name — e.g., "FULL_CRA", "REFLEX_ONLY", "standalone"}
    platform      : {platform name and version — e.g., "CRA v0.1", "Kubernetes 1.28", "bare metal"}
    deployment_mode: {loop-integrated | boundary-only | either}

  HARDWARE:
    cpu           : {minimum cores}
    ram           : {minimum GB}
    gpu           : {required | optional | none} — {type if required}
    storage       : {minimum GB, type}
    architecture  : {x86_64 | aarch64 | any}
    network       : {requirements — e.g., "local-only", "internet required", "air-gapped"}

  KERNELS:
    {kernel_name}:
      type        : {safety | governance | domain | privacy | compliance}
      description : {one-line purpose}
      enforcement : {IP1 | IP2 | IP3 | all}

  ENVIRONMENT:
    {env_name} (e.g., development):
      {key}       : {value or description}
    {env_name} (e.g., production):
      {key}       : {value or description}

  SERVICE_INTEGRATION:
    {service_name}:
      type        : {database | message-queue | external-api | local-service}
      required    : {true | false}
      connection  : {how to connect — URL template, socket path, etc.}
```

### Rules

- If the spec declares `SYSTEM MANIFEST: {file}`, the agent reads that file for
  hardware and kernel details. The agent does NOT guess hardware requirements.
- Kernel names declared here must match kernel definitions in the DataModel and
  Functions sections. If a kernel is listed in the manifest but not defined in the
  spec, the agent reports a SPEC_GAP.
- Environment configurations in the manifest override Config section defaults for
  that environment. The Config section defines the knobs; the manifest sets the
  values per environment.

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

## UXSpec

**Scene and transition definitions for user-facing frontends.** This section is used by
SYSTEM specs that have a UI layer. It defines what the user sees (scenes), how they
navigate between views (transitions), and how visual correctness is verified.

UXSpec is design-system-agnostic. It references components from a registered design system
(declared via `USES ASSET: {DesignSystem} FROM {spec.md}`). The design system provides
the component catalog — UXSpec composes those components into scenes. Any design system
that exports a COMPONENT_CATALOG can be used: l2mflare, Material 3 generators, Shadcn,
custom systems.

**When to use UXSpec:** Any spec that declares screens, views, or user-facing workflows.
If the spec has a frontend, it should have a UXSpec section.

### UXSpec.1 Design System Binding

Declare which design system(s) this spec uses. The design system must be a TYPE: ASSET
spec with a COMPONENT_CATALOG section.

```
USES ASSET: {DesignSystemName} FROM {design-system-spec.md}
```

The ASSET spec's COMPONENT_CATALOG section defines each available component:

```
COMPONENT {ComponentName}:
  category     : {layout | input | display | navigation | feedback | overlay}
  description  : {what this component does}
  props:
    {prop_name} : {Type}                -- {description}
    {prop_name} : {Type} = {default}    -- {description}, with default
    {prop_name} : {Type} | None         -- {description}, optional
  variants:
    {variant_name} : {description of when to use}
  slots:
    {slot_name} : {what goes here — e.g., "children", "header", "footer"}
  states:
    default     : {base appearance}
    hover       : {hover state}
    focused     : {focus state}
    disabled    : {disabled state}
    active      : {active/selected state}
  tokens:
    {which design tokens this component uses — e.g., "color-primary, shape-md, elevation-level-1"}

COMPONENT ActionButton:
  category     : input
  description  : Workflow-initiating button with trailing arrow icon
  props:
    color       : "primary" | "secondary" | "tertiary" = "primary"
    size        : "lg" | "md" | "sm" = "md"
    outlined    : Boolean = false
    hideArrow   : Boolean = false
    disabled    : Boolean = false
    label       : String
    onClick     : Function
  variants:
    filled      : solid background, white text (default)
    outlined    : border only, colored text
  slots:
    children    : button label text
  states:
    default     : filled/outlined per variant
    hover       : 8% opacity overlay
    focused     : 2px primary outline
    disabled    : 38% opacity, no interaction
  tokens:
    color-primary, color-secondary, color-tertiary, shape-action-md, elevation-level-0
```

### UXSpec.2 Scenes

A SCENE is a viewport-filling composition of components — the equivalent of a
UIViewController (iOS), Activity (Android), or a route-level page component (web).
Each scene declares which components it uses, how they're laid out, and how they
respond to viewport changes.

```
SCENE {SceneName}:
  DESCRIPTION: {what the user sees and does on this scene}
  VIEWPORT: full | modal({width}x{height}) | drawer({width})

  LAYOUT:
    type        : {grid | flex | stack}
    {layout-specific properties — columns, rows, direction, gap, padding}

  RESPONSIVE:
    {breakpoint_condition}:
      {layout overrides — column changes, hidden elements, stacking changes}

  COMPONENTS:
    {slot_name}: {ComponentName}({prop overrides})
      {nested components — child slots}

  DATA_BINDINGS:
    {component.prop} ← {data source — API endpoint, local state, parent prop}

  Z_LAYERS:
    base ({z}):    {what lives at this layer}
    sticky ({z}):  {pinned elements}
    overlay ({z}): {modals, dropdowns}
    toast ({z}):   {notifications}

  OVERLAYS:
    {OverlayName}:
      TRIGGER: {what causes it to appear}
      TYPE: modal | drawer | dropdown | tooltip | toast
      BACKDROP: {CSS value or "none"}
      CONTENT: {component tree}
      DISMISS: {how to close it}
```

Example:

```
SCENE Dashboard:
  DESCRIPTION: Main landing screen. Shows active sessions, system health, and quick actions.
  VIEWPORT: full

  LAYOUT:
    type: grid
    columns: [sidebar, main]
    rows: [1fr]

  RESPONSIVE:
    >= 1024px:
      columns: [240px, 1fr]
    < 1024px:
      columns: [1fr]
      sidebar: hidden
      OVERLAY nav-drawer added (Sidebar content in drawer)

  COMPONENTS:
    sidebar: Sidebar(width="default")
      SidebarHeader(title="L2Mify")
      SidebarSection(title="NAVIGATION")
        SidebarItem(icon="dashboard", label="Dashboard", active=true)
        SidebarItem(icon="science", label="Mock", badge={active_mock_count})
        SidebarItem(icon="build", label="Build")
      SidebarDivider
      SidebarSection(title="TOOLS")
        SidebarItem(icon="hub", label="Component Graph")
        SidebarItem(icon="database", label="Knowledge Graph")
      SidebarFooter
        SidebarItem(icon="settings", label="Settings")

    main:
      LAYOUT: grid, rows: [header 64px, content 1fr]

      header:
        LAYOUT: flex, justify: space-between, align: center, padding: 16px 24px
        Title(text="Dashboard", style=headline-medium)
        ActionButton(color="primary", size="md", label="New Mock")

      content:
        LAYOUT: grid, columns: [1fr 1fr 1fr], gap: 16px, padding: 24px
        RESPONSIVE:
          < 768px: columns: [1fr]

        sessions-card: Card(variant="outlined")
          CardHeader(icon="science", title="Active Sessions")
          CardBody: Text({active_session_count} + " running")
          CardFooter(info={session_summary})

        components-card: Card(variant="elevated")
          CardHeader(icon="hub", title="Components")
          CardBody: Text({spec_count} + " specs")

        build-card: Card(variant="tinted-primary")
          CardHeader(icon="build", title="Last Build")
          CardBody: Text({last_build_status})
          CardFooter: ActionButton(size="sm", label="View")

  DATA_BINDINGS:
    active_mock_count    ← GET /api/v1/status → active_sessions.count
    active_session_count ← GET /api/v1/status → active_sessions.count
    session_summary      ← GET /api/v1/status → active_sessions.summary
    spec_count           ← GET /api/v1/status → specs.count
    last_build_status    ← GET /api/v1/build/latest → status

  Z_LAYERS:
    base (0): sidebar, main
    toast (200): notification toasts

  OVERLAYS:
    nav-drawer:
      TRIGGER: hamburger-icon.onClick (visible only < 1024px)
      TYPE: drawer
      BACKDROP: rgba(0,0,0,0.5)
      CONTENT: Sidebar(width="wide") -- same sidebar content
      DISMISS: backdrop.onClick OR swipe-left OR SidebarItem.onClick
```

### UXSpec.3 Transitions

Transitions define navigation between scenes. Each transition specifies a trigger,
animation, and navigation mode. This is the equivalent of UINavigationController
push/present (iOS), NavController navigate (Android), or React Router route changes.

```
TRANSITION {SourceScene} → {TargetScene}:
  TRIGGER:      {component}.{event} [WHERE {guard_condition}]
  ANIMATION:    {animation_type}, duration: {ms}, easing: {easing}
  NAVIGATION:   {push | replace | modal | drawer}
  PARAMS:       {data passed to target scene — e.g., session_id, namespace_id}
  BACK:         {how to return — "pop" | "dismiss" | "none"}
```

Animation types:
- `push-right`: slide in from right (iOS push equivalent)
- `push-left`: slide in from left (back navigation)
- `modal-slide-up`: slide up from bottom
- `modal-fade`: fade in with scale
- `fade-cross`: crossfade (content swap within persistent shell)
- `drawer-slide`: slide from left/right edge
- `none`: instant swap

Navigation modes:
- `push`: adds to navigation stack (back button appears)
- `replace`: replaces current scene (no back, used for content swap within shell)
- `modal`: overlays current scene (dismiss returns to previous)
- `drawer`: slides in from edge (dismiss returns to previous)

Example:

```
TRANSITION Dashboard → MockCreator:
  TRIGGER:      ActionButton("New Mock").onClick
  ANIMATION:    push-right, duration: 300ms, easing: standard
  NAVIGATION:   push
  PARAMS:       none
  BACK:         pop (push-left animation)

TRANSITION MockInteraction → CacheReview:
  TRIGGER:      end_session.response WHERE cache_entries_count > 0
  ANIMATION:    modal-slide-up, duration: 300ms, easing: standard
  NAVIGATION:   modal
  PARAMS:       { session_id }
  BACK:         none (must complete review before dismissing)

TRANSITION Dashboard → BuildMonitor:
  TRIGGER:      SidebarItem("Build").onClick
  ANIMATION:    fade-cross, duration: 200ms, easing: standard
  NAVIGATION:   replace
  PARAMS:       { namespace_id }
  BACK:         replace (fade-cross back to Dashboard)
```

### UXSpec.4 View Hierarchy

The default view hierarchy for a scene is flat — all components at z-index 0.
Deviations are declared in Z_LAYERS (within SCENE) and OVERLAYS. The view hierarchy
determines paint order, event propagation, and keyboard focus trapping.

Rules:
- **base (z: 0)**: main scene content, receives events normally
- **sticky (z: 10-49)**: pinned headers, toolbars — scroll beneath but above base
- **dropdown (z: 50-99)**: dropdowns, popovers — dismiss on outside click
- **overlay (z: 100-149)**: modals, drawers — trap focus, backdrop blocks base
- **toast (z: 200+)**: notifications — no focus trap, auto-dismiss
- Higher z-index layers capture events before lower layers
- Modal overlays trap keyboard focus (Tab cycles within overlay only)
- Multiple overlays stack (nested modals) — each pushes to overlay stack

### UXSpec.5 Visual Verification

Visual verification runs after a frontend BuildUnit is built. It validates that the
rendered output matches the SCENE spec. Three verification layers, run in order:

**Layer A — Structural Verification (deterministic, no AI):**
```
VISUAL_CHECK structural:
  FOR EACH scene IN spec.scenes:
    1. Render scene in headless browser at default viewport
    2. Query DOM for each COMPONENT declared in scene:
       - ASSERT component exists in DOM
       - ASSERT component has correct variant class/attribute
       - ASSERT component props match spec (color, size, label text)
    3. Query layout:
       - ASSERT layout type matches (grid columns count, flex direction)
       - ASSERT responsive: render at each declared breakpoint,
         verify layout changes (columns collapse, elements hide/show)
    4. Query z-index:
       - ASSERT z-index values match Z_LAYERS declaration
       - Trigger each OVERLAY, verify it appears at correct z-layer
    5. Query data bindings:
       - ASSERT bound data renders in correct components
       - Mock API returns test data, verify it propagates to UI
```

**Layer B — Token Compliance (deterministic, no AI):**
```
VISUAL_CHECK tokens:
  FOR EACH component IN rendered_scene:
    1. Get computed styles via getComputedStyle()
    2. Compare against design system tokens:
       - Color values match --md-* CSS variables for active theme
       - Font size/weight/family match type scale tokens
       - Padding/margin/gap match spacing tokens
       - Border radius matches shape tokens
       - Box shadow matches elevation tokens
    3. Report drift:
       - PASS: all values match within tolerance (1px for spacing, exact for colors)
       - FAIL: list each mismatch with expected vs actual
```

**Layer C — Visual Regression (AI-assisted):**
```
VISUAL_CHECK regression:
  FOR EACH scene IN spec.scenes:
    FOR EACH breakpoint IN scene.responsive:
      1. Render scene at breakpoint in headless browser
      2. Take screenshot (PNG, full page)
      3. Send to AI vision with SCENE spec as context
      4. AI checks:
         - Component spatial arrangement matches LAYOUT declaration
         - Visual hierarchy (prominent elements, whitespace balance)
         - Overlay rendering (backdrop opacity, content centering)
         - Responsive adaptation (columns collapse correctly, hidden elements gone)
      5. AI returns:
         - PASS/FAIL per check
         - Annotated screenshot with flagged areas
         - Confidence score per check
      6. FAIL threshold: any check below 0.9 confidence
```

### UXSpec.6 UX Test Scenarios

UX scenarios validate scene rendering and transition flows. They use the same
SCENARIO format as functional tests but add VISUAL and NAVIGATE assertions.

**Scene test** (validates a single scene renders correctly):
```
SCENARIO UX-{N}: {scene renders correctly} [SEC:UXSpec.2]
  GIVEN:
  - {backend state — mock data}
  - {user role/permissions}
  WHEN:
  - {scene} is rendered at {viewport_width}px
  THEN:
  - {component assertions — what's visible, what's not}
  - {data assertions — correct values displayed}
  VISUAL:
  - structural: {specific DOM checks}
  - tokens: {specific token compliance checks}
  - regression: {viewport}px screenshot passes AI check
```

**Flow test** (validates transitions between scenes):
```
SCENARIO UX-{N}: {user journey name} [SEC:UXSpec.3]
  GIVEN:
  - {starting scene and state}
  WHEN:
  - User {interaction — clicks, types, submits}
  THEN:
  - NAVIGATE: {transition_name} fires
  - ANIMATION: {animation_type} plays for {duration}ms
  - SCENE: {target_scene} is now active
  - {assertions about target scene state}
  WHEN:
  - User {next interaction}
  THEN:
  - ...
```

**Interaction test** (validates component behavior within a scene):
```
SCENARIO UX-{N}: {interaction name} [SEC:UXSpec.2]
  GIVEN:
  - {scene} is rendered
  WHEN:
  - User hovers {component}
  THEN:
  - {component} shows hover state (8% opacity overlay)
  WHEN:
  - User clicks {component}
  THEN:
  - {component} shows pressed state
  - {side effect — API call, state change, overlay appears}
```

**Overlay test** (validates overlay lifecycle):
```
SCENARIO UX-{N}: {overlay name} lifecycle [SEC:UXSpec.4]
  GIVEN:
  - {scene} is rendered
  WHEN:
  - {trigger event}
  THEN:
  - Overlay {name} appears at z-index {z}
  - Backdrop is visible (if declared)
  - Focus is trapped within overlay
  WHEN:
  - {dismiss event}
  THEN:
  - Overlay dismissed
  - Focus returns to trigger element
  - Base scene is interactive again
```

UX scenarios execute in a headless browser connected to a mock backend (l2mify's
own mock mode). The test runner:
1. Renders the scene using the built frontend artifacts
2. Connects to l2mify mock backend (REPLAY tier for fast, deterministic responses)
3. Drives interactions via Playwright/Puppeteer (click, type, wait)
4. Asserts DOM state after each action
5. Takes screenshots at assertion points for visual verification
6. Reports pass/fail per scenario with annotated screenshots on failure

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

## AgentPipeline

**Step-by-step execution protocol for an AI agent to autonomously provision, build,
deploy, test, and validate this system.** This section is optional — skip it for specs
where the Deployment and BuildAndRun sections provide sufficient instruction. Include
it when the agent needs a structured pipeline with explicit phases, acceptance criteria,
and rollback procedures.

The AgentPipeline does NOT duplicate Deployment or BuildAndRun content. It references
those sections and adds orchestration: the order of operations, what to verify between
steps, when to abort, and how to report results.

### When to Include This Section

Include AgentPipeline when:
- The system has complex deployment dependencies (multiple services, databases, platform
  services that must start in order)
- The agent needs to provision infrastructure before building
- There are acceptance criteria beyond "tests pass" (e.g., kernel activation, health
  endpoint verification, compliance checks)
- The system deploys on a platform that has its own setup requirements

Skip AgentPipeline when:
- `cargo build && cargo test && cargo run` is sufficient
- The system is a library with no deployment
- Deployment is a single `docker run` or `kubectl apply`

### Pipeline Structure

The pipeline is a sequence of phases. Each phase has actions, verification steps, and
abort conditions. The agent executes phases in order, verifying each before proceeding.

```
AGENT PIPELINE:

  PHASE 1: PROVISION
    description : {what infrastructure to set up}
    actions:
      1. {action — e.g., "Deploy PostgreSQL per Deployment.3"}
      2. {action}
    verify:
      - {check — e.g., "pg_isready returns success"}
      - {check}
    abort_if    : {condition — e.g., "database unreachable after 3 retries"}

  PHASE 2: BUILD
    description : {what to build}
    actions:
      1. {action — e.g., "Execute BuildAndRun.Build commands"}
      2. {action}
    verify:
      - {check — e.g., "All ARTIFACTs from BuildAndRun exist and pass checkable"}
    abort_if    : {condition}

  PHASE 3: DEPLOY
    description : {how to deploy}
    actions:
      1. {action — e.g., "Follow Deployment.4 deploy order"}
      2. {action}
    verify:
      - {check — e.g., "Health endpoint returns 200 with expected JSON"}
      - {check — e.g., "All N kernels report ACTIVE status"}
    abort_if    : {condition}

  PHASE 4: TEST
    description : {what to test}
    actions:
      1. {action — e.g., "Run SMOKE scenarios from Scenarios section"}
      2. {action — e.g., "Run FULL scenario suite"}
    verify:
      - {check — e.g., "All scenarios pass"}
      - {check — e.g., "No BLOCKING constraint violations in logs"}
    abort_if    : {condition — e.g., "Any SMOKE scenario fails"}

  PHASE 5: VALIDATE
    description : {final acceptance criteria}
    actions:
      1. {action — e.g., "Verify audit trail completeness"}
      2. {action — e.g., "Verify compliance markers (FERPA, COPPA, GDPR)"}
    verify:
      - {check}
    abort_if    : {condition}

  PHASE 6: REPORT
    description : {what to output}
    actions:
      1. {action — e.g., "Generate deployment report with artifact checksums"}
      2. {action — e.g., "Output kernel status summary"}
    output      : {what the agent produces — e.g., "JSON report at deploy/report.json"}
```

### Reference to Agent Instructions

For systems that use a shared agent instruction file (e.g., `CLAUDE.md`), this section
can be a brief reference:

```
> The AI agent execution pipeline is defined in `{path-to-agent-instructions}`.
> An agent reads this spec, provisions infrastructure per the SystemManifest,
> executes Deployment and BuildAndRun commands, and validates against Scenarios.
```

### Rules

- Each PHASE must have at least one `verify` check. Phases without verification are
  not phases — they're just commands.
- The agent must not proceed to the next phase until all `verify` checks pass.
- `abort_if` is mandatory for phases that can fail in ways that waste downstream effort
  (e.g., don't build if infrastructure is missing).
- The pipeline must be idempotent: running it twice should produce the same result
  (or detect that it's already complete and skip).

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

**Variant — EXPECTS ONE OF:** When a spec requires one of several alternatives (not a
specific dependency), use the `EXPECTS ONE OF` form. This declares that the spec needs
at least one provider from a set of options, with an optional default preference.

```
EXPECTS ONE OF {ExpectName} FROM {spec_id or "ANY_DEPENDENCY"}:
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  description : {what this spec needs, listing the alternatives and their trade-offs}
  required    : true | false
  default     : {recommended option if no explicit selection is made}
  fallback    : {behavior if required=false and no option is available}
```

Use `EXPECTS ONE OF` when:
- Multiple implementations satisfy the same contract (e.g., transport layers, storage backends)
- The spec is deliberately transport/implementation-agnostic
- The deployer should choose based on their environment

The seed resolver treats `EXPECTS ONE OF` as satisfied when **at least one** of the listed
options is available. If none are available and `required: true`, the build fails.

### Contracts.3 Conflict Resolution

When multiple dependencies export constraints that affect the same target:

1. `override=NEVER` exports always win. If two NEVER exports conflict, the build fails.
2. More restrictive constraint wins (lower limit, stricter policy).
3. Spec closer to the dependency root (fewer hops) wins.
4. If still ambiguous, the build fails with a clear error.

### Contracts.4 Layer-Aware Contracts

**When this spec declares a Layer (header), EXPORTS and EXPECTS gain layer semantics.**
Omit this subsection entirely for specs without a Layer declaration.

Layer-aware exports declare **which layer** the constraint targets and **which direction**
it flows:

```
EXPORT {ExportName}:
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  target      : {component/layer in the consumer}
  layer_target: {L2 | L3 | L4 | "derived" | "lateral" — which layer receives this export}
  direction   : DOWNWARD | UPWARD | LATERAL
  condition   : {when this export applies}
  value       : {the constraint or rule}
  override    : NEVER | WITH_JUSTIFICATION | UNRESTRICTED
  source_ref  : [SEC:SectionName.x]
```

**DOWNWARD exports** (default) flow from a spec to specs that derive from it:
```
EXPORT SearchInterface:
  type        : INVARIANT
  target      : search subsystem
  layer_target: derived
  direction   : DOWNWARD
  condition   : ALWAYS
  value       : "All search implementations must return results within 50ms p99"
  override    : NEVER
  source_ref  : [SEC:Functions.3]
```

**UPWARD exports** propagate preferences or constraints from a lower layer to the
layer it derives from. These represent the "constraint flow inversion" — the user or
configuration defining ground truth that the realization must accommodate:
```
EXPORT ThoroughnessPreference:
  type        : CONSTRAINT
  target      : processing pipeline
  layer_target: L3
  direction   : UPWARD
  condition   : "when user selects thoroughness-first mode"
  value       : "enable async deep-scan pipeline; disable streaming partial results"
  override    : WITH_JUSTIFICATION
  source_ref  : [SEC:Config.2]
```

**LATERAL exports** flow between horizontally composed specs at the same layer.
These are the interface contracts that composing peers depend on:
```
EXPORT BlockDeviceInterface:
  type        : INVARIANT
  target      : filesystem subsystem
  layer_target: lateral
  direction   : LATERAL
  condition   : ALWAYS
  value       : "block read/write via struct bio; minimum 512-byte sector alignment"
  override    : NEVER
  source_ref  : [SEC:API.1]
```

**Layer-aware EXPECTS** declare what a spec needs from a specific layer:
```
EXPECTS {ExpectName}:
  from        : {spec_id}
  from_layer  : {L1 | L2 | L3 | L4 — which layer provides this}
  type        : CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  description : {what this spec needs}
  required    : true | false
  fallback    : {default if not provided}
```

**Derivation inheritance rule:** When a spec at Layer N+1 derives from a spec at Layer N,
it automatically inherits all of the parent's DOWNWARD EXPORTS as constraints it must
satisfy. These inherited constraints do not need to be re-declared as EXPECTS — they
are implicit. The agent validates them during implementation.

**Upward propagation rule:** When a spec at Layer N+1 declares an UPWARD export, the
agent traces the derivation chain to Layer N and checks whether the parent can accommodate
the constraint. If the parent's relevant EXPORT is `override: NEVER`, propagation stops
with a conflict. If `WITH_JUSTIFICATION`, the agent flags it for human review. If
`UNRESTRICTED`, the constraint is applied automatically.

### Contracts.5 Notes for Spec Authors

- Every spec that other specs depend on SHOULD define its EXPORTS explicitly.
- If your spec has no exports, state: `EXPORTS: none — this spec is a leaf.`
- The seed resolver walks the full dependency graph and collects all exports before
  any code is generated. Conflicts are caught at build time, not runtime.
- For layered specs: EXPORTS with `layer_target: derived` apply to ALL specs that
  derive from this one, regardless of which specific layer they're at. Use a specific
  layer target (`L2`, `L3`, `L4`) when the export applies only at that layer.
- Upward exports are powerful but should be used sparingly. Most constraints flow
  downward. Reserve upward exports for genuinely non-negotiable user or domain
  requirements that the system must accommodate structurally.

---

## LayerContext

**Where this spec sits in a layered composition and how it relates to specs at other
layers.** This section is optional — omit it entirely for specs that are not part of a
multi-layer stack. Include it when the header declares a `Layer` value.

The 4-layer composition model lets one specification produce many realizations, each
realization support many configurations, and each configuration serve many user profiles.
Each layer is a **clean substitution boundary** — you swap at the level you care about
without disturbing the layers above or below.

Composition happens in two dimensions. **Vertical** composition is derivation across
layers (L1→L2→L3→L4). **Horizontal** composition is multiple specs at the same layer
composing together to form the complete layer — an OS is not one spec, it's kernel +
filesystem + networking + drivers, all at L2, with defined interfaces between them.

This section declares four things: what this spec derives from, what it composes with
at the same layer, what the full stack looks like, and how constraints flow.

### LayerContext.1 Derivation

A spec at Layer 2+ **derives from** a spec at the layer below it. Derivation means:
the derived spec inherits all contracts, invariants, and interface definitions from
its parent, then adds layer-specific concerns. A derived spec may constrain its parent
further but may not violate any of the parent's EXPORT constraints marked `override: NEVER`.

```
DERIVES FROM {parent-spec-id}:
  layer       : {parent's layer number}
  spec        : {parent spec filename or namespace/spec-id}
  inherits    : {what carries forward — "all EXPORTS", "Contracts + DataModel", or specific list}
  adds        : {one-line summary of what this layer contributes beyond the parent}
  constrained : {which parent EXPORTS this spec further narrows, or "none"}
```

**Layer 1 (Specification)** specs do not derive from anything — they are the root.
Declare: `DERIVES FROM: none — this is a root specification.`

**Layer 2 (Realization)** specs derive from an L1 spec:
```
DERIVES FROM product-spec:
  layer       : 1
  spec        : product-spec.md
  inherits    : all EXPORTS (contracts, interfaces, invariants)
  adds        : platform-specific implementation for React + Rails + Postgres
  constrained : none — faithful realization of the full L1 contract
```

**Layer 3 (Configuration)** specs derive from an L2 spec:
```
DERIVES FROM product-react-realization:
  layer       : 2
  spec        : product-react-spec.md
  inherits    : all EXPORTS + DataModel + API surface
  adds        : healthcare domain configuration (HIPAA mode, audit requirements)
  constrained : DataRetention (narrowed from 90d to 7y per HIPAA)
```

**Layer 4 (User Profile)** specs derive from an L3 spec:
```
DERIVES FROM product-healthcare-config:
  layer       : 3
  spec        : product-healthcare-config.md
  inherits    : all EXPORTS + Config surface
  adds        : radiologist workflow preferences (thoroughness-first, image-heavy layout)
  constrained : UILayout (overrides default grid with radiology-optimized panels)
```

#### Horizontal Composition — COMPOSES WITH

A single layer often requires multiple specs that compose together horizontally.
Each spec in the composition owns a distinct concern but they share a layer, derive
from the same parent(s), and define interfaces between each other. Horizontal
composition uses the existing IMPORT/EXPORT mechanism — composing specs at the same
layer are peers that export contracts to and import from each other.

```
COMPOSES WITH:
  {peer-spec-id}:
    spec        : {peer spec filename or namespace/spec-id}
    interface   : {what this spec exposes to the peer — e.g., "syscall ABI", "event bus API"}
    depends_on  : {what this spec needs from the peer — e.g., "block device interface", "none"}
    relationship: {co-required | optional | alternative}
```

**Relationship types:**
- `co-required` — both specs must be present for the layer to be complete. Neither
  works without the other (e.g., kernel + filesystem in an OS).
- `optional` — the peer adds capability but the layer functions without it (e.g.,
  GPU driver in an OS that can run headless).
- `alternative` — the peer is an alternative implementation of the same concern.
  Only one of the alternatives is used at a time (e.g., ext4-spec vs btrfs-spec
  for the filesystem concern).

Example — an OS realization composed of multiple specs:
```
DERIVES FROM os-spec:
  layer       : 1
  spec        : os-spec.md
  inherits    : all EXPORTS (POSIX interface contracts, security invariants)
  adds        : Linux kernel implementation for arm64
  constrained : none

COMPOSES WITH:
  filesystem-ext4:
    spec        : filesystem-ext4-spec.md
    interface   : VFS layer (struct file_operations, struct inode_operations)
    depends_on  : block device interface from this spec
    relationship: co-required

  networking-stack:
    spec        : networking-spec.md
    interface   : socket API (AF_INET, AF_UNIX)
    depends_on  : memory management, interrupt handling from this spec
    relationship: co-required

  gpu-driver:
    spec        : gpu-driver-spec.md
    interface   : DRM/KMS interface
    depends_on  : PCI subsystem, memory management from this spec
    relationship: optional
```

**Rules for horizontal composition:**
- All specs in a horizontal composition MUST share the same Layer number and the
  same DERIVES FROM parent (or set of parents).
- Interface contracts between composing specs use the standard IMPORT/EXPORT mechanism.
  There is no special "lateral export" — a composing spec exports to its peers the
  same way it exports to any consumer.
- The seed resolver treats horizontally composed specs as a single resolution unit
  at their layer. All must be resolved together — a conflict between composing specs
  fails the build for the entire layer.
- When validating, the agent must check that every `depends_on` is satisfied by an
  EXPORT from the referenced peer spec. Missing dependencies are SPEC_GAPs.
- `alternative` specs are mutually exclusive. The seed resolver selects one based on
  configuration or fails if the selection is ambiguous.

### LayerContext.2 Layer Stack

**The full composition stack this spec participates in.** This gives the agent context
about the entire derivation chain — from the root specification down to the user profile.
Not every layer in the stack needs to exist yet. Mark unwritten layers as `(planned)`.

```
LAYER STACK:

  L1  {spec-id}                     — {one-line purpose}
   │
   └─ L2  {spec-id}                 — {one-line purpose}
      ├── L2  {spec-id}             — {horizontal peer, co-required}
      ├── L2  {spec-id}             — {horizontal peer, optional}
       │
       ├─ L3  {spec-id}             — {one-line purpose}
       │   │
       │   ├─ L4  {spec-id}         — {one-line purpose}
       │   └─ L4  {spec-id}         — {one-line purpose}
       │
       └─ L3  {spec-id}             — {one-line purpose}
           │
           └─ L4  {spec-id}         — {one-line purpose}
```

The stack has two dimensions. **Vertically**, one L1 may have many L2 realizations,
each L2 may have many L3 configurations, each L3 may have many L4 profiles.
**Horizontally**, each layer may consist of multiple composing specs shown as siblings
connected by `├──` at the same indentation level. Mark the current spec with
`← THIS SPEC`. Mark horizontal peers with their relationship type.

Example — vertical-only (simple product):
```
LAYER STACK:

  L1  product-spec                   — core product contracts and interfaces
   │
   ├─ L2  product-react-spec         — React + Rails + Postgres realization
   │   │
   │   ├─ L3  product-healthcare     — HIPAA-compliant healthcare configuration  ← THIS SPEC
   │   │   │
   │   │   ├─ L4  radiologist-profile — radiologist workflow preferences
   │   │   └─ L4  nurse-profile       — nursing station preferences
   │   │
   │   └─ L3  product-education      — FERPA-compliant education configuration
   │       │
   │       └─ L4  teacher-profile     — teacher dashboard preferences
   │
   └─ L2  product-ios-spec           — native iOS realization (planned)
```

Example — horizontal + vertical (OS with composed subsystems):
```
LAYER STACK:

  L1  os-spec                        — POSIX interface contracts, security invariants
   │
   └─ L2  kernel-spec                — Linux kernel for arm64  ← THIS SPEC
      ├── L2  filesystem-ext4-spec   — ext4 filesystem (co-required)
      ├── L2  networking-spec        — TCP/IP networking stack (co-required)
      ├── L2  gpu-driver-spec        — GPU/DRM subsystem (optional)
       │
       ├─ L3  server-config          — headless server configuration
       │   │
       │   └─ L4  db-server-profile  — database server tuning
       │
       └─ L3  desktop-config         — desktop workstation configuration
           │
           └─ L4  developer-profile  — developer workstation preferences
```

**Rules:**
- A spec MUST list at least its own layer, its parent (if any), and its horizontal
  peers (if any, via COMPOSES WITH).
- Sibling and child layers are recommended but not required.
- `(planned)` marks layers that are anticipated but have no spec file yet.
- Horizontal peers are shown at the same indentation level with `├──` and annotated
  with their relationship type (co-required, optional, alternative).
- The agent uses the stack to understand substitution scope: changing an L3 spec
  affects only the L4 specs beneath it, not sibling L3 specs or the L2 above.
  Changing a horizontally composed spec affects its co-required peers (they may
  need re-validation) but not optional or alternative peers.

### LayerContext.3 Constraint Flow

Constraints flow in three directions. **Downward** is the default: each layer constrains
the layer below it. **Upward** is the inversion described in the composition model:
a user's preferences propagate up through configuration into realization and
potentially into the specification itself. **Lateral** is the flow between horizontally
composed specs at the same layer: peers constrain each other through shared interfaces.

```
CONSTRAINT FLOW:

  DOWNWARD (default — higher layers constrain lower layers):
    L1 → L2: {what L1 constraints carry into the realization}
    L2 → L3: {what L2 constraints carry into the configuration}
    L3 → L4: {what L3 constraints carry into the user profile}

  UPWARD (inversion — lower layers propagate preferences upward):
    L4 → L3: {what user preferences the configuration must accommodate}
    L3 → L2: {what configuration needs require realization changes}
    L2 → L1: {what realization discoveries require spec changes}

  LATERAL (same-layer — horizontal peers constrain each other):
    {spec-a} ↔ {spec-b}: {what constraints flow between composing specs}
```

Not every spec needs all three directions. Many systems use only downward flow.
Upward flow matters when user preferences are non-negotiable. Lateral flow matters
when multiple specs compose horizontally at the same layer.

```
CONSTRAINT FLOW:

  DOWNWARD:
    L1 → L2: all interface contracts, invariants, data model shapes
    L2 → L3: API surface, deployment constraints, available feature flags
    L3 → L4: configurable preference dimensions, allowed override ranges

  UPWARD:
    L4 → L3: "thoroughness over speed" preference requires async processing mode
    L3 → L2: async processing mode requires WebSocket realization (not REST-only)
    L2 → L1: (none — L1 already accommodates both sync and async interfaces)

  LATERAL:
    kernel ↔ filesystem: kernel provides block device interface, filesystem provides VFS
    kernel ↔ networking: kernel provides memory management, networking provides socket API
```

**Rules:**
- Downward constraints are mandatory. Every derived spec inherits and must satisfy
  its parent's EXPORTS.
- Upward constraints are advisory until validated. An L4 preference that would require
  an L1 change is flagged as a SPEC_GAP, not silently accommodated.
- When upward flow reaches a constraint marked `override: NEVER`, propagation stops.
  The agent reports: "User preference X conflicts with L{n} constraint Y (NEVER override).
  The preference cannot be accommodated without a spec change at L{n}."
- Lateral constraints between composing specs use the standard IMPORT/EXPORT mechanism.
  When spec-a declares `depends_on` from spec-b in COMPOSES WITH, spec-b must have a
  matching EXPORT. The seed resolver validates lateral dependencies as part of the
  layer's resolution unit.
- AI validates upward flow by mocking the affected layers: change the L3 config per the
  L4 preference, validate L3 against L2, validate L2 against L1. If all pass, the
  preference is safe. If any fail, report the conflict.

### LayerContext.4 Substitution Boundaries

**What can be swapped at this layer without affecting other layers.** This is the core
property that makes the composition model practical — each layer is an independent
substitution unit.

```
SUBSTITUTION BOUNDARY:
  swappable   : {what can be replaced at this layer — e.g., "entire realization",
                 "configuration profile", "user preference set"}
  stable_above: {what the layer above sees that does NOT change when this layer is swapped}
  stable_below: {what the layer below provides that this layer depends on remaining stable}
  swap_cost   : {low | medium | high — how much effort to swap this layer}
  swap_trigger: {when you'd swap — e.g., "new platform", "new domain", "new user type"}
```

Example for an L2 (Realization) spec:
```
SUBSTITUTION BOUNDARY:
  swappable   : entire technology stack (React→Vue, Rails→Express, Postgres→DynamoDB)
  stable_above: L1 contracts, interfaces, invariants — unchanged regardless of technology
  stable_below: (none — L2 is realized against L1, no layer below)
  swap_cost   : high — full re-materialization, but validated against same L1 spec
  swap_trigger: platform migration, performance requirements, team expertise change
```

Example for an L4 (User Profile) spec:
```
SUBSTITUTION BOUNDARY:
  swappable   : all user preferences and layout overrides
  stable_above: L3 configuration, feature flags, domain constraints — unchanged per user
  stable_below: (none — L4 is the leaf layer)
  swap_cost   : low — runtime preference application, no re-materialization
  swap_trigger: new user, user preference change, accessibility requirement
```

### LayerContext.5 Cross-Layer References

When a spec needs to reference a specific element in a spec at another layer, use the
cross-layer reference syntax:

```
[Ln:{spec-id}:{Section}.{subsection}]
```

Examples:
```
This CONFIG parameter maps to [L1:product-spec:Functions.3] — the search interface
defined in the root specification.

This SCENARIO validates behavior required by [L2:product-react-spec:API.2] — the
REST endpoint that this configuration tunes.

The DataRetention override in this profile satisfies [L3:product-healthcare:Config.2]
— the HIPAA retention policy declared in the healthcare configuration.
```

**Rules:**
- Cross-layer references are unidirectional markers, not imports. They say "this
  element relates to that element at layer N" for traceability.
- For actual dependency on types or functions from another layer, use the existing
  IMPORT syntax (Appendix B) with the layer-qualified spec-id.
- The agent follows cross-layer references during validation: when a referenced
  element changes, affected scenarios at the referencing layer should be re-run.
- Cross-layer references to `(planned)` specs are allowed — the agent notes them
  as unresolvable until the referenced spec exists.

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
