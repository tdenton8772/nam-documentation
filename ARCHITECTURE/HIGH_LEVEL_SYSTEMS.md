# High-Level System Architecture

*A conceptual view of NAM*

This document describes NAM at the **system level** — how information flows through it, how responsibilities are separated, and how the major components interact.

This is **not** an implementation guide.
No code, no APIs, no infrastructure assumptions.

---

## System Overview

NAM is a **semantic memory system** that:

1. Accepts unstructured inputs (text, events, records)
2. Projects them into a deterministic semantic address space
3. Stores them in a navigable, partitioned structure
4. Answers queries by **geometric navigation**, not similarity or ranking

At a high level, NAM has **two symmetric paths**:

* **Ingest path** — how knowledge enters memory
* **Query path** — how knowledge is navigated and retrieved

Both paths rely on the **same semantic primitives**.

---

## Core Components (Conceptual)

NAM consists of nine major conceptual components:

1. **Input Interfaces**
2. **Encoder Heads**
3. **Address Builder**
4. **Storage Layer**
5. **Query Planner**
6. **Execution Engine**
7. **Access Control Layer**
8. **Lifecycle Manager**
9. **Entity Cache**

Each component has a single responsibility and is **independently evolvable**.

---

## 1. Input Interfaces

Input interfaces are how data enters NAM.

Examples:

* CDC (change data capture) streams
* Batch records
* Documents
* Questions or prompts

Key properties:

* Inputs are opaque text or payloads
* NAM does not assume structure
* No schema is required at input time

The interface layer does **not** interpret meaning.

---

## 2. Encoder Heads

Encoder heads are **pluggable, deterministic semantic projectors**.

They:

* Inspect input text
* Extract specific classes of signals (entities, attributes, context, ontology, affordance)
* Emit symbolic coordinates for address construction

### Key architectural properties

Encoder heads are:

* **Independently managed**
* **Independently trained**
* **Independently deployable**

A system may:

* Add new heads
* Remove unused heads
* Retrain a single head without affecting others
* Maintain different head configurations per customer or domain

### Training model

Encoder heads are trained **offline**, often using:

* Customer-specific corpora
* Domain-specific language
* Field-generated data (logs, documents, events)

Once trained:

* A head’s behavior is **fixed at runtime**
* Its outputs are deterministic
* It does not adapt or learn during queries

This allows NAM to be **domain-aware without being non-deterministic**.

Encoder heads exist to **translate language into structure**, not to reason or retrieve.

---

## 3. Address Builder

The address builder composes encoder outputs into **addresses**.

It:

* Combines ontology, entity anchors, and axis coordinates
* Scopes address construction to entities — each semantic component is associated only with the entities it actually modifies (entity-anchored bundling)
* Applies namespace and partition rules
* Produces one or more deterministic addresses per record

Key idea:

> Address construction is the *only* place where semantic meaning becomes spatial structure.

Entity-anchored bundling ensures that addresses reflect the linguistic structure of the input, not a combinatorial explosion of all possible component combinations. This bounds address count per record to what the text actually expresses.

Once an address is built, it is immutable.

The address builder is:

* Stateless
* Deterministic
* Shared by ingest and query paths

→ See: [Addressing Model](ADDRESSING_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md)

---

## 4. Storage Layer

The storage layer persists addresses and record references.

Conceptually, it provides:

* Keyed lookup by address
* Deterministic partitioning
* Read-only navigation during queries

### Swappable by design

The storage layer is intentionally **decoupled from semantics**.

It:

* Does not interpret meaning
* Does not execute logic
* Does not inspect payloads

As a result, storage backends are **fully swappable**, including:

* Local file-based stores
* Embedded databases
* Distributed key-value systems
* Cloud-managed storage services

Different deployments may choose different storage implementations **without changing encoder behavior, addressing logic, or query planning**.

Critical property:

> Storage is address-aware, not content-aware.

---

## 5. Query Planner

The query planner translates a user query into an **execution plan**.

It:

* Detects affordances (what kind of question is being asked)
* Selects ontology tiers
* Determines which address families to probe
* Controls widening and fallback behavior

The planner does **not**:

* Rank results
* Score similarity
* Learn from feedback

Planning is structural, not statistical.

---

## 6. Execution Engine

The execution engine carries out the plan.

It:

* Probes the storage layer at specific addresses
* Navigates points, lines, and planes in address space
* Deduplicates records
* Returns stable, deterministic results

Execution is:

* Bounded
* Ordered
* Observable

---

## End-to-End Flow (Ingest)

1. Input arrives
2. Encoder heads extract semantic signals
3. Address builder constructs addresses
4. Addresses and record references are stored

No additional inference occurs later.

---

## End-to-End Flow (Query)

1. Query arrives
2. Encoder heads project intent into semantic signals
3. Planner builds an execution plan
4. Execution engine navigates stored addresses
5. Matching records are returned

Runtime behavior never mutates stored state.

---

## 7. Access Control Layer

The access control layer mediates all external access to NAM.

It:

* Authenticates clients before allowing access to any internal service
* Validates signed tokens on every request
* Enforces role-based access (e.g. admin vs user)
* Rate-limits sensitive operations
* Produces structured audit logs for every access

### Key architectural properties

The access control layer is:

* **Mandatory at external boundaries** — no unauthenticated path exists from outside the cluster to internal services
* **Optional at internal boundaries** — services within the cluster trust network isolation
* **Stateless per request** — authentication decisions are made from the token alone, no session state is maintained
* **Independent of the semantic pipeline** — authentication does not influence addressing, encoding, or retrieval

The access control layer can be deployed as:

* Middleware within the query service
* A standalone proxy at the network edge
* Both, for defense in depth

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## 8. Lifecycle Manager

The lifecycle manager handles cluster-level operations that must happen before — and during — normal system operation.

It:

* Waits for the data layer to become available
* Coordinates node membership and data distribution
* Verifies that expected storage structures exist
* Creates initial administrative credentials
* Signals readiness to downstream components
* Monitors cluster health on a recurring basis
* Re-joins dropped nodes automatically

### Key architectural properties

The lifecycle manager is:

* **A single instance** — only one runs per cluster (no leader election needed)
* **Idempotent** — repeated execution of any lifecycle action produces the same result
* **Gating** — ingest and query services do not start until the lifecycle manager confirms readiness
* **Separate from the semantic pipeline** — lifecycle management does not process records, build addresses, or serve queries

The lifecycle manager exists because distributed systems require explicit coordination during bootstrap and recovery. Without it, services start against uninitialized storage, miss configuration, or race during node joins.

---

## 9. Entity Cache

The entity cache provides **node-level shared memory** for entity resolution.

It:

* Maintains a local copy of entity surface form mappings
* Serves lookups in microseconds via memory-mapped reads
* Is written by a dedicated per-node process that follows the entity store's change stream
* Is shared across all entity resolution workers on a node (read-only)

### Key architectural properties

The entity cache is:

* **Per-node, not per-process** — a single writer serves all consumers on the same node
* **Eventually consistent** — follows the authoritative store's change stream, not a snapshot
* **Read-only from consumers** — workers never write to the cache directly
* **Disposable** — if corrupted or stale, the cache rebuilds automatically from the change stream

The entity cache exists because entity resolution is the most I/O-intensive operation in the encoding pipeline. Without node-level caching, every entity lookup crosses the network to the data service. The cache converts network I/O into memory reads.

---

## Symmetry Between Ingest and Query

A core architectural principle of NAM is **symmetry**:

| Ingest                          | Query                         |
| ------------------------------- | ----------------------------- |
| Encoder heads project meaning   | Encoder heads project intent  |
| Address builder structures data | Planner structures navigation |
| Storage persists geometry       | Execution navigates geometry  |

This symmetry enables determinism, explainability, and rebuildability.

The access control layer, lifecycle manager, and entity cache are **cross-cutting** — they serve both paths and operate independently of semantic logic.

---

## What NAM Is *Not*

At the system level, NAM is explicitly **not**:

* A vector database
* A knowledge graph engine
* A search index
* A rules engine
* A runtime learning system

NAM may integrate with these — but it is not them.

---

## Architectural Guarantees

NAM provides the following guarantees by design:

* Deterministic execution
* Pluggable, independently trained encoder heads
* Swappable storage backends
* Offline semantic training
* Online-only navigation
* Authenticated access at every external boundary
* Automated lifecycle management and readiness gating
* Predictable deployment behavior

These guarantees are more important than individual optimizations.


## Why this architecture matters

This separation of concerns ensures that:

* Customer-specific semantics can be trained safely
* Storage choices do not constrain meaning
* Queries remain explainable and reproducible
* Systems can evolve without semantic drift

NAM is designed to be **operationally boring** — and semantically powerful.

→ See also: [Geometric Retrieval](GEOMETRIC_RETREIVAL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md)

