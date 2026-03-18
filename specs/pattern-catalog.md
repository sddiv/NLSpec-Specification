# Pattern Catalog — Prior Art Reference

> **Type:** CATALOG
> **Version:** 0.1.0
> **Author:** Divyendu Deepak Singh
> **Date:** Feb 2026
> **Status:** Living Document

---

## How to Use This Catalog

This is a reference list of well-known patterns the agent already understands from
training. When a SYSTEM spec declares `USES PATTERN: {name}` without a FROM clause,
the agent looks up the pattern here for context on when to use it and any constraints,
then implements using its training knowledge.

For novel patterns the agent does NOT know, create a full TYPE: PATTERN spec instead.

---

## System-Level Patterns

```
PATTERN: Caching
  level       : system
  description : Store frequently accessed data in fast storage to reduce latency
  when        : Read-heavy workloads, expensive computations, external API calls
  constraints : Cache invalidation strategy must be specified in customization
  variants    : Write-through, write-back, write-around, read-through

PATTERN: Sharding
  level       : system
  description : Partition data across multiple stores by a shard key
  when        : Single store cannot handle volume; need horizontal scalability
  constraints : All data access must include shard key; cross-shard queries are expensive

PATTERN: CQRS
  level       : system
  description : Separate read and write models for a data store
  when        : Read and write patterns are fundamentally different
  constraints : Eventual consistency between read and write models must be acceptable

PATTERN: EventSourcing
  level       : system
  description : Store state changes as an append-only sequence of events
  when        : Need full audit trail; need to reconstruct state at any point in time
  constraints : Event schema must be versioned; replay must be idempotent

PATTERN: CircuitBreaker
  level       : system
  description : Prevent cascading failures by failing fast when a dependency is unhealthy
  when        : Calling external services or unreliable dependencies
  constraints : Failure threshold, recovery timeout, and half-open behavior must be specified

PATTERN: Saga
  level       : system
  description : Manage distributed transactions via a sequence of local transactions with compensating actions
  when        : Multi-service operations that need atomicity without distributed locks
  constraints : Every step must have a compensating action; orchestration vs choreography must be chosen

PATTERN: Sidecar
  level       : system
  description : Deploy helper functionality alongside the main service as a separate process
  when        : Cross-cutting concerns (logging, auth, metrics) across multiple services
  constraints : Communication between sidecar and main must be local (localhost/unix socket)

PATTERN: Bulkhead
  level       : system
  description : Isolate components so failure in one does not cascade to others
  when        : Multiple independent workloads share resources
  constraints : Resource pools must be sized; overflow behavior must be specified

PATTERN: BackPressure
  level       : system
  description : Producer slows down when consumer cannot keep up
  when        : Unbounded producers, bounded consumers, queue-based systems
  constraints : Signaling mechanism and overflow strategy must be specified

PATTERN: LeaderElection
  level       : system
  description : Select one node in a cluster to coordinate work
  when        : Distributed systems needing single-writer or coordinator
  constraints : Election mechanism, failure detection, and re-election behavior must be specified
```

---

## Design-Level Patterns

```
PATTERN: Repository
  level       : design
  description : Mediate between domain and data mapping using a collection-like interface
  when        : Need to decouple business logic from data access
  constraints : All data access goes through repository interface; no direct DB calls from business logic

PATTERN: Factory
  level       : design
  description : Create objects without exposing instantiation logic
  when        : Object creation is complex or varies by context
  constraints : Factory must be the sole creator of the target type

PATTERN: Observer
  level       : design
  description : Notify dependents automatically when state changes
  when        : One-to-many dependency between objects; event notification
  constraints : Observer registration and deregistration must be explicit

PATTERN: Strategy
  level       : design
  description : Define a family of algorithms and make them interchangeable
  when        : Multiple algorithms for the same task; need to switch at runtime
  constraints : All strategies must implement the same interface

PATTERN: DependencyInjection
  level       : design
  description : Supply dependencies from outside rather than creating them internally
  when        : Need testability, flexibility, and loose coupling
  constraints : Dependencies are injected via constructor; no service locator

PATTERN: Middleware
  level       : design
  description : Chain of processing steps that each transform or inspect a request
  when        : HTTP servers, message processing, plugin systems
  constraints : Middleware must be composable and order-independent where possible

PATTERN: Builder
  level       : design
  description : Construct complex objects step by step
  when        : Objects with many optional parameters; immutable objects
  constraints : Builder must validate before build; build must return Result/Option

PATTERN: Adapter
  level       : design
  description : Convert one interface to another that clients expect
  when        : Integrating with external systems or legacy code
  constraints : Adapter must not leak abstraction of the adapted interface
```

---

## Concurrency Patterns

```
PATTERN: ActorModel
  level       : concurrency
  description : Encapsulate state in actors that communicate via message passing
  when        : Need isolation, fault tolerance, and concurrent state management
  constraints : No shared mutable state; all communication via messages

PATTERN: WorkStealing
  level       : concurrency
  description : Idle workers steal tasks from busy workers' queues
  when        : Load balancing across workers with variable task duration
  constraints : Task queues must be thread-safe; stolen tasks must be independent

PATTERN: FanOutFanIn
  level       : concurrency
  description : Distribute work across workers, then collect results
  when        : Embarrassingly parallel tasks; map-reduce style processing
  constraints : Result aggregation strategy must be specified

PATTERN: ProducerConsumer
  level       : concurrency
  description : Decouple production from consumption via a bounded queue
  when        : Rate mismatch between producer and consumer
  constraints : Queue capacity and overflow strategy must be specified
```

---

## Data Patterns

```
PATTERN: WriteAheadLog
  level       : data
  description : Write changes to a log before applying to the main data structure
  when        : Need crash recovery and durability guarantees
  constraints : Log must be fsync'd before acknowledging write

PATTERN: LSMTree
  level       : data
  description : Write-optimized data structure using sorted runs with periodic compaction
  when        : Write-heavy workloads; time-series data
  constraints : Compaction strategy and level count must be specified

PATTERN: BloomFilter
  level       : data
  description : Probabilistic data structure for set membership testing
  when        : Need fast negative lookups; can tolerate false positives
  constraints : False positive rate and expected element count must be specified

PATTERN: ConsistentHashing
  level       : data
  description : Distribute data across nodes with minimal redistribution on changes
  when        : Distributed caches, partitioned databases
  constraints : Virtual node count and rebalancing strategy must be specified
```

---

## Composition Patterns

```
PATTERN: LayerComposition
  level       : composition
  description : Decompose a system into 4 layers: Specification → Realization →
                Configuration → User Profile. Each layer is a clean substitution
                boundary. One L1 spec produces many L2 realizations, each L2
                supports many L3 configurations, each L3 serves many L4 profiles.
  when        : Multi-platform products, configurable systems, multi-tenant with
                per-user preferences, systems that need clean technology migration paths
  constraints : Each layer must define DERIVES FROM (parent), LAYER STACK (tree),
                CONSTRAINT FLOW (downward + optional upward), and SUBSTITUTION
                BOUNDARY (what's swappable). Constraints flow downward by default.
                Upward flow requires explicit declaration and is blocked by NEVER overrides.
  variants    : Full 4-layer (spec→realize→configure→profile),
                3-layer (spec→realize→configure, no per-user profiles),
                2-layer (spec→realize, no configuration layer)
```

---

## How to Add Patterns

**Prior art:** Add an entry to this catalog with name, level, description, when/constraints.

**Novel pattern:** Create a full TYPE: PATTERN spec using NLSPEC-TEMPLATE.md. The
catalog is for patterns the agent already knows. Novel specs are for patterns that
need to be taught.
