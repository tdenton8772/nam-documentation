# Geometric Retrieval in NAM

*Points, lines, planes, and cubes*

This document explains how the **address path** retrieves records: by navigating regions of a structured key space rather than ranking by similarity.

The address path is one of three access paths in NAM's converged index. The other two are an approximate vector path ([Vector Search](VECTOR_SEARCH.md)) and a graph path ([Graph Index](GRAPH_INDEX.md)). A query may run any combination of them; see [Query Model](QUERY_MODEL.md). This document covers only the address path, which is the deterministic core.

---

## The Core Idea

The address path does not ask "which records are closest to this query?"

It asks "which records occupy this region of semantic space?"

That region may be a **point**, a **line**, a **plane**, or a **cube** (volume). Retrieval is navigation, not comparison.

---

## The Address Space

Every record is stored at one or more addresses:

```
tenant | partition | TYPE | entity | x | y | z
```

* `tenant`, `partition`, `TYPE` (a BFO/CCO class) and `entity` form the partition key: *what kind of thing, about which entity, in which scope*.
* `x` (attribute), `y` (affordance) and `z` (context) are the semantic axes.

Each axis holds either an LCA code such as `lca:42:17` (coarse code 42, fine code 17) or the empty code `lca:0:0`, meaning "not constrained along this axis." See [Addressing Model](ADDRESSING_MODEL.md).

---

## Why Geometry Exists at All

Most real text is incomplete. A sentence rarely states a quality, an action and a circumstance for every entity it mentions. If every axis had to be populated, most records would be unusable.

NAM treats an unpopulated axis as openness, and models that openness geometrically.

---

## Points, Lines, Planes, Cubes

The geometry of a match is determined by how many axes have to agree.

| Geometry | Axes that must agree | Meaning |
|---|---|---|
| **Point** | all specified axes | A specific fact: this entity, with this quality, this action, these circumstances |
| **Line** | all but one | One axis left open |
| **Plane** | all but two | Two axes left open |
| **Cube** | none (partition only) | Anything stored about this entity under this type |

A record can itself occupy a line, plane or cube: an address with empty axes (`lca:0:0`) is open along them. A query widens from point toward cube by requiring fewer axes to agree.

---

## How a Region Becomes a Storage Operation

### The covering index answers any set of specified axes

Addresses are stored so that whichever axes a query specifies are a **direct match** (exact or by prefix) on something stored (see [Addressing Model](ADDRESSING_MODEL.md)) — no wildcard copies, no secondary index, nothing derived at read time.

* On the **reference deployment**, each partition's addresses are stored together, and a planning tier is read in a fixed number of batched round trips, however many addresses a partition holds.
* In the **rotations layout**, the planner picks the rotation that puts the specified axes at the front of the key, and the probe is a **prefix scan**.

Everything stored for an address shares its partition key, so it lives in the same storage partition, and a probe of one partition key is a single-partition operation.

### Coarse-code prefixes

Probes are built on **coarse codes**. A prefix such as `…|lca:42:` matches every fine code under coarse code 42 on the leading axis. This gives each probe a small semantic neighbourhood rather than an exact string.

The leading axis is expanded to its exact coarse code plus its nearest codebook neighbours, and each becomes a scan prefix. The remaining specified axes are checked by **post-filtering** the scanned rows against their own acceptable coarse codes (exact plus neighbours).

### Empty codes are neutral

When filtering, a stored empty code (`lca:0:0`) on an axis the query specified neither matches nor disqualifies: it lowers the number of axes that must agree, but does not add to the row's match score. At point level, an empty leading code is excluded, so point matches always carry real signal on the leading axis.

Each surviving row is annotated with a match score and the geometry it matched at.

---

## Progressive Widening

The address path runs geometry levels in order, from tight to broad:

```
level 0: point   all specified axes must agree
level 1: line    one fewer must agree
level 2: plane   two fewer must agree
level 3: cube    partition-only scan, no axis filter
```

At each level the planner builds scans and the runner executes them, deduplicating records across levels. The path stops as soon as it has the requested number of distinct records, or after the broadest level the request allows. The request's `fan_out` caps how far it widens.

Within a level, the rotations layout pages each scan with a cursor until enough distinct records are found or the level's fetch budget is spent; the reference deployment's layout reads each planning tier in a fixed number of batched round trips. A large partition does not starve smaller ones: the first fetch of each known partition takes a fair share of the limit.

Every step is explicit, ordered and deterministic: the same query against the same data plans the same scans and stops at the same point.

---

## Where the TYPE Segment Fits

The type segment is part of the partition key, so a scan always targets one class. The planner decides which classes to scan: first the classes the graph index records for each query entity, then ontology tiers grouped by BFO branch, with a probe of the unclassified class (BFO:0000001) as a safety net. That ordering is described in [Query Model](QUERY_MODEL.md).

---

## Storage as a Geometric Substrate

The storage layer needs only two kinds of operation:

1. Blind appends (an entry or a record pointer under a key)
2. Batched key reads and prefix scans

It does not understand geometry, interpret axes, join, or rank. Geometry is encoded into what is stored, and the storage layout makes every combination of axes an exact or prefix match.

---

## Why Ingest and Query Must Align

For the address path to work, ingest and query must use the same codec, the same storage layout, the same entity normalization, the same ontology class set, and the same tenant and partition scope. If any of these differ, scans land in the wrong partition or match nothing. That failure is visible: the `access_paths` trace reports the address path's contribution as zero.

---

## What the Address Path Is and Is Not

It is:

* Deterministic and reproducible
* Explainable — every result came from a named scan of a named region
* Tolerant of partial facts and vague queries, through empty axes and widening

It is not:

* A similarity ranking over the corpus
* Able to find meaning that ingest never wrote as an address

The second limit is why NAM also runs a vector path. Where the address path cannot reach a relevant record, the vector path may; fusion keeps the address path's ordering first and records which path found each row.

---

## Summary

* The address path stores meaning as coordinates and retrieves by region.
* Regions are points, lines, planes and cubes, matched by how many axes must agree.
* The storage layout makes any set of specified axes an exact or prefix match; coarse codes plus neighbours give each probe a neighbourhood; remaining axes are post-filtered.
* Widening runs point → line → plane → cube until the limit is met.
* Storage only appends, reads keys and scans prefixes.

→ See also: [Addressing Model](ADDRESSING_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Vector Search](VECTOR_SEARCH.md) | [Graph Index](GRAPH_INDEX.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)
