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
> It means “wildcard”

This distinction matters.

* A missing field would be an error
* A wildcard is a geometric expansion

`__null__` allows a record to:

* Participate in multiple queries
* Be retrieved even when queries are underspecified
* Occupy a line, plane, or volume

Without `__null__`, geometric retrieval collapses.

---

## Determinism and Exact Matching

NAM does **not** scan storage.

It does **not** compute similarity.

It performs **exact key-based lookups**.

That means:

* Every query probe corresponds to a concrete address
* Every address corresponds to a concrete storage key

Geometry is resolved **before** storage access.

This is why determinism is possible.

---

## Storage as a Geometric Substrate

From NAM’s perspective, storage is simply:

> A system that can retrieve records by exact address keys.

The storage layer does **not**:

* Understand geometry
* Interpret axes
* Perform joins or ranking

Geometry is encoded *into the keys themselves*.

Because of this:

* Local file stores work
* Embedded databases work
* Distributed KV stores work
* Cloud-native storage works

As long as storage supports deterministic key access, it can support NAM.

---

## Why This Works with KV Stores

Key-value systems require:

* Exact key equality
* Known partitions
* Predictable access patterns

NAM satisfies this by:

* Expanding queries into **sets of geometric probes**
* Explicitly enumerating points, lines, planes, and cubes
* Probing only known addresses

There is no guessing at runtime.

---

## Query Widening (Not Guessing)

When a query does not match a point:

* NAM does not “relax similarity thresholds”
* NAM does not “search harder”

Instead, it **widens geometry**:

```
point → line → plane → cube
```

Each step is:

* Explicit
* Ordered
* Deterministic

This is how NAM preserves control without losing recall.

---

## Why Ingest and Query Must Align

For geometric retrieval to work:

* Ingest must populate:

  * Specific axes **and**
  * Corresponding wildcard axes
* Query must probe:

  * The same address families

If ingest populates only points and queries probe planes, retrieval fails.

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
* Geometry is composed of points, lines, planes, and cubes
* `__null__` enables safe generalization
* Storage is a passive geometric substrate
* Retrieval is navigation, not ranking

If this document makes sense, the rest of NAM will too.

→ See also: [Addressing Model](ADDRESSING_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)