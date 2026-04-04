# nlspec — How Specs Compose

> This document describes how nlspec specs relate to each other: imports,
> graphs, layers, substrates. The format (RECORD, FUNCTION, SCENARIO)
> is defined in NLSPEC-TEMPLATE.md. This document defines how specs
> compose at scale.

---

## One Spec or Many

A project can have one spec or many. The template is the same either way.

A single spec works for small-to-medium systems. When a spec grows too large
or covers too many concerns, you split it. Auth logic shouldn't live in the
same spec as deployment config. Each concern gets its own spec.

```
my-project/
└── specs/
    ├── auth-spec.md             # Auth concern
    ├── storage-spec.md          # Storage concern
    └── api-spec.md              # API surface (imports from auth + storage)
```

The number of specs is unconstrained. A solo project might have one. An
enterprise might have fifty. The standard doesn't care — it defines how
specs relate to each other regardless of how many there are.

---

## Spec Relationships: A Graph, Not a Tree

When a project has multiple specs, those specs form a **directed graph**.
Any spec can import from any other spec. The relationships are many-to-many.
There is no fixed number of specs, no required hierarchy, and no mandatory
naming convention.

Each spec declares its imports. Each import creates an edge in the dependency
graph: "What does this spec depend on? What depends on it? Are there cycles?"

Cycles are allowed — real systems have them. A software spec might import
platform types, while the platform spec imports the software's health check
endpoints.

### Decomposition Patterns

Teams decompose specs however makes sense for their system. Common patterns:

- **By concern:** hardware, platform, software, quality, integration
- **By service:** one spec per microservice, with integration specs bridging them
- **By discipline:** engineering, security, compliance, UX, data governance
- **By lifecycle:** design spec, implementation spec, operations spec
- **Hybrid:** any combination, N specs deep, with any relationship topology

The standard doesn't care about your taxonomy. It cares about the graph.

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

These IMPORT declarations create edges in the spec graph. An agent traces
dependencies from a failing scenario through imports into other specs and
gets exactly the context it needs.

Real systems have complex import patterns. This is fine:

```
project-a/api imports from: project-a/auth, project-a/storage, org/platform/k8s
project-a/auth imports from: project-a/storage, org/security/threat-model
org/compliance/gdpr imports from: project-a/api (to validate data handling)
```

Compliance might cross-cut everything. Security might import from every service
to validate threat models. The graph is what it is.

---

## Spec Types

Not all specs produce running code.

**SYSTEM** (default) — produces a running system. All recommended sections apply.

**PATTERN** — defines a reusable architectural blueprint (e.g., a context stack
pattern). Other specs reference it via `USES PATTERN:` in Architecture.3.

**ASSET** — catalogs static resources and defines design constraints (e.g.,
a design system with color tokens and typography rules). ASSET specs with a
COMPONENT_CATALOG section serve as design system registrations for UI rendering
and verification. Other specs reference it via `USES ASSET:` in Architecture.3.

---

## Additional Element Types

Beyond the core element types defined in NLSPEC-TEMPLATE.md (RECORD, FUNCTION,
SCENARIO, RULE, TOKEN, ALGORITHM), specs can include domain-specific
element types. The standard is extensible — any named, typed block with
consistent structure is valid.

### Distributed System Elements

For distributed systems, specs can include three additional element types:

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

These elements follow the same pattern as any other element type — named,
typed, self-contained, cross-referenceable.

### UX Specifications

SYSTEM specs can include an optional UXSpec section that defines frontend
UX as structured, verifiable elements. SCENEs compose design system components
with layout, responsive rules, data bindings, and z-layer stacking.
TRANSITIONs define navigation between scenes. Visual verification checks
rendered output against the spec at three layers: structural DOM, design
token compliance, and visual regression. This makes UX development
spec-driven and machine-verifiable.

---

## Dependency Contracts

The Contracts section declares what a spec EXPORTS (constraints, invariants,
seed data, policies) and what it EXPECTS from dependencies. A resolver can
walk the graph and produce a deterministic seed manifest — the complete
initial state of the system. No manual configuration, no hidden defaults.

---

## Layer Composition — The 4-Layer Model

Specs can participate in a **4-layer composition model** where one
specification produces many realizations, each realization supports many
configurations, and each configuration serves many user profiles.

| Layer | Name | What It Defines |
|-------|------|-----------------|
| 1 | Specification | Contracts, interfaces, invariants — the possibility space |
| 2 | Realization | Platform-specific concretization of an L1 spec |
| 3 | Configuration | Domain or deployment tuning of an L2 realization |
| 4 | User Profile | Personalization constraints on an L3 configuration |

Layer composition is **opt-in** and **two-dimensional**. A spec declares its
layer in the header (`Layer:` field) and uses the `LayerContext` section to define:

- **Derivation + Horizontal Composition** (LayerContext.1) — which spec at the
  layer below this one derives from (vertical), and which specs at the same
  layer it composes with (horizontal). An OS realization is not one spec — it's
  kernel + filesystem + networking, all at L2, with COMPOSES WITH declarations
  defining their interfaces.
- **Layer Stack** (LayerContext.2) — the full tree of specs across all four layers,
  showing both vertical derivation and horizontal composition at each layer
- **Constraint Flow** (LayerContext.3) — how constraints propagate downward (default),
  optionally upward (when user preferences are non-negotiable), and laterally
  (between horizontally composed peers at the same layer)
- **Substitution Boundaries** (LayerContext.4) — what can be swapped at this layer
  without affecting other layers or horizontal peers
- **Cross-Layer References** (LayerContext.5) — `[Ln:spec-id:Section.x]` syntax for
  tracing constraints back to their source layer

Each layer is a clean substitution boundary. You swap an L2 realization (e.g.,
React → native iOS) without changing the L1 spec above or the L3 configurations
below. Within a layer, you swap an optional horizontal component (e.g., GPU driver)
without disturbing co-required peers (kernel, filesystem). You swap an L4 user
profile without touching the L3 configuration it personalizes.

Specs without a Layer declaration are standalone — they work exactly as before.
The layer model adds composition semantics on top of the existing template
without changing it.

---

## Test Composition Across Specs

When a project has multiple specs, tests follow the same graph structure as the specs
themselves. Each spec produces its own test suite. Cross-spec testing uses mocks at
the IMPORT boundary.

**Rule:** A spec's tests only test that spec's FUNCTIONs. IMPORTed functions from
other specs are mocked. This preserves test isolation and prevents N² test explosion.

**Mock boundary:** When Spec A imports `validate_token` from Spec B, Spec A's tests
mock `validate_token` at the import boundary. Spec B's tests are responsible for
testing `validate_token` directly. Deterministic validation (see TestGeneration.5 in
the template) ensures that mock targets reference real modules — this prevents
hallucinated module paths regardless of how many specs are composed.

**Data contracts across specs:** When Spec A uses a RECORD defined in Spec B, the
CONSTRAINT declarations on that RECORD travel with the import. Spec A's tests must
respect those constraints when constructing test data — tooling can verify this by
matching test fixture values against CONSTRAINT declarations from the source spec.

**Substrate-scoped tests:** Each substrate (e.g., web, iOS, Android) gets its own
test suite derived from its own L2 realization. Tests do not cross substrate boundaries.
The L1 spec's tests validate contracts; each L2 test suite validates that the
realization honors those contracts in its platform-specific way.

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
| **Temporal** | Version history — the same system evolving over time | v1, v2, v3 |
| **Feature** | Feature flag variants — different capabilities enabled | with-payments, without-payments |

A substrate is simply a branching point in the DERIVES FROM graph where one
spec produces multiple children at the next layer (or the same layer, for
L1→L1 branching).

### Substrate Detection

Substrates are not declared explicitly. They are **inferred** from the
DERIVES FROM graph:

1. Walk the DERIVES FROM edges across all specs
2. When a spec at layer N has multiple children at layer N+1 (or N), that's a
   branching point — each child is the root of a substrate
3. Follow each branch down through its L3/L4 stack to get the full substrate
4. Name substrates from the branching spec's identifier (e.g., the "ios" substrate
   is named from the L2 spec `l2-ios-realization`)

No new metadata is needed. The data already exists in the spec headers.

### Cross-Substrate Interaction

Substrates are independent by default. The iOS L3 config does not affect the
Android L3 config. However, some projects need cross-substrate contracts:

- A shared API gateway that all platform substrates call
- A compliance spec that cross-cuts all substrates
- A shared component library used by web and mobile substrates

Cross-substrate interaction is handled through the existing COMPOSES WITH and
IMPORT mechanisms — a spec in one substrate can IMPORT from a spec in another.

---

## Summary

nlspec specs are markdown files. A single spec is self-contained — copy the
template, fill it in, hand it to an agent. Multiple specs form a directed
graph via IMPORT declarations. The 4-layer model adds vertical derivation
and horizontal composition. Substrates add branching for platform, version,
and feature variants. All composition is opt-in. The template is the same
at every scale.
