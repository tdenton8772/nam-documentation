# Data Persistence Model

*How NAM stores and recovers state*

This document describes how NAM's storage layer persists data, how it recovers from failures, and why the persistence model is designed this way.

---

## Core Principle: Derived State Is Rebuildable

All of NAM's indexes are **derived** from source records:

* The address index is rebuilt by re-running the address pipeline over the source store
* The vector index is rebuilt by re-embedding the source store
* The graph index is rebuilt from the address index

The source store is the system of record. NAM's indexes are never authoritative.

Two pieces of state are not derived from source records and are treated with more care: the **head stores** (the vocabulary ingest has learned, which make a rebuild faster and more consistent) and the **coordination data** (leases, checkpoints and user accounts).

---

## The Data Service

NAM runs on its own distributed key-value store. It provides what NAM needs and nothing more:

* Keyed reads, writes and appends
* Ordered prefix scans
* A per-partition, resumable change stream
* Partitioning into a fixed number of vbuckets per bucket, spread across data nodes, with replicas

It runs as a multi-node tier. The reference deployment uses three data nodes with one replica of every vbucket.

---

## Buckets

| Bucket | Purpose | Partitions | Engine |
|---|---|---|---|
| `main` | Source records | 1024 | RocksDB-Cloud: one column family per vbucket |
| `session` | Head stores: entity, attribute, affordance and context vocabularies | 64 | RocksDB-Cloud |
| `metrics` | Leases, change-stream checkpoints, pipeline counters, user accounts | 64 | RocksDB-Cloud |
| `nam` | Semantic addresses | 64 | Raw RocksDB |
| `graph` | Graph index records | 64 | Raw RocksDB |
| `vector` | Vector index records | 64 | Raw RocksDB |

### Two storage engines

**RocksDB-Cloud store** (`main`, `session`, `metrics`). Each vbucket has its own column family in a shared database. An in-memory hash table holds resident items, and a by-sequence index supports a full change stream. This engine suits document-shaped data that is read by key and streamed.

**Raw RocksDB store** (`nam`, `graph`, `vector`). All vbuckets share one data column family; keys carry a short vbucket prefix. There is no in-memory hash table: writes go straight to RocksDB, and reads use prefix bloom filters. A separate by-sequence index lets the change stream deliver each vbucket in order, which is what the graph builder consumes. This engine suits NAM's index workload: very high write volume, key-only or small values, and prefix-scan reads.

Address data, and graph data on the reference deployment, are written as blind appends (RocksDB merges) rather than read-modify-write. Readers de-duplicate repeats, and compaction of the address index keeps one copy of each.

---

## What Each Bucket Holds

### Source store (`main`)

The original records. Opaque payloads that NAM reads during ingestion and fetches at query time to return results. It is the system of record.

### Address store (`nam`)

On the reference deployment each partition's (tenant, partition, type, entity) addresses are stored together; in the rotations layout the keys are the rotated addresses themselves. Either way, an address's record pointers are a list of lines, each naming a source record pointer and a timestamp. Readers de-duplicate lines by record pointer, so a redelivered write costs space, not correctness.

### Graph index (`graph`)

Record-entity edges in both directions and the ontology classes each entity is stored under (in the rotations layout, key-only records that also include one per non-empty axis combination of each address). Built from the address store.

→ See: [Graph Index](GRAPH_INDEX.md)

### Vector index (`vector`)

Each record's embedding, filed by region, with a pointer back to its source record. Written by vector ingest from the source store's change stream.

→ See: [Vector Search](VECTOR_SEARCH.md)

### Session store (`session`)

The head stores: entity surface forms and identifiers, attribute and affordance categories, context values. They accumulate as ingest runs and act as a **warm cache** across reloads. Losing them does not make the system incorrect, but entity resolution has to rediscover names from scratch, and identifiers learned late are not retroactively applied to addresses already written.

A node-level cache keeps memory-mapped copies of these stores, fed by the `session` change stream.

### Coordination store (`metrics`)

Partition leases, change-stream checkpoints, pipeline counters and user accounts. Leases rebuild themselves; checkpoints and counters can be reset deliberately; **user accounts cannot be re-derived**, so resets preserve this bucket unless a lease reset is explicitly intended.

---

## Object Storage Backing

Buckets are mirrored to **S3-compatible object storage**. The reference deployment runs an in-cluster object store on its own node. The coordination bucket can be configured to stay local, since its contents are either regenerated or small.

The storage engine writes locally first and ships immutable data files (SSTs) and manifests to object storage. On open, a store downloads its manifest and fetches data files **on demand** as they are first read, then caches them locally. A data node therefore becomes operational without first downloading its whole dataset.

---

## Storage Maintenance

### Compaction

RocksDB merges small files into larger ones in the background. For the buckets configured for it, the heavy part of that work is **offloaded**: dedicated remote compaction workers, running away from the data nodes, pick up compaction jobs, read their inputs from object storage, compact them and return the result for the data node to install. Jobs are claimed exclusively, so two workers never run the same job, and a failed remote job aborts that compaction without stopping writes. A controller reads each data node's compaction backlog, requests compaction of what is due, and sizes the worker pool to the backlog; a backstop starts a worker if none is running when compactions are waiting.

### Orphan file collection

Compaction leaves superseded files behind in object storage, and RocksDB-Cloud does not reclaim them by itself. A garbage-collection pass reconciles each bucket's objects against its live manifest and deletes the difference. An object is deleted only if it was already dead in a candidate list recorded at least an hour earlier, so a file that a live writer has just produced is never taken.

### Bucket flush

A flush drops each data column family and creates an empty one under the same name. There is no range delete and no compaction, so a flush costs the same whatever the data size. The dropped files are freed later by a reclaim task, once no reader or writer still holds them. Flushing the data buckets needs no restart of any consumer: change streams carry on.

The `session` bucket is normally not flushed, so the head stores stay warm across reloads.

---

## Recovery Model

### Ingest or query pod restart

* Leases expire and are reclaimed; each change-stream consumer resumes from its checkpoint.
* Records already handed to the durable on-disk queue are on the pod's persistent volume and are processed after restart.
* Node-level head-store caches rebuild from the `session` change stream.

### Data node restart

* Stores reopen from local disk; anything not yet local is fetched from object storage on demand.
* Replicas on other nodes hold copies of the node's vbuckets and can be promoted by failover if the node does not return.

### Loss of a data node's local disk

* Stores reopen from object storage using the last uploaded manifest.
* Data written since the last upload may be lost from that node; replicas on other nodes and re-ingestion cover it.

### Loss of the whole cluster

* Source records, head stores and indexes are recoverable from object storage up to the last upload.
* Coordination state starts fresh; derived indexes can be rebuilt by re-ingesting.

### Loss of the source store

* Indexes cannot be rebuilt without source records. Object storage is the backup for the source store, which is why source records are mirrored there.

---

## Consistency Model

* **Eventual consistency for indexes.** An address becomes queryable shortly after its record is written; the graph index lags the address index; the vector index follows the source store on its own schedule.
* **At-least-once delivery.** Change streams and the durable queues can redeliver. Repeated address and graph lines are de-duplicated on read (and address lines also at compaction); the vector index de-duplicates by record when it scores.
* **Deterministic keys.** The same record produces the same address keys and lines, so re-ingesting it repeats lines rather than adding new ones.

This is sufficient because NAM is an index: what matters is that, once writes have landed, the same query over the same data returns the same results.

---

## What Is Not Persisted

* **Messages between pipeline stages**, where a deployment runs the head stages on the lightweight bus instead of durable streams (the reference deployment uses durable streams): a record lost there is evicted after a timeout and counted as an error.
* **In-process caches**: rebuilt on demand.
* **Node-level head-store caches**: rebuilt from the `session` change stream.

---

## Summary

NAM's persistence model is:

* **Derived**: every index is rebuildable from the source store
* **Purpose-built**: a document store for source and state, a raw prefix-scan store for indexes
* **Object-storage backed**: durable files, fetched on demand
* **Maintained online**: offloaded compaction, orphan collection, constant-cost flush
* **Replicated**: partitions spread across data nodes with replicas
* **Eventually consistent**: deterministic once writes land

The semantic pipeline does not depend on how data is stored, only that it is addressable and scannable.

> See also: [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Addressing Model](ADDRESSING_MODEL.md) | [Deployment Architecture](DEPLOYMENT_ARCHITECTURE.md)
