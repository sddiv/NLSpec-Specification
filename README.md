# nlspec — A Specification for Organizational AI

Write specs, not code. Hand the spec to an agent. Get a working implementation.

---

## What Is This?

A specification for writing, managing, and serving natural language specifications
to AI coding agents. You start with one spec file and one agent. As your system
grows, the tooling grows with you — from raw markdown to structured access to
multi-spec decomposition to organizational scale.

This is not a document format. It is not a workflow tool. It is a specification for
treating specs as structured, queryable, composable data that agents consume at scale.

## The Bootstrap Path

Nobody starts with an organizational spec. Everyone starts with one file.

| Phase | What You Have | What nlspec Provides | Trigger for Next |
|-------|--------------|---------------------|------------------|
| **0** | 1 spec, 1 agent | Template + CLAUDE.md (no tooling) | Spec > 3,000 lines |
| **1** | 1 large spec | Bootstrap server (8 MCP tools) | Need to split by concern |
| **2a** | N specs | `nlspec_split`, namespaces, cross-spec queries | Need deterministic init |
| **2b** | N specs with deps | Seed resolver, dependency contracts | Multiple projects/teams |
| **3** | N projects, N teams | Org-scale management, patterns, assets | — |

**Phase 0 is where everyone starts. Phase 0 is where we are now.** You copy the
template, fill it in, hand it to an agent. No server, no namespaces, no MCP tools.
Just a markdown file. This already works with any agent today.

Each phase is self-contained. You don't need Phase 2 to get value from Phase 1.
You don't need Phase 1 to get value from Phase 0. The system meets you where you are.

Read `NLSPEC-SYSTEM.md` for the full description of each phase.

## What's In This Repo?

```
nlspec/
├── README.md                    # You're reading it
├── LICENSE                      # CC-BY-4.0
├── NLSPEC-SYSTEM.md             # Phases, spec graph, agent roles
├── NLSPEC-TEMPLATE.md           # The spec template (supports TYPE: SYSTEM | PATTERN | ASSET)
├── CLAUDE.md                    # Agent instructions (Phase 0 — read files directly)
└── specs/
    ├── CLAUDE-MCP.md            # Agent instructions (Phase 1+ — use MCP tools)
    ├── bootstrap-spec.md        # Phase 1: parser, store, query, 8 tools
    ├── mcp-server-spec.md       # Phase 2a: slice, patch, validate, split, namespaces, substrates
    ├── seed-resolver-spec.md    # Phase 2b: dependency contracts, seed manifests
    └── pattern-catalog.md       # Prior art pattern reference (TYPE: CATALOG)
```

| File | Purpose | Phase |
|------|---------|-------|
| `NLSPEC-TEMPLATE.md` | The blank spec format. Copy it, fill it in. | 0+ |
| `CLAUDE.md` | Agent reads spec files directly from disk. 7 modes: DESIGN → SPEC → IMPLEMENT → VALIDATE + FIX, CONSOLIDATE, DESCRIBE | 0 |
| `specs/CLAUDE-MCP.md` | Agent uses MCP tools for structured spec access. Same 7 modes. | 1+ |
| `NLSPEC-SYSTEM.md` | How specs grow with your system. Phases, layers, substrates, S3+OTF. | All |
| `specs/bootstrap-spec.md` | Core system — parser, store, query engine, 8 MCP tools | 1+ |
| `specs/mcp-server-spec.md` | Extensions — decomposition, namespaces, substrates, S3+OTF storage | 2a+ |
| `specs/seed-resolver-spec.md` | Dependency contracts — graph walker, contract matcher, seed manifest | 2b+ |
| `specs/pattern-catalog.md` | Prior art patterns — names + descriptions, agent uses training knowledge | 0+ |

## Quick Start (Phase 0)

### 1. Start a new project

```bash
mkdir my-project && cd my-project
cp path/to/nlspec/CLAUDE.md .
cp path/to/nlspec/NLSPEC-TEMPLATE.md specs/my-project-spec.md
```

### 2. Fill in the spec

Open `specs/my-project-spec.md` and replace every `{placeholder}` with your actual
specification. The declared sections guide you through everything from abstract to deployment.

### 3. Hand it to the agent

```
claude-code> Implement the spec defined in specs/my-project-spec.md
```

The agent reads `CLAUDE.md` for operating instructions and the spec for what to build.

### 4. Fix bugs with context, not rewrites

```
claude-code> SCENARIO 7 is failing. Fix it.
```

The agent reads the Table of Contents, identifies the affected section, reads only
what it needs, fixes the code, and runs SMOKE + AFFECTED scenarios.

### 5. When you outgrow Phase 0

Your spec hits 3,000+ lines. The agent wastes context reading everything. Time for
Phase 1: run the bootstrap server, swap CLAUDE.md for CLAUDE-MCP.md, and get
structured access via MCP tools.

```bash
pip install nlspec-server                          # or add to pyproject.toml
nlspec-server --specs-dir ./specs                  # stdio for local dev
cp path/to/nlspec/specs/CLAUDE-MCP.md CLAUDE.md    # replace Phase 0 instructions
```

### 6. When you need to split

The monolith spec has too many concerns. Time for Phase 2: use `nlspec_split` to
decompose into multiple specs with correct imports between them.

```
nlspec_split({namespace: "myproject", spec_id: "monolith", strategy: "cluster"})
```

## The Template: Recommended Sections

| Section | What It Defines |
|---------|----------------|
| Abstract | One paragraph: what this spec defines |
| Problem | Current state → deficiency → target → key insight |
| Architecture | ASCII diagram, component inventory, data flows, patterns, assets |
| DataModel | RECORD, ENUM, ALIAS definitions with invariants |
| Functions | FUNCTION specs: inputs, outputs, behavior, errors |
| API | HTTP endpoints, gRPC services, CLI commands, MCP tools |
| Errors | Complete error hierarchy with handling rules |
| Config | Every knob: type, default, env var, validation |
| SystemManifest | Platform composition, hardware, kernels, environment configs |
| Deployment | Container images, K8s manifests, PIPELINE definitions |
| UXSpec | Scenes, transitions, view hierarchy, visual verification, UX test scenarios |
| Scenarios | End-to-end behavioral tests (the holdout set) |
| Dependencies | External systems, libraries, and asset packs with pinned versions and verify-after-install rules |
| FileStructure | Expected directory layout |
| Maintenance | Bug categories, patch format, scenario tiers |
| BuildAndRun | Exact commands to build, test, run, verify, artifact manifest |
| AgentPipeline | AI agent execution protocol (provision → build → deploy → validate) |
| Boundaries | Explicit non-goals (prevents scope creep) |
| Contracts | EXPORTS, EXPECTS, layer-aware contracts, conflict resolution |
| LayerContext | (Optional) Layer composition: derivation, stack, constraint flow |

Sections are declared per-spec. PATTERN and ASSET specs use different sections.
LayerContext is optional across all spec types — include it when composing specs
across the 4-layer model.

## Key Concepts

**Specs are the source of truth, not code.** Code is a derived artifact. When the
spec and code disagree, the spec is correct.

**Scenarios are immutable.** The agent cannot modify scenarios to make tests pass.
If a scenario fails, the implementation is wrong.

**Context slicing, not full-spec feeding.** Every FUNCTION declares `USES:`. Every
RECORD declares `USED BY:`. Every SCENARIO declares `[SEC:SectionName.x]`. The agent walks
these edges to assemble a minimal context slice.

**Specs compose as a graph.** Any spec can import from any other spec. Relationships
are many-to-many. No fixed hierarchy. No mandatory naming.

**Dependency contracts.** The Contracts section declares what a spec EXPORTS (constraints,
invariants, seed data, policies) and what it EXPECTS from dependencies. The seed
resolver walks the graph and produces a deterministic seed manifest — the complete
initial state of the system. No manual configuration, no hidden defaults.

**Spec types.** SYSTEM specs produce running code (the default). PATTERN specs
define reusable architectural blueprints — consumed via `USES PATTERN:` in
Architecture.3. ASSET specs catalog static resources that already exist in the
workspace (images, CSS, fonts, tokens) and define the style guide (RULEs and
TOKENs) that agents must follow when writing code — consumed via `USES ASSET:`.
ASSET specs with a COMPONENT_CATALOG section serve as design system registrations
for UI rendering. The agent applies prior art patterns from training knowledge and
reads full specs for novel patterns and assets.

**UX specifications.** The optional UXSpec section defines frontend UX as structured,
verifiable elements — not mockups or wireframes. SCENEs compose design system
components with layout, responsive rules, data bindings, z-layer stacking, and
overlays. TRANSITIONs define navigation between scenes with animation. View
hierarchy rules govern z-index layering and focus trapping. Visual verification
runs three layers: structural DOM checks, token compliance (computed style vs design
tokens), and AI-assisted visual regression. UX test scenarios drive headless
browsers through complete user flows with NAVIGATE and VISUAL assertions. This
approach eliminates the need for Figma or separate design tools — the spec IS
the design, and it's machine-verifiable.

**Two-dimensional spec composition (optional).** Vertical (DERIVES FROM across layers)
+ horizontal (COMPOSES WITH at the same layer). See below.

**Three-dimensional substrates (optional).** When a project branches into multiple
platforms, versions, or feature variants, each branch forms a substrate — a complete
layer stack diverging from a shared ancestor. The MCP server detects substrates from
the DERIVES FROM graph and resolves linear spec chains for agents. See `NLSPEC-SYSTEM.md`.

**Agent pipeline.** Seven modes: DESIGN → SPEC → IMPLEMENT → VALIDATE, plus FIX,
CONSOLIDATE, and DESCRIBE. DESIGN produces the architecture (Abstract, Problem, Architecture, Boundaries, Contracts).
SPEC fills in the details. IMPLEMENT writes the code. Each mode has machine-readable
exit criteria, autonomous failure classification (SPEC_GAP, IMPL_BUG, ENV_ISSUE,
AMBIGUITY), retry logic, and structured escalation reports. The pipeline can run
end-to-end without human intervention (dark factory mode).

**Dark factory ready.** Autonomous execution chains modes into a self-driving DAG:
validate spec → implement → validate code → fix failures → report. Artifact manifests
define "done." Drift detection catches spec-vs-code divergence. Structured JSON
reports (pipeline-result, validation-report, escalation-report) make every decision
traceable.

**The system grows with you.** Phase 0 requires nothing but a template. Phase 3
manages an organization. Same format at every phase.

## Two-Dimensional Spec Composition

Without the layer model, nlspec specs form a flat dependency graph. Spec A imports
from Spec B. That's one dimension: specs linked by IMPORT/EXPORT. It works, but it
doesn't capture the fact that a product exists at multiple levels of abstraction
simultaneously, and that a single layer of that product is often too large for one spec.

The 4-layer composition model adds a second dimension. Specs now compose in two ways:

```
                        HORIZONTAL (same layer, different concerns)
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    │    kernel ←──→ filesystem ←──→ networking    │
                    │      │             │              │          │
                    │      └─────────────┴──────────────┘          │
                    │         all L2, composing together           │
                    └──────────────────────────────────────────────┘
                                        │
              VERTICAL                  │
           (across layers,              │
            derivation)                 │
                │                       │
                ▼                       ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  L1  os-spec              Contracts, interfaces, invariants  │
  │       │                                                      │
  │       ▼                                                      │
  │  L2  kernel + fs + net    Platform-specific realization      │
  │       │                                                      │
  │       ▼                                                      │
  │  L3  server-config        Domain-specific tuning             │
  │       │                                                      │
  │       ▼                                                      │
  │  L4  db-server-profile    User-specific preferences          │
  └──────────────────────────────────────────────────────────────┘
```

**Vertical composition** is derivation across layers. Each layer inherits from
the one above it and adds its own concerns. One L1 specification produces many L2
realizations (React vs iOS vs CLI). Each L2 supports many L3 configurations (healthcare
vs education vs enterprise). Each L3 serves many L4 user profiles (radiologist vs nurse
vs admin). Swapping at any level doesn't disturb the levels above or below — that's
the substitution boundary property.

**Horizontal composition** is multiple specs at the same layer composing together.
A real system at any layer is rarely one spec. An OS realization is kernel + filesystem
+ networking + drivers. A web app realization is frontend + backend + database schema +
message queue. Each of these is its own spec with its own DataModel, Functions, API,
and Scenarios, but they share a layer, derive from the same parent, and define
interfaces between each other via `COMPOSES WITH`.

The two dimensions are orthogonal. You can use vertical-only (simple products with one
spec per layer), horizontal-only (a complex system at a single layer of abstraction),
or both (complex systems at multiple levels of abstraction). The layer model is opt-in —
specs without a `Layer:` declaration work exactly as before.

**Constraint flow is three-directional.** Downward (parent constrains child — the
default). Upward (child preferences propagate to parent — for non-negotiable user
requirements). Lateral (peers constrain each other through shared interfaces —
kernel provides block device API, filesystem depends on it).

**Relationship types for horizontal peers.** `co-required` means both must be present
(kernel + filesystem). `optional` means the peer adds capability but isn't necessary
(GPU driver). `alternative` means only one of the set is used (ext4 vs btrfs).

See `NLSPEC-TEMPLATE.md` section LayerContext for the full declaration syntax:
DERIVES FROM (vertical parent), COMPOSES WITH (horizontal peers), LAYER STACK
(the full 2D tree), CONSTRAINT FLOW (downward + upward + lateral), and SUBSTITUTION
BOUNDARY (what's swappable at each layer).

## Three-Dimensional Substrates

The 2D model captures composition within a single product variant. But real projects
branch: one L1 spec produces web, iOS, and android L2 realizations, each with its own
L3/L4 stack. These branches are **substrates** — parallel layer stacks sharing an
ancestor but diverging at a branching point.

```
                    L1: Payment Contracts
                   /         |          \
          L2: Web           L2: iOS          L2: Android
          /    \            /    \            /    \
     L3: Prod  L3: Dev  L3: Prod  L3: Dev  L3: Prod  L3: Dev
```

Substrates are polymorphic. The branching axis can represent platform variants
(spatial), version history (temporal), or feature flags (feature). NLSpec's own phase
system is a temporal substrate: Phase 1 → Phase 2a → Phase 2b is a progression where
each phase carries its own independent layer stack.

Substrates are **not declared** — they are inferred from the DERIVES FROM graph by the
MCP server. The server builds the substrate topology on-the-fly in memory and caches it.

For agents, the key tool is `nlspec_substrate_query`: ask for a substrate by name, get
a linear L1→L4 chain back. The agent doesn't need to understand the full 3D topology —
just ask for "ios" and get the payment L1, iOS L2, iOS-prod L3, and US L4.

For humans, the substrate graph powers 3D visualization — X-axis for horizontal
composition, Y-axis for layer derivation, Z-axis for substrate branching.

Spec files remain markdown in a filesystem (local or S3). The MCP server builds two
derived indices: SQLite for element-level search, and an in-memory graph for substrate
topology. Both are rebuildable from the markdown files at any time.

Read `NLSPEC-SYSTEM.md` for the full substrate model.

## How nlspec Differs from Existing Tools

The term "nlspec" was coined by [StrongDM](https://factory.strongdm.ai/) to mean
a human-readable spec directly usable by coding agents. We formalize and extend that
idea into a complete specification for organizational-scale AI development.

| | StrongDM nlspec | GitHub Spec Kit | OpenSpec / BMAD / Kiro | **nlspec (this)** |
|---|---|---|---|---|
| **What it is** | 3 markdown files for one product | CLI + workflow scaffold | Workflow tools for SDD | A specification |
| **Spec format** | Implicit (RECORD, FUNCTION, etc.) | Freeform markdown | Freeform markdown | Formalized template with typed elements (SYSTEM, PATTERN, ASSET) |
| **Tooling model** | None (hand files to agent) | Slash commands | Slash commands | MCP server (programmatic access) |
| **Element types** | Used but not defined | None | None | RECORD, FUNCTION, SCENARIO, ENDPOINT, PIPELINE — first-class, queryable |
| **Cross-references** | None | None | None | USES, USED BY, SEC tags — agents walk edges for context slicing |
| **Multi-spec** | Independent files | One spec per project | One spec per change | N specs as directed graph with typed imports |
| **Growth path** | None | None | None | Phase 0 → 1 → 2 → 3 with tooling at each step |
| **Decomposition** | Manual | Manual | Manual | `nlspec_split` analyzes and decomposes specs |
| **Autonomous exec** | None | None | None | Dark factory mode: self-driving pipeline with retry, escalation, artifact verification |
| **Composition** | None | None | None | 3D: vertical (4-layer derivation) + horizontal (same-layer peers) + substrate branching (platforms, versions, features) |
| **UX specification** | None | None | None | SCENEs, TRANSITIONs, visual verification, UX scenarios — design is spec, not mockup |
| **Self-describing** | No | No | No | Yes — the spec for nlspec is itself an nlspec |

## Why This Will Be the Norm

In 2-3 years, most production software will be written by agents. The bottleneck moves
from code to specs. Whoever controls the spec controls the system.

Today's SDD tools treat specs as documents — markdown files that a human writes and
an agent reads once. That works for a single developer on a single project. It doesn't
work when you have 50 agents across 200 specs with cross-cutting concerns.

The progression:
- **2024-2025:** Vibe coding. Agents write code from prompts. No specs.
- **2025-2026:** Spec-driven development. Specs as documents. One spec, one agent.
  (StrongDM, Spec Kit, OpenSpec, Kiro — we are here.)
- **2026-2027:** Spec-managed organizations. Specs as structured data. N specs, N agents,
  spec graphs, MCP access, CI/CD integration. Specs are the org chart for AI.
  (nlspec — this is where we're going.)

## License

CC-BY-4.0 — use it, adapt it, share it. Just give credit.

## Credits

- Term "nlspec" coined by [StrongDM's Software Factory](https://factory.strongdm.ai/)
  and their [Attractor](https://github.com/strongdm/attractor)
- Created by Divyendu Deepak Singh
