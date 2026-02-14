# Ingestion Model

## How Raw Information Becomes Addressable Meaning in NAM

In NAM, ingestion is not a preprocessing step, an ETL pipeline, or an indexing phase in the traditional sense.
It is the act of **projecting raw information into a deterministic semantic space**.

Ingestion and query are symmetric operations. Both use the same encoder heads, the same addressing rules, and the same geometric model. The only difference is **direction**:

* Ingestion **writes** addresses
* Query **probes** addresses

This symmetry is what makes geometric retrieval possible.

---

## Core Principle: Ingest Defines the Search Space

NAM does not “learn” what a record means at query time.

All meaning that can ever be retrieved must first be **materialized as addresses during ingestion**.

If a query fails to retrieve a record, it is not because:

* The model “missed” it
* Similarity was too low
* The question was phrased incorrectly

It is because the relevant addresses were never emitted at ingest.

This is a deliberate design choice.

---

## What Ingestion Is (and Is Not)

### Ingestion **is**:

* Deterministic address construction
* Semantic projection via encoder heads
* Lossy by design (meaning is discretized)
* Repeatable and auditable
* Domain-trainable via head artifacts

### Ingestion **is not**:

* Feature extraction for a model
* Embedding generation
* Schema enforcement
* Ranking or scoring
* Adaptive or online learning

NAM does not “optimize” ingest. It **commits** it.

---

## Encoder Heads at Ingest Time

During ingestion, raw input (text, events, records, documents) is passed through a set of **encoder heads**.

Each head is responsible for a specific semantic dimension, for example:

* Entity
* Ontology
* Context
* Attribute
* Affordance (when applicable)

Each head may emit:

* One address
* Many addresses
* Or no address at all

All emitted addresses are treated as **equally valid**.

There is no concept of “primary” or “secondary” meaning at ingest.

---

## Entity-Anchored Address Construction

Address construction is entity-anchored. Rather than crossing every semantic component with every other, the system uses dependency parsing to determine which components *actually belong to* which entities.

This means:

* Each entity in the text gets its own bundle of associated components
* Attributes, actions, and context are scoped to the entities they modify
* The address count per record is bounded by the structure of the text, not by combinatorial explosion

Entity-anchored bundling preserves linguistic structure. The system knows that "angry" modifies "technician," not "pump" — a syntactic fact extracted deterministically from the parse tree.

→ See: [Addressing Model](ADDRESSING_MODEL.md)

---

## Address Emission and Multiplicity

A single record may produce many addresses.

This is expected.

For example:

* A biographical paragraph may emit multiple entity-scoped address bundles
* A procedural description may emit several ontology paths
* A vague statement may emit only coarse or partially populated addresses

NAM does not collapse or deduplicate meaning during ingestion.
It preserves **semantic breadth**, not precision.

Precision is handled at query time through geometric narrowing.

---

## Axes and `__null__`

Each address in NAM exists in a multi-axis space (for example, X, Y, Z).

When an encoder head cannot confidently populate an axis:

* The axis is explicitly set to `__null__`

`__null__` does **not** mean “unknown” or “missing”.

It means:

> “This record intentionally does not constrain this axis.”

This allows records to exist as:

* Points (fully specified)
* Lines (one unconstrained axis)
* Planes (multiple unconstrained axes)
* Volumes (highly general meaning)

This is essential for:

* Wildcard queries
* Exploratory retrieval
* Progressive narrowing

---

## Ingest-Time Learning

NAM's ingest pipeline is read-only with respect to navigation rules, but it does learn continuously in a constrained way:

* New entities are registered, and surface forms are mapped to canonical entries
* Unknown vocabulary (verbs, adjectives) is auto-classified based on context
* Curated seed vocabulary takes priority over discovered terms

This learning **does not affect query behavior retroactively**. It only enriches the vocabulary available for future entity resolution and address construction.

→ See: [Design Principles — Learning](../PHILOSOPHY/DESIGN_PRINCIPLES.md#4-field-based-training-not-runtime-learning)

---

## Determinism as a Contract

Given the same input and the same encoder artifacts:

* Ingestion will always emit the same addresses
* In the same order
* Into the same partitions
* With the same keys

There is no randomness.
There are no thresholds that drift.
There is no mutation to navigation rules during ingest.

This guarantees:

* Rebuildability
* Auditability
* Trust in retrieval behavior

---

## Storage Is Address-First

Ingestion does not store records “by content”.

It stores **address → record references**.

The record payload itself is opaque to the addressing system and may live:

* In memory
* In files
* In object storage
* In external systems

NAM’s storage layer only cares about:

* The address
* The record identifier
* The pointer to the payload

This separation is intentional and allows storage backends to be swapped without changing semantics.

---

## No Query-Time Repair

NAM does not “fix” ingest mistakes at query time.

If an address was not emitted:

* It cannot be retrieved later
* No fallback logic will infer it
* No similarity search will approximate it

This forces correctness upstream and makes system behavior predictable downstream.

---

## Why This Matters

The ingestion model is what makes NAM fundamentally different from:

* Vector databases
* RAG systems
* Knowledge graphs
* Search engines

Those systems attempt to **recover meaning at query time**.

NAM commits meaning **once**, geometrically, and never revises it implicitly.

This is what enables:

* Deterministic retrieval
* Explainable misses
* Domain-specific training
* Stable system behavior at scale

---

## Ingestion as a Strategic Boundary

Because ingestion defines the semantic space:

* Customization happens here
* Domain adaptation happens here
* Field-specific training happens here

Encoder heads can be retrained, swapped, or extended **without changing query logic**.

This makes NAM deployable across domains without forking the system.

---

## Summary

In NAM:

* Ingestion is semantic commitment
* Addresses are the only truth
* `__null__` is meaningful
* Determinism is mandatory
* Query can only retrieve what ingest made possible

Everything else follows from this.

→ See also: [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Query Model](QUERY_MODEL.md) | [Addressing Model](ADDRESSING_MODEL.md) | [Geometric Retrieval](GEOMETRIC_RETREIVAL.md)