# nlspec — A Specification for Organizational AI

> This document describes how specs grow with your system. The format
> (RECORD, FUNCTION, SCENARIO) stays the same at every phase.
> What changes is how specs compose, how agents access them, and how
> the system helps you decompose as complexity increases.

---

## The Bootstrap Path

Nobody starts with an organizational spec. Everyone starts with one file.

nlspec is designed around this reality. You start simple and grow into
complexity. Each phase is self-contained — useful on its own, with a clear
trigger for when you need the next one. The same template works
at every phase. The tooling meets you where you are.

This applies to any system: a side project, a startup, an enterprise.
The spec system scales with the organization.

---

## Phase 0: One Spec, One Agent

**What you have:** A single spec file. A single agent. No tooling.

**How it works:** Copy NLSPEC-TEMPLATE.md, fill in the declared sections, hand it
to an agent. The agent reads the markdown directly from the filesystem and
implements your system.

```
my-project/
├── CLAUDE.md                    # Agent instructions
└── specs/
    └── my-project-spec.md       # Your spec (the only one)
```

**What you get:** A working system implemented from a detailed spec. The agent
reads the full document, implements section by section, validates against
scenarios. Bug fixes reference specific sections and scenarios.

**This is where everyone starts.** This is where nlspec itself is right now.
No server, no namespaces, no MCP tools. Just a markdown file and an agent.

**When you outgrow it:** Your spec hits 3,000+ lines. The agent reads 7,000
lines when it only needs 200. Bug fixes take longer because the agent wastes
context on irrelevant sections. Cross-references are followed manually.

---

## Phase 1: Structured Access

**Trigger:** Spec is too large for efficient agent access. Agent re-parses
the same markdown on every interaction. No search, no indexing.

**What changes:** You run the nlspec bootstrap server. The agent connects via
MCP and queries the spec programmatically instead of reading raw markdown.

```
my-project/
├── CLAUDE.md
├── specs/
│   └── my-project-spec.md       # Still one spec
└── .nlspec/
    └── index.sqlite             # Element index (auto-generated)
```

**What you get:** 8 MCP tools — `nlspec_get`, `nlspec_search`, `nlspec_list`,
`nlspec_create`, `nlspec_update`, `nlspec_delete`, `nlspec_init`. The agent
calls `nlspec_search({query: "validate_token", element_type: "FUNCTION"})` instead
of grepping through 3,000 lines. Reads 200 lines instead of 7,000.

**Still one spec.** The bootstrap server indexes it, makes it queryable, but
you're still working with a single document. The spec is still the source of
truth. The server is a read/write index on top of it.

**Defined by:** `specs/bootstrap-spec.md` — parser, store, query engine, 8 tools.

**When you outgrow it:** You need separate concerns — auth logic shouldn't be
in the same spec as deployment config. Or you have a second service. Or compliance
needs its own spec. The single spec is getting too broad. You need to split.

---

## Phase 2: First Split

**Trigger:** The monolith spec needs to decompose. Multiple concerns, multiple
services, or multiple teams need to work on different parts independently.

**What changes:** You split one spec into N specs. Specs live in namespaces.
Cross-spec imports track dependencies. The MCP server helps you decompose.

```
my-project/
├── CLAUDE.md
├── specs/
│   ├── auth-spec.md             # Auth concern
│   ├── storage-spec.md          # Storage concern
│   └── api-spec.md              # API surface (imports from auth + storage)
└── .nlspec/
    ├── index.sqlite             # All specs indexed
    └── namespaces.json          # Namespace registry
```

**What you get:** Context slicing across specs — the agent traces dependencies
from a failing scenario through imports into other specs and gets exactly what
it needs. Validation catches dangling cross-spec references. Namespaces organize
everything.

**Seed resolution:** When specs declare Contracts section dependency contracts (EXPORTS
and EXPECTS), the seed resolver walks the graph and produces a deterministic
seed manifest — the complete initial state of the system. This means no manual
configuration, no hidden defaults, no "works on my machine."

**Spec types:** Not all specs produce running code. A TYPE: PATTERN spec defines
a reusable architectural blueprint (e.g., a context stack pattern). A TYPE: ASSET
spec defines static resources and design constraints (e.g., a design system with
color tokens and typography rules). SYSTEM specs reference these via USES PATTERN
and USES ASSET in Architecture.3. The agent applies patterns from training knowledge
for well-known patterns, or reads the full PATTERN spec for novel ones.

```
nlspec_import({namespace: "myproject", spec_id: "auth", path: "specs/auth-spec.md"})
nlspec_import({namespace: "myproject", spec_id: "storage", path: "specs/storage-spec.md"})
nlspec_import({namespace: "myproject", spec_id: "api", path: "specs/api-spec.md"})
```

**The split itself can be assisted.** The `nlspec_split` tool analyzes a monolith
spec's dependency graph, identifies clusters of related elements, and decomposes
it into separate specs with correct IMPORT declarations between them.

```
nlspec_split({
  namespace: "myproject",
  spec_id: "monolith",
  strategy: "cluster",         # or "by_section", "by_concern"
  suggestions: true            # analyze but don't execute
})
→ Returns: {
    clusters: [
      {name: "auth", sections: ["DataModel.1", "DataModel.2", "Functions.1", "Functions.2"], elements: 47},
      {name: "storage", sections: ["DataModel.3", "Functions.3", "Functions.4"], elements: 31},
      {name: "api", sections: ["API.1", "API.2", "API.3"], elements: 22}
    ],
    cross_refs: [
      {from: "auth:Functions.1:function:validate_token", to: "storage:DataModel.3:record:Session"},
      ...
    ]
  }
```

**Defined by:** `specs/mcp-server-spec.md` — context slicing, validation,
graph operations, namespaces, `nlspec_import`, `nlspec_split`.

**When you outgrow it:** You have multiple projects, multiple teams, compliance
requirements that cross-cut everything. The spec graph IS your organization.

---

## Phase 3: Organizational Scale

**Trigger:** Multiple projects. Multiple teams. Cross-cutting concerns (compliance,
security, data governance) that affect everything. The spec graph mirrors the org.

**What it looks like:**

```
nlspec/bootstrap              → the system itself
nlspec/mcp-server             → the system's extensions
project-a/auth                → Project A's auth service
project-a/storage             → Project A's storage
project-a/api                 → Project A's API
project-b/ml-pipeline         → Project B's ML pipeline
project-b/inference           → Project B's inference service
org/compliance/gdpr           → GDPR compliance (cross-cuts everything)
org/compliance/sox            → SOX compliance
org/security/threat-model     → Security threat model
org/platform/k8s              → Shared Kubernetes platform
```

**What you get:** Agents specialize by role. The security agent reads
`org/security/*` and validates all project specs against threat models. The
compliance agent reads `org/compliance/*` and checks every service spec.
The platform agent manages `org/platform/*` and any project can import from it.

**Patch management matters here.** When the platform spec changes, every
project that imports from it needs to know. `nlspec_graph` shows the blast
radius. `nlspec_validate` catches broken imports. `nlspec_patch_create` tracks
the remediation across affected projects.

**This is where A2A enters.** An orchestrator routes work between agents.
Each agent uses MCP to read and write specs via nlspec. A2A coordinates
the handoffs:

```
A2A:  Orchestrator → "Security Agent, review project-a/auth changes"
MCP:  Security Agent → nlspec_slice({namespace: "project-a", spec_id: "auth", ...})
A2A:  Security Agent → "Orchestrator, LGTM" or "Orchestrator, PATCH-001 filed"
```

---

## Phase Transitions

Each transition has a clear trigger and the tooling to support it:

| Transition | Trigger | Tool |
|------------|---------|------|
| 0 → 1 | Spec > 3,000 lines; agent wastes context | Run bootstrap server |
| 1 → 2 | Multiple concerns in one spec; need to split | `nlspec_split` decomposes the monolith |
| 2 → 3 | Multiple projects/teams; cross-cutting concerns | Namespaces, graph operations, A2A orchestration |

**You can skip phases.** If you know you'll have multiple specs from the start,
go directly to Phase 2. If you're a solo developer with one service, you may
never leave Phase 0. The phases aren't mandatory — they're a growth path.

**You can go backwards.** Merged two services into one? Combine the specs.
Simplified your architecture? Drop namespaces. The tooling doesn't enforce
progression.

---

## Spec Relationships: A Graph, Not a Tree

When a project needs multiple specs (Phase 2+), those specs form a **directed
graph**. Any spec can import from any other spec. The relationships are
many-to-many. There is no fixed number of specs, no required hierarchy, and
no mandatory naming convention.

Each spec declares its imports. Each import creates an edge in the dependency
graph. The system tracks these edges and can query them: "What does this spec
depend on? What depends on it? Are there cycles?"

Cycles are allowed — real systems have them. A software spec might import
platform types, while the platform spec imports the software's health check
endpoints. The system warns about cycles but doesn't prevent them.

### Decomposition Patterns

Teams decompose specs however makes sense for their system. Common patterns:

- **By concern:** hardware, platform, software, quality, integration
- **By service:** one spec per microservice, with integration specs bridging them
- **By discipline:** engineering, security, compliance, UX, data governance
- **By lifecycle:** design spec, implementation spec, operations spec
- **Hybrid:** any combination, N specs deep, with any relationship topology

The system doesn't care about your taxonomy. It cares about the graph.

---

## Example Pattern: Five-Concern Decomposition

One common decomposition for systems with hardware and distributed concerns
splits into five specs by concern type:

```
                    ┌──────────────────┐
                    │  Integration     │
                    │  Spec            │
                    │  (topology,      │
                    │   contracts,     │
                    │   failure modes) │
                    └────────┬─────────┘
                             │ imports all
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──┐  ┌───────▼────┐  ┌──────▼──────┐
    │  Software  │  │  Platform  │  │  Quality    │
    │  Spec      │  │  Spec      │  │  Spec       │
    │  (app code,│  │  (K8s, CI, │  │  (test      │
    │   APIs,    │  │   runtime) │  │   strategy, │
    │   data)    │  │            │  │   scenarios)│
    └─────────┬──┘  └───────┬────┘  └─────────────┘
              │              │
              └──────┬───────┘
                     │ imports
           ┌─────────▼──────────┐
           │  Hardware Spec     │
           │  (GPU, memory,     │
           │   network, BIOS)   │
           └────────────────────┘
```

But this is just one pattern. You might have:
- 2 specs (software + deployment)
- 3 specs (one per microservice)
- 50 specs (enterprise with compliance, security, data governance, UX, ML, etc.)

The pattern emerges from your system's needs. `nlspec_split` can help you
discover natural decomposition boundaries.

---

## How Specs Compose

Imports create typed relationships between specs:

```
IMPORT StorageConfig FROM myproject/storage Config
  RELATIONSHIP: "uses configuration"

IMPORT Entry FROM myproject/storage DataModel.1
  RELATIONSHIP: "depends on data model"

IMPORT validate_token FROM myproject/auth Functions section.1
  RELATIONSHIP: "calls auth function"
```

These IMPORT declarations create edges in the spec graph. The system traverses
them during context slicing: when fixing a bug in the API spec, the agent
follows IMPORTs into auth and storage to get the full picture.

Real systems have complex import patterns. This is fine:

```
project-a/api imports from: project-a/auth, project-a/storage, org/platform/k8s
project-a/auth imports from: project-a/storage, org/security/threat-model
org/compliance/gdpr imports from: project-a/api (to validate data handling)
```

Compliance might cross-cut everything. Security might import from every service
to validate threat models. The graph is what it is — the system models it
faithfully, cycles and all.

---

## Agent Roles and What They See

At Phase 0-1, one agent sees everything. At Phase 2+, agents can specialize:

| Role | Reads | Does |
|------|-------|------|
| Software Agent | Software Spec + imported types | Implement features, fix bugs |
| SRE Agent | Platform Spec + deployment sections | K8s manifests, CI/CD, monitoring |
| QA Agent | Quality Spec + all scenarios | Run scenarios, file patches on failures |
| Security Agent | Security Spec + all API surfaces | Validate auth, audit compliance |
| Integration Agent | Integration Spec + all contract sections | Cross-service contracts, topology |

Each agent uses `nlspec_slice` to get exactly the context it needs. The security
agent doesn't read implementation details — it reads API surfaces and auth flows.
The QA agent doesn't read deployment config — it reads scenarios and the functions
they test.

---

## CI/CD as Part of the Spec

Specs can include PIPELINE definitions (Deployment section) that tell agents when and how
to run tests:

```
PIPELINE: pr-check
  TRIGGER: pull_request
  STEPS:
    - RUN: pytest -k "SMOKE"
    - RUN: pytest -k "AFFECTED" --sections "${CHANGED_SECTIONS}"
  NOTES:
    - SMOKE scenarios run on every PR
    - AFFECTED scenarios run only for sections touched by the PR
```

```
FILE_TO_SECTION_MAP:
  src/parser/*.ts       → Functions section.1
  src/store/*.ts        → Functions section.2
  src/query/*.ts        → Functions.3
  src/mcp/tools/*.ts    → API
```

When a file changes, the agent maps it to a section, finds tagged scenarios,
and runs exactly the right tests. Not the full suite — just what's affected.

---

## Distributed System Awareness

For distributed systems (Phase 2+), the Integration Spec adds three element
types beyond the standard template:

**TOPOLOGY:** How nodes connect.
```
TOPOLOGY: API Gateway → Auth Service → Token Store
  PROTOCOL: HTTPS
  FAILURE: If Auth Service is down, Gateway returns 503 with cached tokens
```

**CONTRACT:** Cross-service agreements.
```
CONTRACT: Auth Service → API Gateway
  ENDPOINT: POST /auth/validate
  REQUEST: { token: String }
  RESPONSE: { valid: bool, user_id: String, roles: List<String> }
  SLA: < 50ms p99
  VERSIONING: URL path (/v1/, /v2/)
```

**FAILURE_MODE:** What breaks and how you recover.
```
FAILURE_MODE: Token Store Unreachable
  DETECTION: Health check fails 3 consecutive times
  IMPACT: Auth Service cannot validate tokens
  MITIGATION: Fall back to JWT signature validation (no revocation check)
  RECOVERY: Automatic when health check passes
  SCENARIO: SCENARIO 45 [SEC:Errors.3] [FULL]
```

These elements follow the same pattern as any other element type — they're
parsed, indexed, queryable, and cross-referenceable.

---

## What Changes in the CLAUDE.md

At Phase 0, CLAUDE.md says:

```markdown
## Identity
You are implementing a system defined by specs/my-project-spec.md.
```

At Phase 2+, CLAUDE.md says:

```markdown
## Identity
You are implementing a system defined by multiple specs. Read the spec graph
to understand what you own and what you import.
Your primary specs: myproject/auth, myproject/storage
Your imports: org/platform/k8s, org/security/threat-model
```

The agent operating modes (IMPLEMENT, FIX, VALIDATE, CONSOLIDATE) work the
same at every phase. What changes is the scope: one spec vs. many specs.

---

## Why MCP, Not A2A

Two protocols are shaping the AI agent ecosystem:

- **MCP (Model Context Protocol):** How an agent connects to tools, data, and resources.
  Vertical integration. Agent ↔ capabilities.
- **A2A (Agent-to-Agent Protocol):** How agents communicate with each other.
  Horizontal integration. Agent ↔ agent.

nlspec is an MCP server because it is a **tool**, not an agent. It stores, indexes,
and serves specs. An agent calls `nlspec_get` the same way it calls a database query
or a file read. The agent does the reasoning. nlspec does the structured access.

A2A becomes relevant at Phase 3, when multiple agents need to coordinate:
the Software Agent finishes implementation and needs to hand off to the QA Agent,
or the QA Agent discovers a bug and needs to notify the Software Agent with a
failing scenario and section tags.

But that coordination layer sits **above** nlspec, not inside it. The orchestrator
uses A2A to route work between agents. Each agent uses MCP to read and write specs.
The two protocols are complementary:

```
A2A:   Orchestrator → "QA Agent, test Functions.3" → QA Agent → "Bug found, SCENARIO 7 fails"
MCP:   QA Agent → nlspec_get({section: "Functions.3"}) → read spec → nlspec_search({tags: ["SEC:Functions.3"]})
MCP:   QA Agent → nlspec_patch_create({sections: ["Functions.3"], scenarios: [7], ...})
A2A:   QA Agent → "Software Agent, PATCH-001 filed for Functions.3" → Software Agent
MCP:   Software Agent → nlspec_slice({scenario: 7}) → read context → fix code
```

At Phase 0-1, a single human orchestrates by talking to one agent at a time.
A2A isn't needed yet. When it is, nlspec remains an MCP server — the data layer
that all agents read from and write to.

---

## Namespaces

Specs live in namespaces (Phase 2+). A namespace is an organizational grouping —
a project, a team, or a domain.

```
nlspec/bootstrap              → the system itself
nlspec/mcp-server             → the system's extensions
myproject/auth                → your auth service
myproject/storage             → your storage service
org/compliance/gdpr           → organizational compliance spec
```

All MCP tools accept an optional `namespace` parameter. Without it, queries
search across all namespaces. With it, results are scoped.

### Self-Bootstrap

The nlspec system is defined by its own specs:
1. `specs/bootstrap-spec.md` — parser, store, query engine, 8 CRUD tools (Phase 1)
2. `specs/mcp-server-spec.md` — imports from bootstrap, adds context slicing,
   patches, validation, graph, namespaces, `nlspec_import`, `nlspec_split` (Phase 2a)
3. `specs/seed-resolver-spec.md` — imports from bootstrap and mcp-server, walks
   dependency graphs, resolves Contracts section contracts, produces seed manifests (Phase 2b)
4. `specs/pattern-catalog.md` — prior art pattern reference (TYPE: CATALOG)

Once all are implemented and the server is running:

```
nlspec_import({namespace: "nlspec", spec_id: "bootstrap", path: "specs/bootstrap-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "mcp-server", path: "specs/mcp-server-spec.md"})
nlspec_import({namespace: "nlspec", spec_id: "seed-resolver", path: "specs/seed-resolver-spec.md"})
```

The system now manages its own specs. Every change to nlspec goes through the
same tools it provides.

---

## Summary

| Phase | What You Have | What You Need | What nlspec Provides |
|-------|--------------|---------------|---------------------|
| **0** | 1 spec, 1 agent | Just a template | NLSPEC-TEMPLATE.md, CLAUDE.md |
| **1** | 1 large spec | Structured access | Bootstrap server (8 MCP tools) |
| **2a** | N specs | Decomposition, cross-spec queries | MCP server extensions, `nlspec_split`, namespaces |
| **2b** | N specs with dependencies | Deterministic initialization | Seed resolver, dependency contracts |
| **3** | N projects, N teams | Org-scale management, agent orchestration | Spec graph, validation, patches, patterns, assets |

The template remains the format at every phase. Complex projects have
multiple spec files forming a dependency graph, each following the same template
scoped to its concerns. The tooling grows with you — Phase 0 requires no tooling
at all.

---

## Layer Composition — The 4-Layer Model

Orthogonal to the phase system, specs can participate in a **4-layer composition
model** where one specification produces many realizations, each realization supports
many configurations, and each configuration serves many user profiles.

| Layer | Name | What It Defines |
|-------|------|-----------------|
| 1 | Specification | Contracts, interfaces, invariants — the possibility space |
| 2 | Realization | Platform-specific concretization of an L1 spec |
| 3 | Configuration | Domain or deployment tuning of an L2 realization |
| 4 | User Profile | Personalization constraints on an L3 configuration |

Layer composition is **opt-in** and **two-dimensional**. A spec declares its layer in
the header (`Layer:` field) and uses the `LayerContext` section to define:

- **Derivation + Horizontal Composition** (LayerContext.1) — which spec at the layer
  below this one derives from (vertical), and which specs at the same layer it composes
  with (horizontal). An OS realization is not one spec — it's kernel + filesystem +
  networking, all at L2, with COMPOSES WITH declarations defining their interfaces.
- **Layer Stack** (LayerContext.2) — the full tree of specs across all four layers,
  showing both vertical derivation and horizontal composition at each layer
- **Constraint Flow** (LayerContext.3) — how constraints propagate downward (default),
  optionally upward (when user preferences are non-negotiable), and laterally (between
  horizontally composed peers at the same layer)
- **Substitution Boundaries** (LayerContext.4) — what can be swapped at this layer
  without affecting other layers or horizontal peers
- **Cross-Layer References** (LayerContext.5) — `[Ln:spec-id:Section.x]` syntax for
  tracing constraints back to their source layer

Each layer is a clean substitution boundary. You swap an L2 realization (e.g., React →
native iOS) without changing the L1 spec above or the L3 configurations below. Within
a layer, you swap an optional horizontal component (e.g., GPU driver) without
disturbing co-required peers (kernel, filesystem). You swap an L4 user profile without
touching the L3 configuration it personalizes.

The layer model works at any phase. A Phase 0 project can declare Layer: 1-Specification
on a single spec file and later add L2/L3/L4 specs as the system grows. A Phase 2+
project with namespaces gains cross-layer validation via the MCP server's graph operations.

Specs without a Layer declaration are standalone — they work exactly as before. The
layer model adds composition semantics on top of the existing template, phases, and
tooling without changing them.

---

## Substrate Dimensions — The 3D Model

The 4-layer model is two-dimensional: vertical (DERIVES FROM) and horizontal
(COMPOSES WITH). But real projects branch along a third axis. An L1 specification
may produce three L2 realizations — web, iOS, android — each carrying independent
L3 and L4 stacks. This branching creates **substrates**: parallel spec stacks that
share an ancestor but diverge at a branching point.

### What Is a Substrate?

A substrate is a complete layer stack (or partial stack) that branches from another
stack at some layer. The branching point can be at any layer:

```
L1 branches → multiple L2s (most common: one spec, many platforms)
L1 branches → multiple L1s (multiple products in the same project)
L2 branches → multiple L3s (same realization, different deployments)
L3 branches → multiple L4s (same deployment, different user profiles)
```

Each substrate carries its own independent layers below the branching point. The
web substrate's L3 (webpack config, CDN settings) is completely independent of
the iOS substrate's L3 (Xcode build settings, App Store config). They share the
L1 above the branch but diverge below it.

```
                        L1: Payment Contracts
                       /         |          \
              L2: Web           L2: iOS          L2: Android
              /    \            /    \            /    \
         L3: Prod  L3: Dev  L3: Prod  L3: Dev  L3: Prod  L3: Dev
           |         |         |         |         |         |
         L4: US    L4: US    L4: US    L4: US    L4: US    L4: US
         L4: EU    L4: EU    L4: JP                        L4: IN
```

Each vertical column is a substrate. The full project topology is a tree of
substrates branching from shared ancestors.

### Substrate Types

Substrates are **polymorphic** — the branching axis can represent different
kinds of divergence:

| Substrate Type | What It Represents | Example |
|---------------|-------------------|---------|
| **Spatial** | Platform or target variants coexisting simultaneously | web, iOS, android |
| **Temporal** | Version history — the same system evolving over time | phase-1, phase-2a, phase-2b |
| **Feature** | Feature flag variants — different capabilities enabled | with-payments, without-payments |

The system does not enforce substrate types. A substrate is simply a branching
point in the DERIVES FROM graph where one spec produces multiple children at the
next layer (or the same layer, for L1→L1 branching).

### NLSpec's Own Phases as Temporal Substrates

NLSpec itself is a 3D spec. The phase system maps directly to temporal substrates:

```
L1: nlspec-core (the template, the format)
 ├── substrate: phase-1
 │   └── L2: bootstrap-spec (parser, store, 8 tools)
 │       └── L3: bootstrap-config (SQLite index, local files)
 │
 ├── substrate: phase-2a
 │   └── L2: mcp-server-spec (DERIVES FROM bootstrap, adds slice/patch/validate/graph)
 │       └── L3: mcp-server-config (namespaces, patches dir)
 │
 ├── substrate: phase-2b
 │   └── L2: seed-resolver-spec (DERIVES FROM mcp-server, adds dependency contracts)
 │       └── L3: seed-resolver-config
 │
 └── substrate: phase-3
     └── L2: org-scale-spec (planned — DERIVES FROM seed-resolver)
         └── L3: org-scale-config
```

Each phase is a substrate that carries its own L2/L3/L4 stack. Phase 2a subsumes
Phase 1 (it DERIVES FROM bootstrap). Phase 2b subsumes Phase 2a. The temporal
substrate axis is the versioning axis — you can "slice" at any phase and see a
coherent, self-contained system at that point in time.

### 3D Visualization as Versioning

The 3D model serves two audiences:

**For humans:** A spatial visualization where the X-axis is horizontal composition
(COMPOSES WITH), the Y-axis is layer derivation (DERIVES FROM), and the Z-axis is
substrate branching. You can rotate, zoom, slice at a substrate to see just that
stack. For temporal substrates, the Z-axis becomes a timeline — you see the system
evolving.

**For agents:** The MCP server resolves substrate paths. An agent asks:
"give me the spec chain for namespace=payments, substrate=ios, L1→L4" and gets
exactly the payment L1, the iOS L2, the iOS-prod L3, and the US L4 — a linear
chain cut from the 3D tree.

This makes the 3D model a **versioning tool**: temporal substrates let you navigate
the evolution of a spec system the same way git lets you navigate the evolution of
code. The difference is that spec evolution is semantic (Phase 1 adds structured access,
Phase 2a adds decomposition) rather than line-level diffs.

### Substrate Detection

Substrates are not declared explicitly. They are **inferred** from the existing
DERIVES FROM graph:

1. Walk the DERIVES FROM edges across all specs in a namespace
2. When a spec at layer N has multiple children at layer N+1 (or N), that's a
   branching point — each child is the root of a substrate
3. Follow each branch down through its L3/L4 stack to get the full substrate
4. Name substrates from the branching spec's identifier (e.g., the "ios" substrate
   is named from the L2 spec `l2-ios-realization`)

No new metadata is needed. The data already exists in the spec headers. The MCP
server's graph engine builds the substrate tree on-the-fly from DERIVES FROM edges.

### Cross-Substrate Interaction

Substrates are independent by default. The iOS L3 config does not affect the
Android L3 config. However, some projects need cross-substrate contracts:

- A shared API gateway that all platform substrates call
- A compliance spec that cross-cuts all substrates
- A shared component library used by web and mobile substrates

Cross-substrate interaction is handled through the existing COMPOSES WITH and
IMPORT mechanisms — a spec in one substrate can IMPORT from a spec in another.
The MCP server can detect these cross-substrate edges and include them in
substrate queries when requested.

**Scoping note:** Cross-substrate interaction detection and visualization is a
future extension. The initial substrate model focuses on detection, querying,
and linear chain resolution.

### Spec Storage and the On-The-Fly Graph

Specs remain markdown files — stored in a filesystem (local or S3). The markdown
file is the source of truth: human-readable, diffable, version-controllable.

The MCP server builds the substrate graph **on-the-fly** from the spec files:

1. On startup (or first query), read all spec files from the configured storage
2. Parse each spec's header: Layer, DERIVES FROM, COMPOSES WITH
3. Build an in-memory directed graph: nodes are specs, edges are derivation and
   composition relationships
4. Detect substrate branching points from the graph topology
5. Cache the graph. Invalidate when a spec file changes (filesystem watcher for
   local, S3 event notifications for cloud)

This is the **S3+OTF model**: files in S3 as the persistent source of truth, with
an on-the-fly graph built in memory by the MCP server. No graph database needed.
The graph is a derived index, like SQLite is a derived index of the parsed elements.

```
Source of truth:     specs/*.md (local filesystem or S3)
Derived index 1:     .nlspec/index.sqlite (element-level search)
Derived index 2:     in-memory graph (substrate topology, layer chains)
Cache invalidation:  filesystem watcher (local) or S3 event → SNS/SQS (cloud)
```

Both indices are rebuildable from the markdown files at any time. Lose the SQLite
file? Re-import. Restart the server? Graph rebuilds on first query.
