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

NAM consists of six major conceptual components:

1. **Input Interfaces**
2. **Encoder Heads**
3. **Address Builder**
4. **Storage Layer**
5. **Query Planner**
6. **Execution Engine**

Each component has a single responsibility and is **independently evolvable**.

---

## 1. Input Interfaces

Input interfaces are how data enters NAM.

Examples:

* Streaming events
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
* Applies namespace and partition rules
* Produces one or more deterministic addresses per record

Key idea:

> Address construction is the *only* place where semantic meaning becomes spatial structure.

Once an address is built, it is immutable.

The address builder is:

* Stateless
* Deterministic
* Shared by ingest and query paths

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

## Symmetry Between Ingest and Query

A core architectural principle of NAM is **symmetry**:

| Ingest                          | Query                         |
| ------------------------------- | ----------------------------- |
| Encoder heads project meaning   | Encoder heads project intent  |
| Address builder structures data | Planner structures navigation |
| Storage persists geometry       | Execution navigates geometry  |

This symmetry enables determinism, explainability, and rebuildability.

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
* Predictable deployment behavior

These guarantees are more important than individual optimizations.


## Why this architecture matters

This separation of concerns ensures that:

* Customer-specific semantics can be trained safely
* Storage choices do not constrain meaning
* Queries remain explainable and reproducible
* Systems can evolve without semantic drift

NAM is designed to be **operationally boring** — and semantically powerful.

