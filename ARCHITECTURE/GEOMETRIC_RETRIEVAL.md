# Geometric Retrieval in NAM

*Points, lines, planes, and cubes*

This document explains the core retrieval mechanism in NAM.

Everything else — encoders, affordances, planners, storage — exists to make **geometric retrieval** possible, deterministic, and efficient.

---

## The Core Idea

NAM retrieves information by **navigating geometry**, not by ranking similarity.

Instead of asking:

> “Which records are closest to this query?”

NAM asks:

> “Which records occupy the same region of semantic space?”

That region may be:

* A **point**
* A **line**
* A **plane**
* Or a **volume (cube)**

Retrieval is navigation, not comparison.

---

## The Address Space

Every record in NAM is stored at one or more **semantic addresses**.

Conceptually, an address looks like:

```
(namespace, ontology, entity, x, y, z)
```

Where:

* `namespace` and `ontology` define *what kind of thing* this is
* `entity` anchors meaning (optional)
* `x`, `y`, `z` are semantic axes

Each axis may contain:

* A concrete value (e.g. `outdoor`, `bicycle`, `professional`)
* Or a special wildcard value: `__null__`

---

## Why Geometry Exists at All

Most real-world data is **incomplete**.

A record rarely expresses:

* All attributes
* All dimensions
* All relationships

If NAM required every axis to be populated, most records would be unusable.

Instead, NAM treats missing information as **intentional openness**, not absence.

That openness is modeled geometrically.

---

## Points

A **point** is a fully specified address.

Example:

```
(entity:tyler_denton, location, x=park, y=outdoor, z=public)
```

This represents:

* A very specific semantic fact
* High precision
* Low recall

Points are useful when:

* The data is explicit
* The query is narrow
* Precision matters

---

## Lines

A **line** occurs when one axis is open (`__null__`).

Example:

```
(entity:tyler_denton, location, x=park, y=__null__, z=__null__)
```

This represents:

* “Any park-related location”
* Across all outdoor/indoor/public/private distinctions

Lines allow:

* Partial information
* Flexible matching
* Safe generalization

---

## Planes

A **plane** occurs when two axes are open.

Example:

```
(entity:tyler_denton, location, x=__null__, y=__null__, z=outdoor)
```

This represents:

* “Anything outdoor related”
* Across many specific activities or places

Planes are essential for:

* Natural language queries
* Vague or exploratory questions
* Human-style reasoning

---

## Cubes (Volumes)

A **cube** exists when all axes are open.

Example:

```
(entity:tyler_denton, location, x=__null__, y=__null__, z=__null__)
```

This represents:

* “Any location-related information”
* Maximum recall
* Minimum specificity

Cubes are typically used:

* As fallbacks
* During exploratory retrieval
* When affordance detection is weak or ambiguous

---

## Why `__null__` Is Not “Missing”

A critical design choice in NAM:

> `__null__` does **not** mean “unknown”
> It means “non-specific along this axis”

This distinction matters.

* A missing field would be an error
* `__null__` is an intentional declaration of openness

At ingest time, `__null__` values participate in the covering index like any other coordinate value — they are stored in all N rotations of the key.

At query time, `__null__` axes are treated as unspecified. The query planner omits them from the prefix, which causes the prefix scan to match across all values along that axis. This is how a query naturally widens from a point (all axes specified) to a line, plane, or volume — by shortening the prefix.

`__null__` allows a record to:

* Participate in multiple queries
* Be retrieved even when queries are underspecified
* Occupy a line, plane, or volume

Without `__null__`, geometric retrieval collapses.

---

## Determinism and Exact Matching

NAM does **not** compute similarity.

It performs **deterministic, structured storage lookups**.

That means:

* Every query corresponds to a concrete region of address space
* Every region maps to a deterministic set of storage operations
* The same query against the same data always returns the same results

Geometry is resolved **before** storage access.

This is why determinism is possible.

---

## Covering Index: How Geometry Becomes Storage

NAM encodes geometric relationships directly into storage keys using a **covering index** based on cyclic permutations.

For each base address with N coordinate axes, NAM stores N rotations of the axis ordering:

* Rotation 0: `(partition, x, y, z)`
* Rotation 1: `(partition, y, z, x)`
* Rotation 2: `(partition, z, x, y)`

This creates a **covering index** — every possible single-axis query can be answered by scanning the rotation where that axis appears first. Queries specifying multiple axes use the rotation where those axes form a contiguous prefix.

The covering index replaces the older approach of storing explicit wildcard copies. Instead of storing 2^N copies per base address with every possible combination of wildcarded axes, NAM stores only N copies — one per rotation. For 3 axes, this is 3 rotations instead of 8 wildcard combinations.

### Why Rotations Work

Consider a query that specifies only the y-axis value. In rotation 1, the y-axis comes first in the key. A prefix scan on `(partition, y_value)` efficiently finds all matching records regardless of their x and z values.

This is geometrically equivalent to querying a **plane** — all points along a plane where y is fixed — but implemented as a simple prefix scan rather than 2^(N-1) individual lookups.

---

## Storage as a Geometric Substrate

From NAM’s perspective, storage must support two operations:

1. **Key-based writes** — store a record at a specific address key
2. **Prefix scans** — retrieve all records whose keys share a common prefix

The storage layer does **not**:

* Understand geometry
* Interpret axes
* Perform joins or ranking

Geometry is encoded *into the keys themselves*, and the rotation scheme ensures any axis combination can be queried via a single prefix scan.

---

## Why This Works with KV Stores

Key-value systems with ordered storage (LSM trees, B-trees) naturally support prefix scans. NAM exploits this by:

* Encoding geometric coordinates into key prefixes
* Using cyclic rotations to ensure any query axis can appear first
* Routing all rotations of the same address to the same storage partition

The result: geometric retrieval without specialized index structures, secondary indexes, or query planners in the storage layer. The "index" is the data itself, arranged so that any geometric query maps to a contiguous key range.

There is no guessing at runtime.

---

## Query Widening (Not Guessing)

When a query does not match at maximum specificity:

* NAM does not “relax similarity thresholds”
* NAM does not “search harder”

Instead, it **widens geometry** by scanning progressively broader regions:

```
point → line → plane → volume
```

Each widening step corresponds to a prefix scan on a shorter prefix — specifying fewer axes means scanning a broader region of the covering index. The rotation scheme ensures that any subset of axes can be efficiently queried regardless of which axes are specified.

Each step is:

* Explicit
* Ordered
* Deterministic

This is how NAM preserves control without losing recall.

---

## Why Ingest and Query Must Align

For geometric retrieval to work:

* Ingest must store addresses using the covering index (all N rotations per base address)
* Query must use the same coordinate encoding and rotation scheme
* The same axes, the same codebook, the same encoder heads

If the covering index at ingest uses different rotations or coordinate encoding than the query planner expects, prefix scans will miss records.

This alignment is **a core architectural invariant**.

---

## What This Enables

Geometric retrieval allows NAM to:

* Handle vague language safely
* Support partial facts
* Remain deterministic
* Avoid similarity math
* Scale with storage, not vectors

Most importantly:

> NAM retrieves meaning without collapsing it.

---

## Summary

* NAM stores meaning as geometry
* Geometry is composed of points, lines, planes, and volumes
* A covering index of cyclic rotations enables efficient geometric queries on ordered KV stores
* Prefix scans replace explicit wildcard enumeration — N rotations instead of 2^N copies
* Storage is a passive geometric substrate that supports key writes and prefix scans
* Retrieval is navigation, not ranking

If this document makes sense, the rest of NAM will too.

→ See also: [Addressing Model](ADDRESSING_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)