# Glossary

A quick-reference list of NAM terms. Canonical, longer definitions of the core concepts live in [Terminology](../PHILOSOPHY/TERMINOLOGY.md); the architecture documents linked from each entry explain the mechanisms.

---

## Access Path

One of the three ways a query can reach records: the **address** path, the **vector** path or the **graph** path. Each path is a separate index built from the same source records.

→ [Query Model](../ARCHITECTURE/QUERY_MODEL.md)

## access_paths

The per-query trace returned with every response. For each path it records whether the path ran (`used`, `fallback`, `error`, `skipped`, `not_requested`), why, how long it took, path-specific statistics, and how many results it contributed.

→ [API Reference](../ARCHITECTURE/API_REFERENCE.md)

## Address

A structured semantic location: tenant, partition, type, entity, attribute, affordance and context. Addresses are produced deterministically at ingest and stored in a covering index.

→ [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md)

## Affordance

The axis that records the action defining how a record is navigated (from verbs, with synonym folding). At query time, a recognized affordance can switch the planner from exploratory to affordance mode.

## Axis (plural: Axes)

One of the three coordinate dimensions of an address: **attribute** (x), **affordance** (y) and **context** (z). Each holds an LCA code or the empty coordinate `lca:0:0`.

## BFO / CCO

Basic Formal Ontology (BFO 2020, an ISO/IEC standard) and the Common Core Ontologies (CCO), the upper- and mid-level ontologies NAM uses for the type segment of every address. Classes are written as CURIEs, for example `BFO:0000015` or `cco:ont00001262`.

## BFO:0000001

BFO's root class, *entity*. Assigned when the ontology head cannot support a more specific class. NAM treats it as a classification miss: its rate is tracked, and the planner probes it as a safety net.

## Change Stream (DCP)

The data service's ordered stream of mutations per partition. The ingest pipeline, the vector pipeline, the graph index builder and the per-node LMDB caches are all change-stream consumers with their own checkpoints.

## Cosine Floor

The minimum cosine similarity a vector result must reach. It is set per deployment, a request cannot go below it, and it is a per-model calibration that must be re-derived when the embedding model changes.

→ [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md)

## Covering Index

Storing every address so that any combination of specified axes is a direct match on something stored, with no wildcard copies: on the reference deployment each partition's addresses are stored together under that guarantee; the rotations layout stores each cyclic rotation of an address's axes.

## Encoder Head

A pluggable component that extracts one semantic signal from parsed text: ontology, entity, attribute, affordance or context. Heads load offline-trained artifacts and behave deterministically at runtime. Each fills a head slot.

## Entity Bundle

An entity together with the attributes, affordances and contexts the dependency parse links to it. Addresses are built per bundle, which bounds address count by the structure of the text.

## Entity Hash

The entity segment of an address: a hash of the entity's normalized name, independent of its type, so ingest and query always agree on identity.

## Exploratory Mode / Affordance Mode

The two address-path planning modes. Exploratory mode (the default) explores type tiers in order; affordance mode narrows to the tiers associated with a recognized affordance.

## Fusion

Combining the results of the access paths that ran into one list: de-duplicated by record pointer, address results first, and records that are themselves a resolved entity moved to the front.

## Geometry (Point, Line, Plane, Cube)

How specific an address-path probe is: all three axes specified (point), two (line), one (plane) or none (cube). Queries widen progressively from point toward cube.

## Graph Index

Records that link source records to the entities they mention, and entities to the ontology classes they are stored under. Built from the address store by the index builder.

→ [Graph Index](../ARCHITECTURE/GRAPH_INDEX.md)

## Head Slot

A named dimension of a record (ontology, entity, attribute, affordance, context, or an added facet) that one selected encoder-head implementation fills. The selection lives in a manifest that ingest and query share; a head that keeps no state can be swapped on a running cluster. Implementations developed outside NAM's own tree are published in the open-source `nam-heads` package.

## Index Builder

The sharded change-stream consumer of the address store that writes the graph index. Each shard owns a fixed subset of partitions. It is never on the ingest acknowledgement path, so the graph can lag the address store.

## LCA (Learned Codec for Addressing)

A character-level CNN with two-stage residual vector quantization that maps an axis value to a code pair `lca:coarse:fine`. Similar strings map to nearby codes, which enables codebook-neighbour fan-out at query time.

## lca:0:0

The empty (non-specific) coordinate: the axis is not constrained. Records with empty axes occupy lines, planes or cubes rather than points.

## Lease

A time-bounded claim on a source partition, stored as a document in the coordination bucket. Ingest pods divide work with leases and take over a dead pod's partitions when its leases expire.

## LMDB Cache

A per-node, memory-mapped copy of the head stores, kept current by a change-stream follower of the session bucket. Heads read it locally instead of crossing the network for every lookup.

## Record

An ingested unit of information in the source bucket. Every index entry points back to its record.

## Rotation

In the rotations layout, one of the three cyclic orderings of an address's axes (`xyz`, `yzx`, `zxy`), stored as separate keys to form the covering index.

## Session Store

The bucket that holds the head stores (entities and surface forms, attribute, affordance and context vocabularies). It accumulates across loads and acts as a warm cache; it is not flushed when the derived indexes are reset.

## sources

The field on every fused result row listing which access paths found that record.

## Task Framework

The supervisor's system of pluggable maintenance tasks (loading, flush, compaction, healing, ingest scaling and others) with schedules, retries, timeouts and history. Destructive tasks run only by hand.

## Vector Index

The index behind the vector path: each record's embedding is filed into a region of a partition trained on the data, and a query reads the regions near it and ranks candidates by cosine similarity. Approximate by contract.

→ [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md)

## Write-Ahead Log (WAL)

The durable, file-backed JetStream stream each ingest pod uses to hold handed-off records until their writes land. It rejects new work when full and never evicts unacknowledged records.

---

→ See also: [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [FAQ](FAQ.md)
