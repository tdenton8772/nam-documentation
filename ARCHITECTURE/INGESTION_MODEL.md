# Ingestion Model

## How Raw Information Becomes Addressable Meaning in NAM

In NAM, ingestion is not a preprocessing step or an indexing phase in the traditional sense.
It is the act of **projecting raw information into deterministic, retrievable structure**.

Every record that enters NAM is indexed three ways, from the same source:

* **Semantic addresses**, built by the encoding pipeline and written to the address index
* **A vector index entry**, built by embedding the record, written to the vector index
* **Graph records**, derived from the record's addresses, written to the graph index

The semantic address path is the core, and most of this document is about it. For that path, ingestion and query are symmetric: both use the same parser, the same encoder heads, the same entity resolver and the same coordinate codec. The only difference is direction:

* Ingestion **writes** addresses (as covering-index entries)
* Query **probes** addresses (with key lookups and prefix matches on the covering index)

---

## Core Principle: Ingest Defines the Address Space

The address path does not learn what a record means at query time. Everything it can retrieve must first be **materialized as addresses during ingestion**.

If the address path fails to retrieve a record, it is not because a similarity score was too low. It is because the relevant addresses were never emitted, or the query encoded to different ones. That is an explainable miss: the address emitted at ingest and the prefix scanned at query can both be inspected.

The vector path is an independent index with its own coverage. It is a second way in, not a repair of the first, and every result says which path found it.

---

## What Ingestion Is (and Is Not)

### Ingestion **is**:

* Deterministic address construction
* Semantic projection via encoder heads
* Lossy by design (meaning is discretized into classes and codes)
* Repeatable and auditable
* Domain-trainable via head artifacts and head stores

### Ingestion **is not**:

* Schema enforcement
* Ranking or scoring
* A system of record (the source store is)

---

## How Records Enter

Records enter through **change data capture**: they are written to the source store, and NAM's pipelines follow its change stream. There is no separate import API. Each pipeline resumes from its own checkpoints after a restart.

The address pipeline runs each record through:

1. **NLP**: rule-based tokenization, POS tagging, lemmatization, NER and dependency parsing. Deterministic, no learned weights.
2. **Ontology**: a BFO 2020 / CCO v2.2 class for each sentence and for the record.
3. **Encoder heads**: entity, attribute, affordance, context. On the reference deployment the entity head runs first and a deterministic sentence resolver ties each sentence's statements to the resolved entities; the attribute, affordance and context heads then run in parallel.
4. **Join and durable hand-off**: the record's head results are joined and written to an on-disk queue.
5. **Addressing**: entity-anchored bundling, coordinate coding, address construction, write, acknowledge.

> See: [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) for the runtime architecture

---

## Ontology: The Type of Every Address

The ontology head answers "what kind of thing is this record about?" with a class from a certified, standards-based hierarchy (BFO 2020 and the Common Core Ontologies).

* Each sentence is scored against a trained codebook; an uncertain sentence backs off to the deepest ancestor class the evidence supports.
* The record's class is the vote of its sentences.
* When nothing beyond existence can be asserted, the head abstains with the root class `BFO:0000001`. Such records are still addressed; the abstain rate is tracked as a classification-quality metric, not treated as a valid type.

The record's class is written into the **type segment of every address the record emits**, including addresses anchored on other entities the record mentions. A biography's address anchored on an award is filed under the class of the person the biography is about.

---

## Encoder Heads at Ingest Time

| Head | Extracts | Address role |
|---|---|---|
| **Entity** | Specific things referred to, resolved to canonical identifiers | Entity anchor |
| **Attribute** | Qualities and properties asserted in the text | Axis x |
| **Affordance** | Actions and interactions, folded to canonical verbs | Axis y |
| **Context** | Particular circumstances: time periods, places, institutions, fields | Axis z |

Each head receives the same parse and the record's class, extracts its signal independently, and emits zero or more values. A head that finds nothing for a record emits nothing, and the axis is marked empty.

---

## Entity Resolution at Ingest Time

Entity resolution maps surface forms ("Abraham Lincoln", "Lincoln") to a **canonical entity identifier**.

* Mentions are extracted and normalized to a canonical form.
* The normalized form is looked up in the entity store. A known surface form resolves to its existing entity.
* An unknown form registers a new entity. Its identifier is a **hash of the canonical form**, independent of type, so the same name always yields the same identifier on every replica.

The entity store lives in the `session` bucket, is cached on every node, and **persists across reloads**: it is a warm cache of everything ingest has learned. A maintenance task can re-resolve addresses written early in a load, when the store knew fewer names, against what it knows now.

Query-time resolution uses the same resolver over the same store, so ingest and query agree on entity identity.

---

## Entity-Anchored Address Construction

Rather than crossing every semantic component with every other, the bundler uses the dependency parse to decide which components *belong to* which entity:

* Each entity gets its own bundle of attributes, actions and context
* Attributes and actions are scoped to the entities they modify or govern
* Context values carry the sentences they came from and are scoped to entities through those sentences

The number of addresses per record is bounded by the structure of the text, not by a combinatorial product.

→ See: [Addressing Model](ADDRESSING_MODEL.md)

---

## Coordinates and Empty Axes

Each axis value is encoded by the LCA codec into a code pair `lca:coarse:fine`. An axis with no value is encoded as `lca:0:0`.

An empty axis does not mean "unknown". It means:

> "This record does not constrain this axis."

This lets records exist as points (every axis specified), lines, planes or cubes (fewer axes specified), and lets queries widen geometrically by scanning shorter prefixes.

Addresses are stored in a covering layout, so any combination of specified axes is a direct match on something stored.

---

## Address Emission and Multiplicity

A single record emits many addresses: one set per entity bundle, each stored in the covering index. This is expected. NAM preserves the breadth of what a record says; precision is applied at query time by choosing which prefixes to scan.

---

## Ingest-Time Learning

Ingest learns, in a constrained and visible way:

* New entities are registered and new surface forms are mapped to existing entities
* Unknown attribute and affordance terms are categorized from context
* New context values (places, institutions, fields) join the context store
* Curated seed vocabulary takes priority over discovered terms

All of this learning lands in the head stores. It does not rewrite addresses already written; it changes what future records (and future queries) resolve to. Navigation rules, codecs and the ontology codebook are fixed artifacts that change only by an explicit retraining and reload.

→ See: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md)

---

## Durability of Ingest

The address pipeline is **acknowledge-after-write**:

* A record whose head results are complete is written to a durable, on-disk queue before addressing.
* A queued batch is acknowledged only after all of its address writes have landed.
* A failed write sends the batch back for redelivery. If a redelivery limit is configured and reached, the records are counted in the pipeline's error total, never dropped silently.
* Redelivery may append an address line twice; readers de-duplicate lines by record pointer.

On the reference deployment every hop between the stages upstream of that queue is a durable on-disk stream as well, and each stage acknowledges its input only after the next stage has it. A deployment can run those hops on a lightweight pod-local bus instead. Either way, a record whose head results never all arrive is evicted after a timeout and counted as an error.

---

## Storage Is Address-First

The address index does not store records by content. It stores **address keys whose values point back to records**: each value line holds the record's pointer and a timestamp. The record itself stays in the source store and is fetched at query time.

The graph builder follows the address index's own change stream and writes the graph records; the vector pipeline follows the source store independently. Neither is on the address pipeline's acknowledgement path.

→ See: [Data Persistence](DATA_PERSISTENCE.md) | [Graph Index](GRAPH_INDEX.md) | [Vector Search](VECTOR_SEARCH.md)

---

## Determinism as a Contract

Given the same input and the same artifacts (parser rules, ontology codebook, head vocabularies, LCA codec):

* The address pipeline emits the same addresses
* Into the same partitions
* With the same keys

Head-store learning is the one moving part, and it is recorded state, not randomness: replaying the same records over the same store produces the same result.

---

## Ingestion as a Strategic Boundary

Because ingestion defines the address space, customization happens here: the ontology codebook, the head vocabularies and the codec can be retrained for a domain without changing query logic. Changing them means re-ingesting, because addresses written under one artifact are not comparable with addresses built under another.

---

## Summary

In NAM:

* Every record is indexed three ways from one source
* Address ingestion is semantic commitment: the record's class, its entities and its coded axes
* Empty axes (`lca:0:0`) are meaningful
* Entity identity is a hash of the canonical name, shared by ingest and query
* Handed-off records are acknowledged only after their writes land
* The address path can only retrieve what ingest made addressable

→ See also: [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Query Model](QUERY_MODEL.md) | [Addressing Model](ADDRESSING_MODEL.md) | [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Data Persistence](DATA_PERSISTENCE.md)
