# Terminology

*A shared language for reasoning about NAM*

NAM introduces a vocabulary that overlaps with databases, knowledge graphs, and machine learning — but **does not mean the same things**.

This document defines how key terms are used **specifically within NAM**.
If a term is defined here, this definition takes precedence over industry norms.

---

## Address

An **address** is a deterministic identifier that places a record at a specific location in NAM’s semantic space.

An address is **not**:

* a row key
* a pointer
* a similarity score
* a label

An address is:

* **structural**
* **composable**
* **stable**
* **navigable**

Addresses are how NAM *knows where something is*.

---

## Address Space

The **address space** is the full multidimensional space in which all NAM records live.

* It is finite but extensible
* It is deterministic
* It is shared across ingest and query

The address space does **not** change shape at runtime.
It only grows as new records are added.

---

## Coordinate

A **coordinate** is a single value along an address axis.

In production, coordinates are **LCA byte codes** — compact integer pairs produced by a learned codec:

* `"lca:42:17"` (an attribute coordinate)
* `"lca:201:55"` (an affordance coordinate)
* `"entity:tyler_denton"` (entity anchors remain string-based)
* `"__null__"` (wildcard)

Conceptually, coordinates are symbolic — their meaning comes from **position** in the codebook, not numeric magnitude. Semantically similar strings map to nearby codes, enabling geometric neighborhood queries.

---

## Axis (plural: Axes)

An **axis** is one dimension of the address space.

NAM commonly uses three primary axes:

* **X axis** — semantic attribute or modifier
* **Y axis** — relational or contextual qualifier
* **Z axis** — categorical or ontological signal

(Exact meanings depend on the encoder head and ontology tier.)

Axes are:

* fixed in count
* fixed in interpretation
* populated deterministically

---

## Null Coordinate (`__null__`)

`__null__` is a **first-class coordinate**, not a missing value.

It explicitly means:

> “This record is not constrained along this axis.”

Important properties:

* `__null__` participates in address construction
* `__null__` is queryable
* `__null__` is intentional, not accidental

A record with `__null__` is not “incomplete” — it is **less specific**.

---

## Point, Line, Plane (Geometric Interpretation)

Using the axes:

* **Point** — all axes populated
* **Line** — one axis unconstrained (`__null__`)
* **Plane** — two axes unconstrained

This is not metaphorical — it is how retrieval works.

Queries navigate:

* points when specific
* lines when partially specified
* planes when exploratory

---

## Partition

A **partition** is a coarse-grained namespace that groups related addresses.

Typical partition components include:

* namespace
* ontology tier
* entity anchor (if present)

Partitions exist to:

* constrain search
* improve performance
* preserve semantic boundaries

Partitions are **internal** and never exposed via the public API.

---

## Ontology

An **ontology** in NAM is a *semantic role category*, not a formal logic system.

Examples:

* `person`
* `location`
* `event`
* `artifact`
* `organization`

Ontologies define:

* *what kind* of thing something is
* not *what it means*

Multiple ontologies may apply to the same record.

---

## Ontology Tier

An **ontology tier** is an ordered group of ontologies used during query execution.

Tiers define:

* probe order
* widening behavior
* exploratory fallback

Tiers are static and defined offline.

---

## Role

A **role** is an abstract semantic function a record may play in a query.

Examples:

* `subject`
* `agent`
* `object`
* `location`
* `cause`

Roles bridge:

* affordances (what the user is asking)
* ontologies (what exists in storage)

---

## Affordance

An **affordance** describes *what kind of question* a query is asking.

Examples:

* identity
* location
* ownership
* activity
* relationship

Affordances:

* are detected at query time
* do **not** affect storage
* select execution strategies

If no affordance is detected, NAM enters **exploratory mode**.

---

## Entity Bundle

An **entity bundle** is a group of semantic components (attributes, affordances, contexts) that are linguistically linked to a specific entity.

Bundles are constructed by walking the dependency parse tree:

* Adjectival modifiers → attributes for the governed entity
* Verb arguments → affordances for the governed entity
* Contexts are shared across all bundles (they apply to the record as a whole)

Bundling prevents combinatorial address explosion by only generating addresses for relationships that actually exist in the text.

---

## Progressive Fan-Out

**Progressive fan-out** is the query execution strategy that probes addresses in order of decreasing specificity.

* Within each ontology tier, fully specified addresses (points) are probed first
* Then partially specified addresses (lines, planes) are probed
* If a tier yields enough matches (above a satisfaction threshold), remaining tiers are skipped

Progressive fan-out is:

* **Deterministic** — the same query always probes in the same order
* **Budget-bounded** — total probes and fetches are capped
* **Early-terminating** — stops when satisfaction is reached

---

## Satisfaction Threshold

The **satisfaction threshold** is the minimum number of results that must be found within an ontology tier before the query engine stops probing additional tiers.

This is not a quality measure — it is a **completeness gate**. If enough results exist in the most likely tiers, there is no need to probe less likely ones.

---

## Lease

A **lease** is a time-bounded ownership claim on a data partition.

In NAM:

* Leases are stored as key-value documents with TTLs
* Pods claim leases via compare-and-swap (CAS) operations
* Expired leases are automatically reclaimable by any surviving pod
* No external coordinator (Redis, ZooKeeper, etc.) is required

Leases exist to distribute CDC streaming work across ingest replicas without consensus protocols.

---

## Exploratory Mode

**Exploratory mode** is deterministic, not heuristic.

It means:

* no affordance constraint was applied
* queries probe across predefined ontology tiers
* results are structural, not ranked

Exploratory does **not** mean:

* fuzzy
* approximate
* “best guess”

It means *structurally broad*.

---

## Record

A **record** is the atomic unit of storage in NAM.

A record contains:

* payload (opaque to NAM)
* metadata
* one or more addresses

NAM does not interpret payloads at query time.

---

## Encoder Head

An **encoder head** is a deterministic component that:

* inspects input text
* emits coordinates
* never performs retrieval

Encoder heads:

* run at ingest and/or query time
* are trained offline
* do not learn at runtime

---

## Determinism

**Determinism** means:

> Given the same input, the same outputs are produced — always.

This applies to:

* address generation
* affordance detection
* query planning
* execution order

Determinism is a **hard requirement**, not a preference.

---

## Retrieval vs Navigation

NAM does not “retrieve results.”

It **navigates the address space** defined by:

* the query
* the stored records
* the execution plan

Results are intersections, not matches.

---

## Why terminology matters

NAM fails if teams use:

* “similarity” when they mean proximity
* “learning” when they mean accumulation
* “null” when they mean wildcard
* “ranking” when there is none

This document exists so that:

* design discussions stay precise
* implementations remain aligned
* expectations stay realistic

If a word is overloaded — **stop and define it**.

---

## Affordance Mode

**Affordance mode** is the query execution mode activated when an affordance verb is recognized in the query.

Affordance mode differs from exploratory mode in that:

* Specific ontology tiers are selected based on the affordance
* Probing is narrower and more targeted
* Verb synonym folding maps the query verb to canonical forms established at ingest

If no affordance verb is recognized, the query defaults to exploratory mode.

→ See also: [Design Principles](DESIGN_PRINCIPLES.md) | [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md) | [Geometric Retrieval](../ARCHITECTURE/GEOMETRIC_RETRIEVAL.md) | [Pipeline Architecture](../ARCHITECTURE/PIPELINE_ARCHITECTURE.md) | [Glossary](../APPENDIX/GLOSSARY.md)
