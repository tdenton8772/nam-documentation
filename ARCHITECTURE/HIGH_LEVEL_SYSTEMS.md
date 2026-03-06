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

NAM consists of eleven major conceptual components:

1. **Input Interfaces**
2. **NLP Pipeline**
3. **Ontology Classifier**
4. **Encoder Heads**
5. **Address Builder**
6. **Storage Layer**
7. **Query Planner**
8. **Execution Engine**
9. **Access Control Layer**
10. **Lifecycle Manager**
11. **Entity Cache**

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

## 2. NLP Pipeline

The NLP pipeline is a **dedicated linguistic analysis stage** that sits between input interfaces and encoder heads.

It performs:

* Tokenization
* Part-of-speech tagging
* Lemmatization
* Named entity recognition
* Dependency parsing

### Key architectural properties

The NLP pipeline is:

* **Rule-based** — no neural models, no learned weights, no GPU required
* **Deterministic** — identical input always produces identical linguistic output
* **Fast** — throughput exceeds 13,000 parses per second on a single core
* **Mandatory** — every record passes through NLP before reaching encoder heads

The NLP pipeline exists as a separate component because:

* All encoder heads depend on its output
* Linguistic analysis is expensive and should happen exactly once per record
* Deterministic parsing is the foundation on which deterministic addressing is built

→ See: [Pipeline Architecture](PIPELINE_ARCHITECTURE.md)

---

## 3. Ontology Classifier

The ontology classifier determines the **semantic type** of a record.

It assigns one or more ontology labels from a controlled vocabulary:

* **Tier 1** (most general): person, group, organization, location, place, object, concept
* **Tier 2** (temporal): time, event, action, process, sequence
* **Tier 3** (state): state, change, cause, effect, condition
* **Tier 4** (attributes): attribute, quantity, relationship, ownership, identifier, record, document, question, goal

### Key architectural properties

The ontology classifier is:

* **Separate from encoder heads** — it runs after NLP but before heads, providing type context that informs entity resolution and address partitioning
* **Deterministic** — classification is based on linguistic features, not probabilistic models
* **Exhaustive** — every record receives at least one ontology label; there is no "unknown" or "other" fallback

The ontology classifier was recently hardened to eliminate a catch-all "OTHER" category that was degrading query precision. Every record now maps to a real semantic type, with "concept" serving as the broadest valid category.

---

## 4. Encoder Heads

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

NAM currently uses five encoder heads:

* **Entity** — extracts named entities, resolves them to canonical hash-based identifiers via a shared entity store
* **Attribute** — extracts adjectives and descriptors linked to entities via dependency parsing
* **Affordance** — extracts verbs and actions, with synonym folding to canonical forms
* **Context** — extracts situational and temporal signals at the sentence level
* **Ontology** — classifies the semantic type of the record (see Ontology Classifier above)

The **entity head** is the most resource-intensive (due to entity resolution I/O) and is therefore parallelized with multiple workers per ingest replica. Other heads run as single instances per replica.

---

## 5. Address Builder

The address builder composes encoder outputs into **addresses**.

It:

* Combines ontology, entity anchors, and axis coordinates
* Scopes address construction to entities — each semantic component is associated only with the entities it actually modifies (entity-anchored bundling)
* Applies namespace and partition rules
* Produces one or more deterministic addresses per record

Key idea:

> Address construction is the *only* place where semantic meaning becomes spatial structure.

Entity-anchored bundling ensures that addresses reflect the linguistic structure of the input, not a combinatorial explosion of all possible component combinations. This bounds address count per record to what the text actually expresses.

Coordinate values are encoded through **LCA (Learned Codec for Addressing)** — a learned byte-code representation that provides geometric structure. Semantically similar strings map to nearby codes, enabling neighborhood-based fan-out at query time.

Once an address is built, it is immutable.

The address builder is:

* Stateless
* Deterministic
* Shared by ingest and query paths

→ See: [Addressing Model](ADDRESSING_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md)

---

## 6. Storage Layer

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

### Current implementation

The current storage layer is a **distributed key-value store backed by object storage (S3-compatible)**:

* Data is written locally, then asynchronously replicated to object storage
* On restart, only metadata is downloaded; data files are fetched on demand
* This enables fast cold starts (seconds, not minutes) and efficient resource use
* All address state is rebuildable from source data — the address index is derived, never authoritative

→ See: [Data Persistence](DATA_PERSISTENCE.md) for the full persistence model

Critical property:

> Storage is address-aware, not content-aware.

---

## 7. Query Planner

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

The planner also:

* Captures **ontology hints** from the bundler (what types were observed in the query text)
* Reorders ontology tiers so hinted tiers are probed first
* Caps total planned addresses to a configurable budget

Planning is structural, not statistical.

---

## 8. Execution Engine

The execution engine carries out the plan.

It:

* Probes the storage layer at specific addresses
* Navigates points, lines, and planes in address space
* Deduplicates records
* Returns stable, deterministic results

Execution uses **progressive fan-out**:

* Addresses are probed in specificity order (most specific first)
* Ontology tiers are probed in hint-prioritized order
* If a tier yields enough matches (above a satisfaction threshold), remaining tiers are skipped
* Total probes and fetches are budget-capped

Execution is:

* Bounded
* Ordered
* Observable
* Early-terminating when satisfaction is reached

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

## 9. Access Control Layer

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

## 10. Lifecycle Manager

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

## 11. Entity Cache

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

→ See also: [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Data Persistence](DATA_PERSISTENCE.md) | [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md)

