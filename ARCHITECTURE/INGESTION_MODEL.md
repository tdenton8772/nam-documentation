# Ingestion Model

## How Raw Information Becomes Addressable Meaning in NAM

In NAM, ingestion is not a preprocessing step, an ETL pipeline, or an indexing phase in the traditional sense.
It is the act of **projecting raw information into a deterministic semantic space**.

Ingestion and query are symmetric operations. Both use the same encoder heads, the same addressing rules, and the same geometric model. The only difference is **direction**:

* Ingestion **writes** addresses (as rotated covering index keys)
* Query **scans** addresses (via prefix scans on the covering index)

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

## The Ingest Pipeline

Ingestion in NAM is a **staged pipeline**, not a monolithic function call.

Raw data enters through **change data capture (CDC)** — NAM watches an upstream data store for mutations and streams them into the pipeline automatically. Data does not enter via batch import or direct API submission.

Each record passes through the following stages in order:

1. **NLP** — Rule-based linguistic analysis (tokenization, POS tagging, lemmatization, NER, dependency parsing). Fully deterministic, no neural models.
2. **Ontology classification** — Determines the semantic type of the record (person, object, location, concept, etc.)
3. **Encoder head fan-out** — Multiple heads run in parallel, each extracting a specific semantic signal
4. **Addressing** — Entity-anchored bundling and deterministic address construction

> See: [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) for the full runtime architecture

---

## Encoder Heads at Ingest Time

During ingestion, the parsed document is passed through a set of **encoder heads** running in parallel.

NAM currently uses five encoder heads:

* **Entity** — extracts named entities and resolves them to canonical identifiers
* **Ontology** — classifies the semantic type of the overall record
* **Attribute** — extracts properties and descriptors (adjectives, modifiers)
* **Affordance** — extracts actions and capabilities (verbs)
* **Context** — extracts situational, temporal, and categorical signals

Each head:

* Receives the same parsed document (NLP output)
* Extracts its specific signal independently
* Emits one or more semantic components

All emitted components are treated as **equally valid**.

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

## Entity Resolution at Ingest Time

Entity resolution is the process of mapping surface forms ("Abraham Lincoln", "Lincoln", "Abe Lincoln") to a **canonical entity identifier**.

NAM uses **hash-based entity identifiers**: a deterministic hash of the canonical entity form, independent of the entity's ontology type. This ensures that:

* The same entity always receives the same identifier, regardless of how it was classified
* Ingest and query always agree on entity identity
* Ontology reclassification (e.g., "concept" at ingest vs "object" at query) does not break retrieval

Entity resolution follows a **fuzzy-first** strategy:

* Candidate keys are checked first (approximate matches via canonical form similarity)
* Surface form lookups serve as a fallback
* New entities are registered when no match is found

The entity store acts as a **warm cache** that accumulates knowledge across ingestion cycles. It is not flushed during pipeline resets.

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

The record payload itself is opaque to the addressing system and lives in the source store. NAM’s storage layer only cares about:

* The address
* The record identifier
* The pointer to the payload

Storage writes during ingestion are **asynchronous and fire-and-forget**. The addressing stage queues writes to a background thread and never blocks the pipeline. Write batching happens naturally as the background thread accumulates and flushes in groups.

This separation is intentional and allows storage backends to be swapped without changing semantics.

→ See: [Data Persistence](DATA_PERSISTENCE.md)

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

→ See also: [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Query Model](QUERY_MODEL.md) | [Addressing Model](ADDRESSING_MODEL.md) | [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Data Persistence](DATA_PERSISTENCE.md)