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
    ├── mcp-server-spec.md       # Phase 2a: slice, patch, validate, split, namespaces
    ├── seed-resolver-spec.md    # Phase 2b: dependency contracts, seed manifests
    └── pattern-catalog.md       # Prior art pattern reference (TYPE: CATALOG)
```

| File | Purpose | Phase |
|------|---------|-------|
| `NLSPEC-TEMPLATE.md` | The blank spec format. Copy it, fill it in. | 0+ |
| `CLAUDE.md` | Agent reads spec files directly from disk. 7 modes: DESIGN → SPEC → IMPLEMENT → VALIDATE + FIX, CONSOLIDATE, DESCRIBE | 0 |
| `specs/CLAUDE-MCP.md` | Agent uses MCP tools for structured spec access. Same 7 modes. | 1+ |
| `NLSPEC-SYSTEM.md` | How specs grow with your system. | All |
| `specs/bootstrap-spec.md` | Core system — parser, store, query engine, 8 MCP tools | 1+ |
| `specs/mcp-server-spec.md` | Extensions — decomposition, namespaces, 8 more tools | 2a+ |
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
npx @nlspec/server --specs-dir ./specs
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
| Deployment | Container images, K8s manifests, PIPELINE definitions |
| Scenarios | End-to-end behavioral tests (the holdout set) |
| Dependencies | External systems, libraries, and asset packs with pinned versions and verify-after-install rules |
| FileStructure | Expected directory layout |
| Maintenance | Bug categories, patch format, scenario tiers |
| BuildAndRun | Exact commands to build, test, run, verify |
| Boundaries | Explicit non-goals (prevents scope creep) |
| Contracts | EXPORTS, EXPECTS, conflict resolution (for multi-spec projects) |

Sections are declared per-spec. PATTERN and ASSET specs use different sections.

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
The agent applies prior art patterns from training knowledge and reads full specs
for novel patterns and assets.

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
