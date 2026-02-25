# nlspec — A Specification for Organizational AI

Write specs, not code. Hand the spec to an agent. Get a working system.

---

## What Is This?

A specification system for AI coding agents. You write a detailed spec in natural
language using a structured 16-section template. An agent reads the spec and builds
the system. Specs compose into a dependency graph. Dependencies export contracts
that seed the system with rules at build time. The spec is the source of truth —
code is a derived artifact.

This is not a document format. It is not a workflow tool. It is a specification for
treating specs as structured, queryable, composable data that agents consume at scale.

## The Bootstrap Path

Nobody starts with an organizational spec. Everyone starts with one file.

| Phase | What You Have | What nlspec Provides | Trigger for Next |
|-------|--------------|---------------------|------------------|
| **0** | 1 spec, 1 agent | Template + CLAUDE.md (no tooling) | Spec > 3,000 lines |
| **1** | 1 large spec | Bootstrap server (8 MCP tools) | Need to split by concern |
| **2** | N specs | Split, namespaces, seed resolver, cross-spec queries | Multiple projects/teams |
| **3** | N projects, N teams | Org-scale management, agent orchestration | — |

**Phase 0 is where everyone starts. Phase 0 is where I am now.** You copy the
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
├── NLSPEC-SYSTEM.md             # Phases, spec graph, agent roles, MCP vs A2A
├── NLSPEC-TEMPLATE.md           # The 16-section spec template
├── CLAUDE.md                    # Agent instructions (Phase 0 — read files directly)
└── specs/
    ├── CLAUDE-MCP.md            # Agent instructions (Phase 1+ — use MCP tools)
    ├── bootstrap-spec.md        # Phase 1: parser, store, query, 8 tools
    ├── mcp-server-spec.md       # Phase 2: slice, patch, validate, split, namespaces
    └── seed-resolver-spec.md    # Phase 2: dependency contract resolution, seed manifests
```

| File | Purpose | Phase |
|------|---------|-------|
| `NLSPEC-TEMPLATE.md` | The blank spec format. Copy it, fill it in. | 0+ |
| `CLAUDE.md` | Agent reads spec files directly from disk. | 0 |
| `specs/CLAUDE-MCP.md` | Agent uses MCP tools for structured spec access. | 1+ |
| `NLSPEC-SYSTEM.md` | How specs grow with your system. | All |
| `specs/bootstrap-spec.md` | Core: parser, store, query engine, 8 MCP tools. | 1 |
| `specs/mcp-server-spec.md` | Extensions: slicing, patches, validation, graph, namespaces. | 2 |
| `specs/seed-resolver-spec.md` | Dependency contracts: resolve exports, produce seed manifests. | 2 |

## Quick Start (Phase 0)

### 1. Start a new project

```bash
mkdir my-project && cd my-project
cp path/to/nlspec/CLAUDE.md .
cp path/to/nlspec/NLSPEC-TEMPLATE.md specs/my-project-spec.md
```

### 2. Fill in the spec

Open `specs/my-project-spec.md` and replace every `{placeholder}` with your actual
specification. The 16 sections guide you through everything from abstract to
dependency contracts.

### 3. Hand it to the agent

```
claude-code> Implement the system defined in specs/my-project-spec.md
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

When specs start importing from each other, the seed resolver walks the dependency
graph and produces a deterministic seed manifest — the complete set of constraints,
invariants, policies, and initial data the system must boot with. No manual wiring.

```
nlspec-seed resolve specs/my-entry-spec.md
```

## The Template: 16 Sections

| # | Section | What It Defines |
|---|---------|----------------|
| 1 | Abstract | One paragraph: what this system is |
| 2 | Problem Statement | Current state → deficiency → target → key insight |
| 3 | Architecture Overview | ASCII diagram, component inventory, data flows |
| 4 | Data Model | RECORD, ENUM, ALIAS definitions with invariants |
| 5 | Core Functions | FUNCTION specs: inputs, outputs, behavior, errors |
| 6 | API Surface | HTTP endpoints, gRPC services, CLI commands, MCP tools |
| 7 | Error Model | Complete error hierarchy with handling rules |
| 8 | Configuration | Every knob: type, default, env var, validation |
| 9 | Deployment Artifacts | Container images, K8s manifests, PIPELINE definitions |
| 10 | Scenarios | End-to-end behavioral tests (the holdout set) |
| 11 | Dependencies | External systems and libraries |
| 12 | File Structure | Expected directory layout |
| 13 | Maintenance Workflow | Bug categories, patch format, scenario tiers |
| 14 | Build and Run | Exact commands to build, test, run, verify |
| 15 | Boundaries | Explicit non-goals (prevents scope creep) |
| 16 | Dependency Contracts | EXPORTS, EXPECTS, conflict resolution rules |

## Key Concepts

**Specs are the source of truth, not code.** Code is a derived artifact. When the
spec and code disagree, the spec is correct.

**Scenarios are immutable.** The agent cannot modify scenarios to make tests pass.
If a scenario fails, the implementation is wrong.

**Context slicing, not full-spec feeding.** Every FUNCTION declares `USES:`. Every
RECORD declares `USED BY:`. Every SCENARIO declares `[SEC:x.x]`. The agent walks
these edges to assemble a minimal context slice.

**Specs compose as a graph.** Any spec can import from any other spec. Relationships
are many-to-many. No fixed hierarchy. No mandatory naming.

**Dependency contracts seed the system.** When spec A imports spec B, B's EXPORTS
(Section 16) automatically become constraints, invariants, policies, or initial data
in A's runtime. The seed resolver walks the graph and produces a deterministic
manifest — the initial state of the system is a computable function of its spec graph.
No manual configuration. No hidden defaults.

**The system grows with you.** Phase 0 requires nothing but a template. Phase 3
manages an organization. Same format at every phase.

## Build Order (for nlspec itself)

nlspec is specified by its own specs:

1. **Phase 0 (now):** The template and CLAUDE.md work standalone.
2. **Phase 1:** Implement `specs/bootstrap-spec.md` → parser, store, 8 MCP tools.
3. **Phase 2a:** Implement `specs/mcp-server-spec.md` → slicing, patches, validation, graph.
4. **Phase 2b:** Implement `specs/seed-resolver-spec.md` → dependency contracts, seed manifests.
5. **Self-hosting:** nlspec manages its own specs. The system builds itself.

## How nlspec Differs from Existing Tools

The term "nlspec" was coined by [StrongDM](https://factory.strongdm.ai/) to mean
a human-readable spec directly usable by coding agents. I formalize and extend that
idea into a complete specification for organizational-scale AI development.

| | StrongDM nlspec | GitHub Spec Kit | OpenSpec / BMAD / Kiro | **nlspec (this)** |
|---|---|---|---|---|
| **What it is** | 3 markdown files for one product | CLI + workflow scaffold | Workflow tools for SDD | A specification |
| **Spec format** | Implicit (RECORD, FUNCTION, etc.) | Freeform markdown | Freeform markdown | Formalized 16-section template with typed elements |
| **Tooling model** | None (hand files to agent) | Slash commands | Slash commands | MCP server (programmatic access) |
| **Element types** | Used but not defined | None | None | RECORD, FUNCTION, SCENARIO, ENDPOINT, PIPELINE — first-class, queryable |
| **Cross-references** | None | None | None | USES, USED BY, SEC tags — agents walk edges for context slicing |
| **Multi-spec** | Independent files | One spec per project | One spec per change | N specs as directed graph with typed imports |
| **Dependency contracts** | None | None | None | EXPORTS/EXPECTS with seed resolver — deterministic initialization |
| **Growth path** | None | None | None | Phase 0 → 1 → 2 → 3 with tooling at each step |
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
  (StrongDM, Spec Kit, OpenSpec, Kiro — this is where the industry is.)
- **2026-2027:** Spec-managed organizations. Specs as structured data. N specs, N agents,
  spec graphs, dependency contracts, MCP access, CI/CD integration. Specs are the org
  chart for AI. (nlspec — this is where I'm going.)

## License

CC-BY-4.0 — use it, adapt it, share it. Just give credit.

## Credits

- Term "nlspec" coined by [StrongDM's Software Factory](https://factory.strongdm.ai/)
  and their [Attractor](https://github.com/strongdm/attractor)
- Created by Divyendu Deepak Singh
