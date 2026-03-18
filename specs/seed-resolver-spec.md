# Seed Resolver Specification

**Spec ID:** seed-resolver  
**Version:** 0.1.0  
**Type:** SYSTEM  
**Status:** DRAFT  
**IMPORTS:** nlspec/bootstrap-spec, nlspec/mcp-server-spec  

---

## Abstract

```
The Seed Resolver is a build-time tool that walks a project's spec dependency graph,
collects all EXPORT declarations from Contracts section (Dependency Contracts) of each spec,
validates EXPECTS contracts are satisfied, detects conflicts, and produces a deterministic
seed manifest. The seed manifest contains the complete set of rules, constraints,
invariants, policies, and initial data that the system must be initialized with before
any runtime behavior begins. The resolver guarantees that the initial state of a system
is a deterministic function of its spec graph — no manual configuration, no hidden
defaults, no runtime surprises.
```

---

## Problem Statement

```
CURRENT STATE:
  Systems built from multiple specs require manual configuration to wire dependencies
  together. A spec may declare that it depends on another spec, but the implications
  of that dependency — what constraints propagate, what initial state is required,
  what invariants must hold — are left to the implementer to discover and enforce.

DEFICIENCY:
  This manual wiring is error-prone, inconsistent, and invisible to auditing. Two
  developers building from the same spec graph may produce systems with different
  initial states. Safety-critical constraints defined in foundational specs may be
  missed or incorrectly applied by consuming specs. There is no build-time guarantee
  that dependency contracts are honored.

TARGET STATE:
  A build-time resolver that deterministically produces the complete initial state
  from the spec graph. Running the resolver on the same graph always produces the
  same seed manifest. All dependency contracts are validated. All conflicts are
  detected and reported. The system boots into a fully constrained, auditable state.

KEY INSIGHT:
  If specs define their exports and expectations formally (Contracts section), then the
  initial state of any system is a computable function of its dependency graph.
  The resolver is that function.
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Spec Graph (on disk)                  │
│                                                         │
│  spec-a.md ──IMPORTS──► spec-b.md ──IMPORTS──► spec-c.md│
│      │                      │                     │     │
│   [SEC:Contracts]               [SEC:Contracts]              [SEC:Contracts] │
│   EXPORTS: ...           EXPORTS: ...          EXPORTS: │
│   EXPECTS: ...           EXPECTS: ...          (leaf)   │
└────────────────────────────┬────────────────────────────┘
                             │
                      ┌──────▼──────┐
                      │   Resolver  │
                      │             │
                      │  1. Parse   │  ← Read Contracts section from each spec
                      │  2. Collect │  ← Gather all EXPORTS
                      │  3. Match   │  ← Validate EXPECTS against EXPORTS
                      │  4. Detect  │  ← Find conflicts
                      │  5. Order   │  ← Topological sort by precedence
                      │  6. Emit    │  ← Produce seed manifest
                      └──────┬──────┘
                             │
                      ┌──────▼──────┐
                      │    Seed     │
                      │  Manifest   │
                      │             │
                      │ constraints │  ← override=NEVER rules
                      │ invariants  │  ← runtime-verified properties
                      │ policies    │  ← behavioral defaults
                      │ seed_data   │  ← initial state populations
                      │ provenance  │  ← which spec produced each rule
                      └─────────────┘

COMPONENTS:
  1. Graph Walker      — traverses the spec dependency graph via IMPORTS declarations
  2. Contract Parser   — extracts EXPORT and EXPECTS blocks from Contracts section
  3. Contract Matcher  — validates that every EXPECTS has a matching EXPORT
  4. Conflict Detector — identifies overlapping exports targeting the same component
  5. Conflict Resolver — applies precedence rules to resolve or reject conflicts
  6. Manifest Emitter  — produces the final seed manifest with full provenance

DATA FLOW:
  spec files → Graph Walker → ordered spec list
  ordered spec list → Contract Parser → raw exports + expects
  raw exports + expects → Contract Matcher → matched pairs + unmatched expects
  matched pairs → Conflict Detector → conflict report
  conflict report → Conflict Resolver → resolved set OR build failure
  resolved set → Manifest Emitter → seed-manifest.json
```

---

## DataModel

```
RECORD SeedExport:
  id              : String              -- unique identifier: "{spec_id}.{export_name}"
  spec_id         : String              -- which spec declares this export
  export_name     : String              -- name from the EXPORT block
  type            : ExportType          -- CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  export_context  : String              -- "runtime" (SYSTEM), "architectural" (PATTERN), "design" (ASSET)
  target          : String              -- which component in the consumer receives this
  condition       : String              -- when this applies ("ALWAYS" if unconditional)
  value           : String              -- the rule, constraint, or data
  override        : OverrideLevel       -- NEVER | WITH_JUSTIFICATION | UNRESTRICTED
  source_ref      : String              -- [SEC:x.x] in the originating spec
  layer_target    : String | None       -- which layer receives this export (L2, L3, L4, "derived", or None for non-layered)
  direction       : String = "DOWNWARD" -- DOWNWARD (default), UPWARD, or LATERAL
  graph_depth     : Integer             -- hops from the consuming spec (0 = direct dependency)

  USED BY: collect_exports, detect_conflicts, resolve_conflicts, emit_manifest

RECORD SeedExpect:
  id              : String              -- unique identifier: "{spec_id}.{expect_name}"
  spec_id         : String              -- which spec declares this expectation
  expect_name     : String              -- name from the EXPECTS block
  from            : String              -- spec_id or "ANY_DEPENDENCY"
  from_layer      : Integer | None      -- which layer provides this (1-4, or None for non-layered)
  type            : ExportType          -- expected type
  description     : String              -- what is needed and why
  required        : Boolean             -- true = build fails if unmatched
  fallback        : String | None       -- default if required=false and unmatched

  USED BY: match_contracts, emit_manifest

RECORD ConflictReport:
  target          : String              -- the component where conflict occurs
  exports         : List<SeedExport>    -- the conflicting exports
  resolution      : String | None       -- how it was resolved, or None if unresolvable
  resolved_winner : SeedExport | None   -- which export won, or None if build fails

  USED BY: resolve_conflicts, emit_manifest

RECORD SeedManifest:
  version         : String              -- manifest format version
  generated_at    : Timestamp           -- when the resolver ran
  spec_graph_hash : String              -- hash of all input specs (for cache invalidation)
  constraints     : List<SeedRule>      -- override=NEVER rules
  invariants      : List<SeedRule>      -- runtime-verified properties
  policies        : List<SeedRule>      -- behavioral defaults
  seed_data       : List<SeedRule>      -- initial state populations
  conflicts       : List<ConflictReport>-- resolved conflicts (for audit trail)
  unmatched       : List<SeedExpect>    -- optional expects with fallbacks applied
  layer_conflicts : List<ConflictReport> -- conflicts from upward constraint flow crossing NEVER boundaries

  USED BY: emit_manifest, validate_manifest

RECORD SeedRule:
  id              : String              -- unique rule identifier
  source_spec     : String              -- which spec produced this rule
  source_export   : String              -- which EXPORT block
  source_ref      : String              -- [SEC:x.x] in the originating spec
  type            : ExportType          -- CONSTRAINT | INVARIANT | SEED_DATA | POLICY
  target          : String              -- which component receives this
  value           : String              -- the actual rule content
  override        : OverrideLevel       -- preserved from the export
  graph_depth     : Integer             -- how far from the consumer in the dep graph
  condition       : String              -- when this rule applies

  USED BY: emit_manifest, validate_manifest, apply_seeds

ENUM ExportType:
  CONSTRAINT          -- a limit that restricts behavior
  INVARIANT           -- a property that must always hold at runtime
  SEED_DATA           -- initial state to populate at deploy time
  POLICY              -- a behavioral default

ENUM OverrideLevel:
  NEVER               -- cannot be overridden by any consumer, ever
  WITH_JUSTIFICATION  -- can be overridden if the consumer documents why
  UNRESTRICTED        -- consumer can freely override
```

---

## Functions

```
FUNCTION walk_graph:
  INPUT:  entry_spec_id: String -- the top-level spec to resolve from
  OUTPUT: List<String>          -- spec_ids in topological order (leaves first)

  BEHAVIOR:
  1. Read entry spec's IMPORTS declarations
  2. Recursively read each imported spec's IMPORTS
  3. Build directed graph of all spec dependencies
  4. Topological sort (leaves first — specs with no IMPORTS come first)
  5. If cycle detected, warn but do not fail. Process each spec in the cycle once.
  6. When traversing specs with Layer declarations, also follow DERIVES FROM links
     in addition to IMPORTS declarations. The derivation chain (L1→L2→L3→L4) is a
     separate axis from the dependency chain (IMPORTS). Both must be walked.
  7. Layer derivation links are walked parent-first (L1 before L2, L2 before L3)
  8. When traversing specs with COMPOSES WITH declarations, also add edges for
     horizontal peers. All specs in a horizontal composition are resolved as a
     single unit at their layer level.

  USES: none (reads spec files directly)
  THROWS: SpecNotFound, CycleDetected (warning only)

  NOTES:
  - Leaves-first ordering ensures that foundational specs' exports are collected
    before consumer specs' expects are validated.


FUNCTION collect_exports:
  INPUT:  spec_ids: List<String>, relative_to: String
  OUTPUT: List<SeedExport>

  BEHAVIOR:
  1. For each spec_id in the ordered list:
     a. Parse Contracts section (Dependency Contracts)
     b. Extract all EXPORT blocks
     c. Read the spec's Type header (SYSTEM, PATTERN, or ASSET)
     d. Tag each SeedExport with export_context:
        - SYSTEM exports → runtime constraints (enforced at boot/runtime)
        - PATTERN exports → architectural constraints (enforced at implementation time)
        - ASSET exports → design constraints (enforced when generating UI/config)
     e. Create SeedExport records with graph_depth calculated as the shortest
        path from relative_to to this spec in the dependency graph
  2. For layer-aware exports (layer_target is set): record the direction (DOWNWARD or UPWARD)
     - DOWNWARD exports flow from parent to derived specs in the layer hierarchy
     - UPWARD exports flow from derived specs back toward their parent layer
     - LATERAL exports (direction=LATERAL): these flow between horizontally composed
       peers at the same layer. They are resolved within the layer's composition unit.
     - Non-layered exports (layer_target=None) are treated as before (standard dependency flow)
  3. Return all collected exports

  USES: SeedExport
  THROWS: MalformedExport, MissingSection16


FUNCTION collect_expects:
  INPUT:  spec_ids: List<String>
  OUTPUT: List<SeedExpect>

  BEHAVIOR:
  1. For each spec_id in the ordered list:
     a. Parse Contracts section (Dependency Contracts)
     b. Extract all EXPECTS blocks
     c. Create SeedExpect records
  2. Return all collected expects

  USES: SeedExpect
  THROWS: MalformedExpect, MissingSection16


FUNCTION match_contracts:
  INPUT:  exports: List<SeedExport>, expects: List<SeedExpect>
  OUTPUT: matched: List<(SeedExpect, SeedExport)>,
          unmatched_required: List<SeedExpect>,
          unmatched_optional: List<SeedExpect>

  BEHAVIOR:
  1. For each EXPECTS:
     a. If from is a specific spec_id, look for matching EXPORT from that spec
        where type matches
     b. If from is "ANY_DEPENDENCY", look for any EXPORT where type matches
        and target is compatible
     c. If no match found and required=true, add to unmatched_required
     d. If no match found and required=false, add to unmatched_optional
  2. If unmatched_required is non-empty, build fails with clear error listing
     each unmatched expect, which spec declared it, and what it needs

  USES: SeedExport, SeedExpect
  THROWS: UnmatchedRequiredExpect


FUNCTION detect_conflicts:
  INPUT:  exports: List<SeedExport>
  OUTPUT: List<ConflictReport>

  BEHAVIOR:
  1. Group exports by target
  2. Within each group, identify overlapping exports:
     a. Same target AND overlapping conditions = potential conflict
     b. Same target AND one condition is "ALWAYS" AND another is conditional = no conflict
        (conditional narrows the general case)
     c. Same target AND both conditions are "ALWAYS" = definite conflict
  3. Create ConflictReport for each conflict

  USES: SeedExport, ConflictReport
  THROWS: none (conflicts are data, not errors — resolution decides if it's fatal)


FUNCTION resolve_conflicts:
  INPUT:  conflicts: List<ConflictReport>
  OUTPUT: resolved: List<ConflictReport>

  BEHAVIOR:
  For each conflict, apply rules in order:
  1. If any export has override=NEVER:
     a. If exactly one NEVER export → it wins
     b. If multiple NEVER exports conflict → BUILD FAILS (human must resolve)
  2. If no NEVER exports, apply "more restrictive wins":
     a. For numeric constraints: lower limit wins
     b. For boolean constraints: the more restrictive value wins (false > true for permissions)
     c. For policies: cannot auto-resolve → BUILD FAILS with suggestion
  3. If still tied, apply "closer to root wins" (lower graph_depth)
  4. If still tied → BUILD FAILS with detailed conflict report
  5. For UPWARD exports: trace the derivation chain to find the parent layer's constraint
     a. If the parent has an override=NEVER export on the same target, the upward export
        is a conflict. Add to layer_conflicts in the manifest.
     b. If the parent has override=WITH_JUSTIFICATION, flag for human review but do not fail
     c. If the parent has override=UNRESTRICTED, the upward export is applied automatically
  6. For LATERAL exports: verify that both sides of a lateral interface agree on the
     exported contract. If spec-a exports an interface that spec-b depends on but spec-b
     expects a different contract, this is a conflict. Add to layer_conflicts.

  USES: SeedExport, ConflictReport
  THROWS: UnresolvableConflict


FUNCTION emit_manifest:
  INPUT:  exports: List<SeedExport>,
          resolved_conflicts: List<ConflictReport>,
          unmatched_optional: List<SeedExpect>
  OUTPUT: SeedManifest

  BEHAVIOR:
  1. Convert winning exports to SeedRules, preserving full provenance
  2. Apply fallbacks for unmatched optional expects
  3. Group rules by export_context first (runtime, architectural, design),
     then by type within each group: constraints, invariants, policies, seed_data
  4. Sort within each group by graph_depth (deepest first — foundational rules listed first)
  5. Compute spec_graph_hash from all input spec file hashes
  6. Emit SeedManifest as JSON

  USES: SeedManifest, SeedRule, ConflictReport, SeedExpect
  THROWS: none


FUNCTION validate_manifest:
  INPUT:  manifest: SeedManifest
  OUTPUT: validation_result: Boolean, issues: List<String>

  BEHAVIOR:
  1. Verify no duplicate rule IDs
  2. Verify all source_spec references point to real specs
  3. Verify all source_ref [SEC:x.x] references exist in their source specs
  4. Verify no CONSTRAINT rules conflict with INVARIANT rules
  5. Verify all SEED_DATA rules reference valid targets
  6. Return validation result and any issues found

  USES: SeedManifest, SeedRule
  THROWS: InvalidManifest
```

---

## API Surface

```
CLI COMMANDS:

  nlspec-seed resolve <entry-spec> [--output <path>] [--dry-run] [--verbose]
    Walks the dependency graph from <entry-spec>, collects all contracts,
    resolves conflicts, and emits seed-manifest.json.
    --dry-run: validate contracts and report conflicts without emitting manifest
    --verbose: print each export collected and each conflict resolved

  nlspec-seed validate <manifest-path>
    Validates an existing seed manifest against the current spec graph.
    Reports stale rules (spec changed since manifest was generated).

  nlspec-seed diff <manifest-a> <manifest-b>
    Compares two seed manifests. Reports added, removed, and changed rules.
    Useful for reviewing what a spec graph change does to the initial state.

  nlspec-seed audit <manifest-path>
    For each rule in the manifest, prints the full provenance chain:
    rule → export → spec → section. Answers "why does this rule exist?"

MCP TOOLS:

  nlspec_seed_resolve:
    INPUT:  { entry_spec_id: String, dry_run: Boolean }
    OUTPUT: { manifest: SeedManifest } | { conflicts: List<ConflictReport> }
    Called by agents during build mode to produce or validate seed state.

  nlspec_seed_audit:
    INPUT:  { rule_id: String }
    OUTPUT: { provenance: { rule: SeedRule, export: SeedExport, spec_section: String } }
    Called by agents to understand why a specific seed rule exists.
```

---

## Errors Model

```
ERROR SpecNotFound:
  trigger   : walk_graph encounters an IMPORTS reference to a nonexistent spec
  message   : "Spec '{spec_id}' referenced in IMPORTS of '{parent_spec}' not found"
  severity  : FATAL
  handling  : Build fails. List all missing specs.

ERROR MissingSection16:
  trigger   : A spec in the graph has no Contracts section (Dependency Contracts)
  message   : "Spec '{spec_id}' has no Dependency Contracts section"
  severity  : WARNING
  handling  : Treat spec as having no exports and no expects. Continue resolution.

ERROR MalformedExport:
  trigger   : An EXPORT block is missing required fields
  message   : "EXPORT '{name}' in spec '{spec_id}' is missing field '{field}'"
  severity  : FATAL
  handling  : Build fails. List all malformed exports.

ERROR MalformedExpect:
  trigger   : An EXPECTS block is missing required fields
  message   : "EXPECTS '{name}' in spec '{spec_id}' is missing field '{field}'"
  severity  : FATAL
  handling  : Build fails. List all malformed expects.

ERROR UnmatchedRequiredExpect:
  trigger   : A required EXPECTS has no matching EXPORT in any dependency
  message   : "Spec '{spec_id}' expects '{expect_name}' from '{from}' but no
               matching export exists. Required type: {type}"
  severity  : FATAL
  handling  : Build fails. List all unmatched required expects with suggestions
              (closest matching exports by type/target).

ERROR UnresolvableConflict:
  trigger   : Two or more exports conflict and cannot be auto-resolved
  message   : "Conflict on target '{target}': exports from {spec_a} and {spec_b}
               both have override=NEVER with incompatible values"
  severity  : FATAL
  handling  : Build fails. Print both exports in full with their source specs
              and sections. Human must edit one spec to resolve.

ERROR InvalidManifest:
  trigger   : validate_manifest finds structural issues
  message   : "Seed manifest validation failed: {issues}"
  severity  : FATAL
  handling  : Build fails. List all issues.

ERROR StaleManifest:
  trigger   : validate detects spec_graph_hash mismatch
  message   : "Seed manifest was generated from a different spec graph. Re-run resolve."
  severity  : WARNING
  handling  : Warn but don't fail. Recommend re-running resolve.
```

---

## Configuration

```
OPTION resolve.strict_mode:
  type     : Boolean
  default  : true
  env      : NLSPEC_SEED_STRICT
  effect   : When true, MissingSection16 is FATAL instead of WARNING.
             Use in production. Disable during early development.

OPTION resolve.output_format:
  type     : Enum (json | yaml)
  default  : json
  env      : NLSPEC_SEED_FORMAT
  effect   : Format of the emitted seed manifest.

OPTION resolve.output_path:
  type     : String
  default  : "./seed-manifest.json"
  env      : NLSPEC_SEED_OUTPUT
  effect   : Where the seed manifest is written.

OPTION resolve.max_graph_depth:
  type     : Integer
  default  : 20
  env      : NLSPEC_SEED_MAX_DEPTH
  effect   : Maximum dependency depth to traverse. Fails if exceeded (likely a cycle).

OPTION resolve.conflict_strategy:
  type     : Enum (strict | permissive)
  default  : strict
  env      : NLSPEC_SEED_CONFLICT
  effect   : strict = any ambiguous conflict fails the build.
             permissive = apply heuristics and warn instead of failing.
```

---

## Deployment Artifacts

```
ARTIFACT nlspec-seed:
  type     : CLI binary
  language : TypeScript (Node.js)
  install  : npm install -g @nlspec/seed-resolver
  depends  : @nlspec/core (parser, graph walker from bootstrap spec)

PIPELINE seed-resolve:
  trigger  : Any .md file in specs/ directory changes
  steps    :
    1. nlspec-seed resolve specs/entry-spec.md --output build/seed-manifest.json
    2. nlspec-seed validate build/seed-manifest.json
    3. If step 1 or 2 fails, block deployment

FILE_TO_SECTION_MAP:
  src/walker.ts         → [SEC:Functions] walk_graph
  src/collector.ts      → [SEC:Functions] collect_exports, collect_expects
  src/matcher.ts        → [SEC:Functions] match_contracts
  src/conflict.ts       → [SEC:Functions] detect_conflicts, resolve_conflicts
  src/emitter.ts        → [SEC:Functions] emit_manifest
  src/validator.ts      → [SEC:Functions] validate_manifest
  src/cli.ts            → [SEC:API] CLI commands
  src/mcp.ts            → [SEC:API] MCP tools
```

---

## Scenarios

```
SCENARIO 1: Simple linear dependency [SMOKE] [SEC:Functions]
  GIVEN: spec-a IMPORTS spec-b, spec-b IMPORTS spec-c
  AND:   spec-c EXPORTS a CONSTRAINT with override=NEVER
  AND:   spec-b EXPORTS a POLICY with override=UNRESTRICTED
  AND:   spec-a EXPECTS the CONSTRAINT from spec-c (required=true)
  WHEN:  resolve is run with spec-a as entry
  THEN:  manifest contains both the CONSTRAINT and the POLICY
  AND:   CONSTRAINT has graph_depth=2, POLICY has graph_depth=1
  AND:   provenance traces back to correct specs and sections

SCENARIO 2: Conflict between two NEVER exports [SMOKE] [SEC:Functions]
  GIVEN: spec-a IMPORTS spec-b and spec-c
  AND:   spec-b EXPORTS "max_memory <= 16GB" with override=NEVER
  AND:   spec-c EXPORTS "max_memory <= 32GB" with override=NEVER
  WHEN:  resolve is run with spec-a as entry
  THEN:  build FAILS with UnresolvableConflict
  AND:   error message lists both exports with their source specs

SCENARIO 3: More restrictive wins [AFFECTED] [SEC:Functions]
  GIVEN: spec-a IMPORTS spec-b and spec-c
  AND:   spec-b EXPORTS "timeout <= 30s" with override=WITH_JUSTIFICATION
  AND:   spec-c EXPORTS "timeout <= 10s" with override=WITH_JUSTIFICATION
  WHEN:  resolve is run with spec-a as entry
  THEN:  manifest contains "timeout <= 10s" (more restrictive)
  AND:   conflict report shows resolution reasoning

SCENARIO 4: Unmatched required expect [SMOKE] [SEC:Functions]
  GIVEN: spec-a EXPECTS "HardwareLimits" from spec-b (required=true)
  AND:   spec-b has no Contracts section or no matching EXPORT
  WHEN:  resolve is run with spec-a as entry
  THEN:  build FAILS with UnmatchedRequiredExpect
  AND:   error suggests closest matching exports if any exist

SCENARIO 5: Optional expect with fallback [AFFECTED] [SEC:Functions]
  GIVEN: spec-a EXPECTS "LoggingPolicy" from ANY_DEPENDENCY (required=false)
  AND:   fallback = "level=INFO, format=JSON"
  AND:   no dependency exports a LoggingPolicy
  WHEN:  resolve is run with spec-a as entry
  THEN:  manifest contains the fallback as a POLICY rule
  AND:   manifest.unmatched lists this expect

SCENARIO 6: Conditional export only applies when condition met [FULL] [SEC:Functions]
  GIVEN: spec-b EXPORTS a CONSTRAINT with condition="consumer.has_gpu == true"
  AND:   spec-a IMPORTS spec-b and declares gpu capability
  AND:   spec-c IMPORTS spec-b and does NOT declare gpu capability
  WHEN:  resolve is run for spec-a
  THEN:  manifest includes the GPU constraint
  WHEN:  resolve is run for spec-c
  THEN:  manifest does NOT include the GPU constraint

SCENARIO 7: Graph depth tiebreaker [FULL] [SEC:Functions]
  GIVEN: spec-a IMPORTS spec-b IMPORTS spec-c
  AND:   spec-a also IMPORTS spec-c directly
  AND:   spec-c EXPORTS a POLICY
  WHEN:  resolve is run with spec-a as entry
  THEN:  the POLICY has graph_depth=1 (direct import wins over transitive)

SCENARIO 8: Cycle in dependency graph [FULL] [SEC:Functions]
  GIVEN: spec-a IMPORTS spec-b, spec-b IMPORTS spec-a
  AND:   both have EXPORTS
  WHEN:  resolve is run with spec-a as entry
  THEN:  CycleDetected warning is emitted
  AND:   both specs' exports are collected (each processed once)
  AND:   manifest is produced successfully

SCENARIO 9: Stale manifest detection [AFFECTED] [SEC:Functions]
  GIVEN: a seed-manifest.json was generated from a spec graph
  AND:   one spec in the graph has been modified since generation
  WHEN:  validate is run on the manifest
  THEN:  StaleManifest warning is emitted with the changed spec identified

SCENARIO 10: Audit trail [FULL] [SEC:API]
  GIVEN: a valid seed manifest with 5 rules
  WHEN:  audit is run for rule_id "spec-c.PhysicalMemoryLimit"
  THEN:  output shows: rule → SeedExport → spec-c Contracts section → [SEC:DataModel.2]
  AND:   includes the original EXPORT block text
```

---

## Dependencies

```
DEPENDENCY @nlspec/core:
  version  : >=0.1.0
  provides : Spec parser, graph walker, Contracts section parsing
  source   : nlspec/bootstrap-spec

DEPENDENCY Node.js:
  version  : >=20.0.0
  provides : Runtime environment
```

---

## FileStructure Structure

```
nlspec-seed-resolver/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              -- entry point, exports public API
│   ├── walker.ts             -- walk_graph: dependency graph traversal
│   ├── collector.ts          -- collect_exports, collect_expects
│   ├── matcher.ts            -- match_contracts
│   ├── conflict.ts           -- detect_conflicts, resolve_conflicts
│   ├── emitter.ts            -- emit_manifest
│   ├── validator.ts          -- validate_manifest
│   ├── cli.ts                -- CLI command handlers
│   ├── mcp.ts                -- MCP tool handlers
│   └── types.ts              -- all RECORD and ENUM type definitions
├── tests/
│   ├── fixtures/
│   │   ├── simple-chain/     -- specs for SCENARIO 1
│   │   ├── never-conflict/   -- specs for SCENARIO 2
│   │   ├── restrictive/      -- specs for SCENARIO 3
│   │   ├── unmatched/        -- specs for SCENARIO 4
│   │   ├── fallback/         -- specs for SCENARIO 5
│   │   ├── conditional/      -- specs for SCENARIO 6
│   │   ├── depth-tie/        -- specs for SCENARIO 7
│   │   ├── cycle/            -- specs for SCENARIO 8
│   │   └── stale/            -- specs for SCENARIO 9
│   ├── walker.test.ts
│   ├── collector.test.ts
│   ├── matcher.test.ts
│   ├── conflict.test.ts
│   ├── emitter.test.ts
│   ├── validator.test.ts
│   └── integration.test.ts   -- full resolve pipeline
└── seed-resolver-spec.md     -- this file
```

---

## Maintenance Workflow

```
BUG CATEGORIES:
  - Resolution bugs: wrong rule wins a conflict (fix in conflict.ts)
  - Collection bugs: exports or expects missed during parsing (fix in collector.ts)
  - Graph bugs: incorrect traversal order or cycle handling (fix in walker.ts)
  - Manifest bugs: invalid or incomplete seed manifest (fix in emitter.ts)

PATCH FORMAT:
  Standard nlspec patch format. SCENARIO is immutable — if a scenario fails,
  the implementation is wrong, not the scenario.

SCENARIO TIERS:
  [SMOKE]    → Run on every change. SCENARIOS 1, 2, 4.
  [AFFECTED] → Run when related code changes. SCENARIOS 3, 5, 9.
  [FULL]     → Run before release. SCENARIOS 6, 7, 8, 10.
```

---

## BuildAndRun and Run

```
# Install
npm install

# Build
npm run build

# Test
npm test                           # all scenarios
npm test -- --grep "SMOKE"         # smoke only

# Run
npx nlspec-seed resolve specs/my-entry-spec.md
npx nlspec-seed resolve specs/my-entry-spec.md --dry-run --verbose
npx nlspec-seed validate build/seed-manifest.json
npx nlspec-seed diff build/old-manifest.json build/new-manifest.json
npx nlspec-seed audit build/seed-manifest.json --rule "spec-c.PhysicalMemoryLimit"
```

---

## Boundaries

```
### This System Does:
- Walk spec dependency graphs and collect Contracts section contracts
- Validate that all required EXPECTS are matched by EXPORTS
- Detect and resolve (or reject) conflicting exports
- Produce a deterministic seed manifest from a spec graph
- Provide full provenance for every rule in the manifest
- Detect stale manifests when specs change
- Expose functionality as CLI and MCP tools

### This System Does NOT:
- Apply seed rules to a running system (that's the runtime's job)
- Monitor invariants at runtime (that's the runtime's job)
- Edit or modify specs (that's the spec author's job)
- Resolve semantic conflicts (only structural/type-based resolution)
- Manage spec versions or history (use git)
- Execute code generation from the manifest (agent's job)

### Future Extensions:
- Visual dependency graph with conflict highlighting
- Incremental resolution (only re-resolve changed subgraph)
- Semantic conflict resolution via LLM analysis
- Integration with CI/CD pipelines as a pre-deploy gate
```

---

## Contracts Contracts

```
### EXPORTS

EXPORT SeedManifestSchema:
  type        : SEED_DATA
  target      : runtime_initializer
  condition   : ALWAYS
  value       : "SeedManifest JSON schema for parsing and applying seed rules"
  override    : NEVER
  source_ref  : [SEC:DataModel]

EXPORT ConflictResolutionRules:
  type        : POLICY
  target      : build_system
  condition   : ALWAYS
  value       : "Conflict precedence: NEVER > restrictive > closer_to_root"
  override    : WITH_JUSTIFICATION
  source_ref  : [SEC:Functions]

### EXPECTS

EXPECTS SpecParser:
  from        : nlspec/bootstrap-spec
  type        : CONSTRAINT
  description : "Ability to parse spec markdown into structured sections and elements"
  required    : true
  fallback    : none

EXPECTS GraphWalker:
  from        : nlspec/bootstrap-spec
  type        : CONSTRAINT
  description : "Ability to traverse IMPORTS declarations across specs"
  required    : true
  fallback    : none

### CONFLICT RESOLUTION

Standard rules from NLSPEC-TEMPLATE Contracts section apply.
```
