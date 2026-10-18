# The NAM Addressing Model

*Determinism, axes, types, and empty coordinates*

This document defines how semantic addresses are constructed in NAM and why that construction is strictly deterministic.

The semantic address index is the core of NAM's converged index. NAM answers a query through three access paths built from the same source records — the **address** path described here, an approximate **vector** path ([Vector Search](VECTOR_SEARCH.md)), and a **graph** path ([Graph Index](GRAPH_INDEX.md)). The vector and graph paths are additions around the address index; the graph index is itself built from addresses.

If the addressing contract is violated, the address path does not degrade gracefully — it returns nothing. This is by design.

---

## What an Address Is

An **address** is a deterministic mapping from meaning to a storage key.

An address is not:

* A vector
* A probability
* A similarity score

An address *is*:

* A structured semantic coordinate
* Produced by the encoder heads and the address builder
* Encoded with a learned codec (LCA) into compact codes
* Used directly as a storage key

The same input, with the same encoder artifacts, always produces the same addresses.

---

## Address Shape

Every address has seven segments:

```
tenant | partition | TYPE | entity | attribute | affordance | context
```

| Segment | Meaning | Source |
|---|---|---|
| `tenant` | Isolation scope | Supplied per record by the writer (deployment default otherwise) |
| `partition` | Client-defined sub-scope within a tenant | Supplied per record by the writer; no encoder head produces it |
| `TYPE` | The ontology class of what the record is about, as a BFO/CCO CURIE | Ontology head |
| `entity` | The entity this address is anchored on | Entity head (hash-based identifier) |
| `attribute` | x-axis: a quality or property asserted about the entity | Attribute head |
| `affordance` | y-axis: an action or interaction associated with the entity | Affordance head |
| `context` | z-axis: the circumstances the statement applies under (time, place, institution, field) | Context head |

The first four segments form the **partition key**. Storage routing hashes the partition key, so everything stored for the same (tenant, partition, type, entity) lives in the same storage partition. Geometry never influences routing.

Each address holds a set of record pointers: one line per record that produced that address, with its data pointer and timestamp. NAM stores where to find a record, never the record itself.

---

## The TYPE Segment: BFO/CCO Classes

The type segment holds a class from the **Basic Formal Ontology (BFO 2020)** and the **Common Core Ontologies (CCO)**, written as a CURIE (for example a CCO class for Person, or a BFO class such as Process).

The ontology head assigns one class per record:

* Each sentence is scored against a trained codebook of class centroids.
* An uncertain sentence backs off to the deepest ancestor class that carries the probability ("a geospatial region, not sure which kind").
* If nothing beyond existence can be asserted, the class is **BFO:0000001** (`entity`, the BFO root). This is the **unclassified** class. It is a valid partition, and NAM tracks how often records land there as a classification-quality metric.
* The record's class is the vote of its sentences.

Every address of a record carries the record's class. All of a biography's addresses — including those anchored on other entities the biography mentions — sit under the class of the biography's subject.

Because the class set is a formal ontology, NAM can reason about it structurally: which BFO branch a class belongs to, which classes subsume which, and which are disjoint. The query planner uses this (see [Query Model](QUERY_MODEL.md)).

---

## Entity Anchoring

The entity segment is a **hash-based identifier**: the first 12 hex characters of a SHA-256 of the entity's normalized name.

Entity identifiers are:

* **Deterministic** — the same normalized name always gives the same identifier, on every machine.
* **Type-independent** — the class is not part of the identifier. The same entity can be classified differently in different texts (or between a long article and a short query) and still have one identifier.
* **Stable** — reclassifying a record does not change its entity identifiers.

Entity resolution (mapping "Abraham Lincoln", "Lincoln" and similar surface forms to one identifier) uses the session store, which accumulates surface forms as records are ingested. The query path uses the same resolver as the ingest entity head, so ingest and query agree on identity.

---

## Axis Values: LCA Codes

Axis values are not stored as strings. Each attribute, affordance and context value is encoded by **LCA (Learned Codec for Addressing)**:

1. A character-level CNN embeds the string.
2. A two-stage residual vector quantizer maps the embedding to two codebook indices, each from a codebook of 1024 entries.
3. The axis value is written `lca:coarse:fine`.

Properties:

* **Deterministic** — the same string always encodes to the same code pair.
* **Geometric** — similar strings tend to share a coarse code or sit at neighbouring codebook entries, which the query path exploits through neighbour fan-out.
* **Compact** — keys stay short regardless of the source string.
* **Frozen** — the codec is trained offline and fixed at deployment.

At ingest, the exact code of each value is stored. At query time, each axis code is also expanded to its nearest codebook neighbours (see [Query Model](QUERY_MODEL.md)).

---

## Empty Axes

When a head produces no value for an axis, that axis is encoded as **`lca:0:0`**, a reserved code meaning "this address does not constrain this axis."

Every axis always has a value: either a concrete code or `lca:0:0`. There is no third state.

An empty axis is a semantic statement, not missing data. A record whose only signal is an entity and an affordance occupies a *line* in the address space, not a point. At query time, empty axes are left out of scan prefixes, and when filtering scan results an empty stored code counts as neutral: it neither matches nor disqualifies. See [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md).

---

## Storage Layouts: A Covering Index

A query specifies some of the three axes and leaves the rest open. The address index is stored so that every combination of specified axes is answered by a **direct match** (exact or by prefix) on something stored — no wildcard copies, and nothing derived at read time. NAM supports two storage layouts with the same partition key, the same addresses and the same query semantics.

### The reference deployment's layout

Each partition's addresses are stored together, under that covering guarantee. A query reads what its plan needs in a fixed number of batched round trips per planning tier, however many addresses a partition holds.

### Rotations layout

Each base coordinate (x, y, z) is stored under three keys, one per cyclic rotation of the axes:

| Rotation | Axis order after the partition key |
|---|---|
| `xyz` | attribute, affordance, context |
| `yzx` | affordance, context, attribute |
| `zxy` | context, attribute, affordance |

For any set of axes a query specifies, one rotation puts those axes at the front of the key, so the query is answered by a **prefix scan** on that rotation, and each key's value holds the record pointers.

In both layouts everything stored for an address shares its partition key and therefore its storage partition, and results are deduplicated by record.

---

## Entity-Anchored Bundling

A record mentions many entities and many attributes, actions and contexts. Crossing all of them would multiply addresses without meaning.

NAM instead builds **entity bundles** using the dependency parse produced by the NLP stage:

* Each entity gets its own bundle.
* Attributes are attached to the entities they modify.
* Affordances are attached through verb-argument relations.
* Context values are scoped to entities where the evidence allows, and otherwise apply to the record.
* Empty axes in a bundle become `lca:0:0`.

Addresses are then built per bundle. Address count per record is bounded by what the text actually expresses.

---

## Determinism as a First Principle

Given the same input text, the same encoder artifacts and the same tenant and partition, NAM emits the same addresses, on every machine, every time. There is no stochastic fallback. Where an internal step would otherwise depend on iteration order (for example, sets of values), the builder sorts by content first.

This enables:

* Rebuildable state — the address index can be regenerated from source records.
* Reproducible behaviour — the same query against the same data plans the same scans.
* Safe distributed deployment — every ingest worker writes the same keys for the same record.

---

## Addressing vs Retrieval

Addressing answers: *where could this meaning be stored?*

Retrieval answers: *which of those locations actually hold records?*

Addressing is semantic, deterministic and side-effect free. Retrieval is mechanical, storage-backed, and bounded by the addresses the planner produces.

---

## Failure Modes (By Design)

If ingest and query disagree on any part of the contract — tenant or partition scope, codec, storage layout, or entity normalization — probes route to the wrong place or match nothing. The address path then returns no results, predictably. In the converged query, the vector path may still answer, and the `access_paths` trace shows which path produced what.

---

## Summary

* An address is `tenant | partition | TYPE | entity | attribute | affordance | context`.
* TYPE is a BFO/CCO class; BFO:0000001 means unclassified.
* Entities are 12-hex SHA-256 identifiers of the normalized name, independent of type.
* Axis values are LCA codes `lca:coarse:fine`; `lca:0:0` is an empty axis.
* The storage layout makes every combination of specified axes a direct match on something stored.
* Bundling ties components to the entities they belong to.
* Ingest and query must align exactly.

→ See also: [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [Query Model](QUERY_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Graph Index](GRAPH_INDEX.md) | [Vector Search](VECTOR_SEARCH.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)
