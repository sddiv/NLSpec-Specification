# nlspec — A Specification for Organizational AI

Write specs, not code. Hand the spec to an agent. Get a working implementation.

---

## What Is This?

A standard for writing natural language specifications that AI coding agents can
consume to produce high-fidelity implementations. The spec defines your system's
data model, functions, API, scenarios, and constraints in a structured, machine-readable
markdown format.

This is not a document format. It is not a workflow tool. It is a formalization of
how to write specs so that generative AI produces correct output — not plausible output.

## What's In This Repo?

```
nlspec/
├── README.md                    # You're reading it
├── LICENSE                      # CC-BY-4.0
├── NLSPEC-SYSTEM.md             # How specs compose: layers, substrates, graph relationships
└── NLSPEC-TEMPLATE.md           # The spec template (SYSTEM | PATTERN | ASSET types)
```

| File | Purpose |
|------|---------|
| `NLSPEC-TEMPLATE.md` | The blank spec format. Copy it, fill it in. Defines all element types, section structures, and the generation pipeline pattern. |
| `NLSPEC-SYSTEM.md` | How specs compose and scale. Layer model, substrates, graph relationships. |

## Quick Start

### 1. Start a new project

```bash
mkdir my-project && cd my-project
mkdir specs
cp path/to/nlspec/NLSPEC-TEMPLATE.md specs/my-project-spec.md
```

### 2. Fill in the spec

Open `specs/my-project-spec.md` and replace every `{placeholder}` with your actual
specification. The declared sections guide you through everything from abstract to deployment.

### 3. Hand it to the agent

```
claude-code> Implement the spec defined in specs/my-project-spec.md
```

The agent reads the spec for what to build. The spec is self-contained — it defines
the system's data model, functions, API, scenarios, and everything else the agent needs.

### 4. Fix bugs with context, not rewrites

```
claude-code> SCENARIO 7 is failing. Fix it.
```

The agent identifies the affected section, reads only what it needs, fixes the code,
and validates against scenarios.

## The Two Core Structures

### Named, Typed Blocks

Every piece of knowledge in a spec is a named, typed block: RECORD, FUNCTION, SCENARIO,
RULE, TOKEN, ALGORITHM, ENDPOINT, PIPELINE. Each block has a unique name, a declared type,
and self-contained content. This makes specs machine-addressable — a generator resolves
a named reference and gets verbatim content, not a summary.

The type vocabulary is open. A design system spec uses RULE and TOKEN blocks. A backend
API spec uses RECORD and FUNCTION blocks. An infrastructure spec uses ALGORITHM blocks.
The structural contract is fixed: every block is addressable by name and extractable
without surrounding context.

### Closed-Context Generation

Named blocks enable precise context slicing for code generation. Instead of feeding
an entire spec to a generator and hoping it extracts the right details, a build tool
can resolve specific block references to get verbatim content. The `depends` on named
blocks creates a closed context — everything the generator needs is explicitly listed.
Nothing ambient, nothing implied.

This eliminates the summarization problem: a large spec summarized into a generation
prompt loses the details that matter. Named block references ensure the generator sees
exact RULE constraints, TOKEN values, and RECORD fields as the spec author wrote them.

## Recommended Sections

| Section | What It Defines |
|---------|----------------|
| Abstract | One paragraph: what this spec defines |
| Problem | Current state → deficiency → target → key insight |
| Architecture | ASCII diagram, component inventory, data flows, patterns, assets |
| DataModel | RECORD, ENUM, ALIAS definitions with invariants and data contracts |
| Functions | FUNCTION specs: inputs, outputs, behavior, errors, param/return constraints |
| API | HTTP endpoints, gRPC services, CLI commands, MCP tools |
| Errors | Complete error hierarchy with handling rules |
| Config | Every knob: type, default, env var, validation |
| Scenarios | End-to-end behavioral tests (the holdout set) |
| TestGeneration | How tests derive from spec contracts, naming rules, deterministic validation |
| Dependencies | External systems, libraries, asset packs with pinned versions |
| FileStructure | Expected directory layout |
| BuildAndRun | Exact commands to build, test, run, verify |
| Boundaries | Explicit non-goals (prevents scope creep) |
| Contracts | EXPORTS, EXPECTS, conflict resolution |
| LayerContext | (Optional) Layer composition: derivation, stack, constraint flow |

Sections are declared per-spec. Not all sections apply to every spec. PATTERN and ASSET
specs use different section sets. See NLSPEC-TEMPLATE.md for details.

## Key Concepts

**Specs are the source of truth, not code.** Code is a derived artifact. When the
spec and code disagree, the spec is correct.

**Scenarios are immutable.** The agent cannot modify scenarios to make tests pass.
If a scenario fails, the implementation is wrong.

**Named blocks are the atomic unit.** Every RECORD, FUNCTION, RULE, TOKEN, and SCENARIO
is a named, extractable block. Agents and tooling resolve blocks by name to get
verbatim content.

**Named blocks enable closed generation contexts.** A build tool resolves block
references to get verbatim spec content. This eliminates summarization and prevents
hallucination from training data.

**Data contracts on fields.** RECORD fields and FUNCTION parameters can declare
CONSTRAINT blocks (RANGE, NOT_EMPTY, UNIQUE, PATTERN, IMMUTABLE, etc.) — like Avro
schemas but in natural language specs. Constraints define the valid value space for
each field, enabling tooling to compute test sufficiency deterministically: does the
test suite cover the constraint space? The answer is computable from the spec alone,
without generating code or running tests.

**Tests derive from spec contracts.** The TestGeneration section defines how every
spec element produces testable assertions. Every FUNCTION gets a happy path test,
every THROWS gets an error path test, every CONSTRAINT gets boundary tests. Tests
are spec artifacts — they trace back to named blocks and are validated against the
spec structure deterministically. This makes test coverage a property of the spec,
not a property of the implementation.

**Spec types.** SYSTEM specs produce running code (the default). PATTERN specs
define reusable architectural blueprints. ASSET specs catalog static resources
and define style guides (RULEs and TOKENs).

**Specs compose as a graph.** Any spec can import from any other spec. Relationships
are many-to-many. No fixed hierarchy. No mandatory naming.

**Layer composition (optional).** Specs can participate in a 4-layer model:
Specification → Realization → Configuration → User Profile. Vertical derivation
across layers, horizontal composition within a layer. See `NLSPEC-SYSTEM.md`.

## How nlspec Differs from Existing Tools

The term "nlspec" was coined by [StrongDM](https://factory.strongdm.ai/) to mean
a human-readable spec directly usable by coding agents. We formalize and extend that
idea into a complete specification standard.

| | StrongDM nlspec | GitHub Spec Kit | OpenSpec / BMAD / Kiro | **nlspec (this)** |
|---|---|---|---|---|
| **What it is** | 3 markdown files for one product | CLI + workflow scaffold | Workflow tools for SDD | A specification standard |
| **Spec format** | Implicit (RECORD, FUNCTION, etc.) | Freeform markdown | Freeform markdown | Formalized template with typed elements (SYSTEM, PATTERN, ASSET) |
| **Element types** | Used but not defined | None | None | RECORD, FUNCTION, SCENARIO, RULE, TOKEN, ALGORITHM — first-class, named, extractable |
| **Data contracts** | None | None | None | CONSTRAINT blocks on fields (RANGE, NOT_EMPTY, PATTERN, etc.) — Avro-like schemas in NL specs |
| **Test derivation** | None | None | None | Tests derived from spec contracts with deterministic coverage validation |
| **Artifact generation** | None | None | None | Closed-context generation: named block dependencies + verification contracts |
| **Cross-references** | None | None | None | USES, USED BY, SEC tags — agents walk edges for context slicing |
| **Composition** | None | None | None | 4-layer derivation + horizontal composition + substrate branching |
| **Self-describing** | No | No | No | Yes — the spec for nlspec is itself an nlspec |

## License

CC-BY-4.0 — use it, adapt it, share it. Just give credit.

## Credits

- Term "nlspec" coined by [StrongDM's Software Factory](https://factory.strongdm.ai/)
  and their [Attractor](https://github.com/strongdm/attractor)
- Created by Divyendu Deepak Singh
