# Terminology

*A shared language for reasoning about NAM*

NAM introduces a vocabulary that overlaps with databases, knowledge graphs, and machine learning — but **does not mean the same things**.

This document defines how key terms are used **specifically within NAM**.
If a term is defined here, this definition takes precedence over industry norms.

---

## Address

An **address** is a deterministic identifier that places a record at a specific location in NAM's semantic space.

An address is **not**:

* a row key for the record itself
* a similarity score
* a label

An address is:

* **structural**
* **composable**
* **stable**
* **navigable**

Addresses are how NAM *knows where something is*. An address has the shape:

```
tenant | partition | type | entity | x | y | z
```

Each stored address points back to the source records that produced it.

---

## Address Space

The **address space** is the full multidimensional space in which all addressed records live.

* It is finite but extensible
* It is deterministic
* It is shared across ingest and query

The address space does **not** change shape at runtime.
It only fills in as new records are added.

---

## Access Path

An **access path** is one of the three ways NAM can answer a query:

* **address** — prefix scans over the semantic address index (deterministic)
* **vector** — nearest records by embedding, above a cosine floor (approximate)
* **graph** — walks outward from resolved entities over the graph index

A query may run several paths at once, chosen by a routing policy, or the caller may name the paths explicitly.

---

## Access Paths Trace

The **access paths trace** (`access_paths`) is returned with every query response. For each path it reports whether the path was used, skipped, not requested, used as a fallback, or failed; the reason; the time it took; path-specific work counters; and how many results it contributed.

---

## Fusion

**Fusion** merges the results of the access paths into one list, de-duplicated by the record they point to. Address results lead; a record whose key names a resolved entity is moved to the front. Each fused row carries **sources** — the list of paths that found it — and, when the vector path contributed, its `cosine`.

---

## Coordinate

A **coordinate** is a single value along an address axis.

Axis coordinates are **LCA codes** — pairs of codebook indices produced by a learned codec:

* `lca:42:17` — a coarse code and a fine code
* `lca:0:0` — the non-specific coordinate (no value on this axis)

Coordinates are symbolic — their meaning comes from **position** in the codebook, not numeric magnitude. Semantically similar strings map to nearby codes, which the query planner exploits for neighbourhood fan-out.

---

## LCA (Learned Codec for Addressing)

**LCA** is the codec that turns an axis value (an attribute, affordance, or context string) into a coordinate. A character-level convolutional encoder produces an embedding; a two-stage residual vector quantizer maps it to a coarse index and a fine index, each from a codebook of 1024 entries per stage. LCA is trained offline and frozen at deployment; the same string always produces the same code.

---

## Axis (plural: Axes)

An **axis** is one dimension of the address space. NAM uses three:

* **X axis** — attribute: qualities asserted about the entity
* **Y axis** — affordance: the action or interaction associated with the entity
* **Z axis** — context: the circumstances under which the record applies (time, place, institution, field)

Axes are:

* fixed in count
* fixed in interpretation
* populated deterministically

---

## Non-Specific Coordinate

When an encoder head produces no value for an axis, the coordinate is set to `lca:0:0` — a reserved code that means "non-specific along this axis."

This is a **first-class coordinate**, not missing data.

It explicitly means:

> "This record is not constrained along this axis."

* Non-specific coordinates are stored in the covering index like any other value
* At query time, the planner omits non-specific axes from the scan prefix, matching across all values along that axis

A record with a non-specific axis is not "incomplete" — it is **less specific**.

---

## Point, Line, Plane, Cube (Geometric Interpretation)

Using the three axes:

* **Point** — all axes in the scan
* **Line** — one axis left open
* **Plane** — two axes left open
* **Cube** — all axes left open (the whole partition)

This is not metaphorical — it is how address-path retrieval works. Queries widen from points to cubes by requiring fewer axes to agree.

---

## Covering Index (and Rotations)

The **covering index** stores each address so that any combination of specified axes is a direct match (exact or by prefix) on something stored, without wildcard copies and with nothing derived at read time. On the reference deployment each partition's addresses are stored together under that guarantee. In the rotations layout each address is stored under the **cyclic rotations** of its axes — `xyz`, `yzx`, `zxy` — so that one prefix scan answers any partial-axis query.

---

## Partition

A **partition** is the leading part of an address key that groups related addresses and decides where they are stored:

```
tenant | partition key | ontology class | entity
```

* **tenant** and **partition key** come from the source record (callers can scope a query to them)
* **ontology class** is the record's type
* **entity** is the entity anchor

Everything stored for an address shares its partition and therefore lands in the same storage partition, so probes are local.

---

## Ontology

NAM's **ontology** is the BFO 2020 upper ontology together with the Common Core Ontologies (CCO v2.2). Every record is typed with one class from this standard vocabulary, identified by a compact identifier such as `cco:ont00001262` (Person) or `BFO:0000015` (process). That class becomes the type segment of every address the record produces.

The ontology classifier backs off when unsure: from a specific class to the deepest ancestor it is confident in, and finally to `BFO:0000001` (entity).

---

## Unclassified (BFO:0000001)

`BFO:0000001` is the root class of BFO — "entity." In NAM it means **no class beyond existence was asserted**. Records the classifier could not type are stored under it, and the query planner always includes a probe of it as a safety net. The share of records landing there is tracked as a classification quality metric; it is a miss, not a category.

---

## Ontology Tier

An **ontology tier** is an ordered group of ontology classes the address-path planner explores together. Tiers follow BFO's top-level branches (independent continuants, occurrents, and the dependent continuants). A query's resolved entities are probed first under the classes they are actually stored under (the **STORED** tier), then broader tiers are explored.

---

## Entity

An **entity** is a specific thing a record refers to. Its identifier is a deterministic hash of its normalized name, independent of its type, so the same entity has the same identifier at ingest and at query time regardless of how it was classified.

---

## Entity Resolution

**Entity resolution** maps a surface form ("Lincoln", "Abraham Lincoln") to an entity identifier using the session store of known names. At query time, the same resolver used by ingest finds the entities a query names: the longest specific name wins, and generic nouns are dropped once a specific name resolves.

---

## Entity Bundle

An **entity bundle** is a group of semantic components (attributes, affordances, contexts) that are linguistically linked to a specific entity.

Bundles are built from the dependency parse:

* Adjectival modifiers become attributes of the entity they modify
* Governing verbs become affordances of their arguments
* Context values are scoped to the entities they relate to where the evidence allows

Bundling bounds the number of addresses a record produces to the relationships the text actually expresses.

---

## Affordance

The word **affordance** is used in two related senses:

* As an **axis value**: the action or interaction associated with an entity, extracted from verbs at ingest and folded to canonical forms (the Y axis).
* As a **query affordance**: the kind of question a query is asking, detected from its interrogative shape. A recognized query affordance selects **affordance mode**; otherwise the query runs in **exploratory mode**.

---

## Affordance Mode / Exploratory Mode

* **Affordance mode** — a recognized query affordance selects specific ontology tiers, so planning is narrower.
* **Exploratory mode** — no affordance constraint; the planner explores across ontology tiers in a fixed order.

Exploratory does **not** mean fuzzy or approximate. It means *structurally broad*. Both modes are deterministic.

---

## Progressive Fan-Out

**Progressive fan-out** is the address path's execution strategy: probe in order of decreasing specificity.

* Geometry widens level by level — point, then line, plane, cube — only while the request is not yet satisfied
* Within a level, tiers are scanned in planned order, and each axis code is expanded to its nearest codebook neighbours
* Prefix scans page through results until enough distinct records are found
* Scans and fetches are budget-capped

Progressive fan-out is deterministic, bounded, and early-terminating.

---

## Satisfaction Threshold

The **satisfaction threshold** is the number of distinct records that, once found, stops the planner from moving on to further tiers. It is a **completeness gate**, not a quality measure.

---

## Vector Index

The **vector index** files each record's embedding into a region of a partition trained on the data. A query reads only the regions nearest to it and ranks the records found there by cosine similarity. It is approximate by contract: it returns the top results above the cosine floor, not a complete set.

---

## Cosine Floor

The **cosine floor** is the minimum similarity a vector result must have. It is a relevance requirement enforced by the deployment: a request may raise it, never lower it. The value is calibrated per embedding model.

---

## Graph Index

The **graph index** is derived from the addresses: record-entity edges in both directions, and the ontology classes each entity is stored under. It supports graph walks and record walks (both by record co-occurrence), entity-document lookups, and tells the planner which classes an entity is stored under.

---

## Index Builder

The **index builder** is the change-stream consumer that builds the graph index from address mutations. It is sharded across replicas, resumes from checkpoints, and runs off the ingest acknowledgement path — so the graph trails the address index and never blocks it.

---

## Change Data Capture (CDC)

**CDC** is how data enters NAM: NAM consumes the data service's change stream of the source bucket. Records are never pushed into the pipeline directly.

---

## Write-Ahead Log (WAL)

Each ingest pipeline hands records to its workers through a durable, file-backed message stream on a per-pod volume — the **WAL**. A batch is acknowledged and removed only after its writes have landed; a failure causes redelivery. A full WAL rejects new records rather than evicting old ones.

---

## Lease

A **lease** is a time-bounded ownership claim on a source partition (vbucket).

* Leases are documents with a server-side expiry in the coordination store
* The owner renews them periodically; an expired lease is claimable by any surviving pod
* No external coordinator is required

Leases distribute change-stream work across ingest replicas without a consensus protocol.

---

## Session Store

The **session store** holds the vocabularies the encoder heads build up: entities and their surface forms, attribute, affordance and context categories. It is a warm cache — kept across pipeline resets — mirrored into node-local memory-mapped caches for fast lookup.

---

## Record

A **record** is a source document. NAM reads it during ingest but never owns it; the source store is the system of record.

A record contributes:

* one or more addresses
* one vector-index entry
* graph-index entries derived from its addresses

NAM does not interpret payloads at query time.

---

## Encoder Head

An **encoder head** is a deterministic component that:

* inspects parsed text
* emits values for one dimension of meaning
* never performs retrieval

NAM's heads are ontology, entity, attribute, affordance, and context. They:

* run at ingest and at query time
* are trained or curated offline
* do not learn at runtime

Each head fills a **slot**. The implementation in a slot is selected by an admin and, for heads that keep no state, can be swapped on a running cluster. Implementations developed outside NAM's own tree are published in the open-source `nam-heads` package.

---

## Determinism

**Determinism** means:

> Given the same input, the same outputs are produced — always.

In NAM this applies to:

* address generation
* query affordance detection
* address-path planning and execution order

The vector path's candidate set is approximate by contract; that is stated, not hidden.

---

## Retrieval vs Navigation

NAM's address path does not "retrieve results" by score.

It **navigates the address space** defined by:

* the query
* the stored records
* the execution plan

Results are intersections, not matches.

---

## Why terminology matters

NAM fails if teams use:

* "similarity" when they mean proximity in address space
* "learning" when they mean accumulation
* "null" when they mean non-specific
* "ranking" on a path where there is none
* "the result" when they mean "the result from one path"

This document exists so that:

* design discussions stay precise
* implementations remain aligned
* expectations stay realistic

If a word is overloaded — **stop and define it**.

→ See also: [Design Principles](DESIGN_PRINCIPLES.md) | [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md) | [Geometric Retrieval](../ARCHITECTURE/GEOMETRIC_RETRIEVAL.md) | [Query Model](../ARCHITECTURE/QUERY_MODEL.md) | [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md) | [Graph Index](../ARCHITECTURE/GRAPH_INDEX.md) | [Glossary](../APPENDIX/GLOSSARY.md)
