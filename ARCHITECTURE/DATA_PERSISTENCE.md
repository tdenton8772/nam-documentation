# Data Persistence Model

*How NAM stores and recovers state*

This document describes how NAM's storage layer persists data, how it recovers from failures, and why the persistence model was designed this way.

---

## Core Principle: Derived State Is Rebuildable

NAM's primary storage — the address index — is entirely **derived** from source data.

This means:

* If the address index is lost, it can be rebuilt by re-ingesting the source data
* The source data is the system of record, never the address index
* NAM never stores anything that cannot be reconstructed

This property is fundamental to NAM's trust model. There is no hidden state that, if lost, cannot be recovered.

---

## Storage Architecture

NAM uses a distributed key-value store as its persistence layer. Data is organized into four logical stores, each with a distinct purpose:

| Store | Purpose | Persistence |
|-------|---------|-------------|
| Source store | Raw records (documents, articles, events) | Persistent, S3-backed |
| Address store | Semantic addresses pointing to source records | Persistent, S3-backed |
| Coordination store | Leases, checkpoints, pipeline counters | Ephemeral (in-memory) |
| Session store | Entity, attribute, affordance, and context caches | Persistent, acts as warm cache |

### Source store

Holds the original records that enter NAM through CDC. These are opaque payloads — NAM reads them during ingestion but does not modify them. The source store is the system of record.

### Address store

Holds the semantic addresses constructed by the pipeline. Each address is a key-value pair:

* **Key**: the deterministic semantic coordinate (partition key + axes)
* **Value**: a pointer back to the source record, plus a timestamp

The address store is the core of NAM's retrieval capability. It is fully rebuildable from the source store.

### Coordination store

Holds lease documents, DCP checkpoints, and pipeline metrics. This data is ephemeral — if lost, leases expire and are re-established, checkpoints restart from the beginning, and counters reset. No semantic data is lost.

### Session store

Holds the entity resolution cache, attribute vocabulary, affordance mappings, and context classifications. This data accumulates over time as the pipeline processes records. It acts as a **warm cache** — losing it means entity resolution must rediscover entities from scratch, but the system remains correct.

The session store is **not flushed during pipeline resets**. This preserves the accumulated entity knowledge across re-ingestion cycles.

---

## Object Storage Backing

The persistent stores (source, address, session) use **object storage (S3-compatible)** as their durable backing layer.

The storage engine writes data locally first, then asynchronously replicates to object storage. On restart, the engine can recover from object storage without local state.

### On-Demand Caching

Rather than downloading all data on startup, the storage engine uses **on-demand caching**:

* On startup, only the metadata manifest is downloaded (small, fast)
* Individual data files are fetched from object storage only when first accessed
* Fetched files are cached locally for subsequent reads
* This enables startup times measured in seconds, not minutes

### Why on-demand?

* **Fast cold start** — new pods become operational quickly
* **Efficient resource use** — only accessed data consumes local storage
* **Graceful scaling** — new replicas do not need to download the entire dataset
* **Cost efficiency** — infrequently accessed data stays in object storage

---

## Bloom Filter Strategy

The storage engine uses a two-tier bloom filter approach:

1. **Per-file bloom filters** — each data file contains its own bloom filter, checked during reads to avoid unnecessary I/O
2. **Engine-level filter bypass** — because data may exist in object storage but not locally, the engine-level bloom filter is intentionally disabled for S3-backed stores

This means:

* Every key lookup that misses the local cache triggers a background fetch from object storage
* The per-file bloom filters handle false-positive filtering efficiently
* No key is incorrectly reported as "not found" simply because it hasn't been cached locally yet

---

## Recovery Model

NAM's recovery model depends on what was lost:

### Pod restart (no data loss)

* Leases expire and are re-established
* DCP checkpoints resume from last saved position
* Pipeline continues from where it left off
* On-demand caching means no lengthy warmup

### Local storage loss (pod rescheduled to new node)

* Metadata manifest is downloaded from object storage
* Data files are fetched on demand as requests arrive
* Session store rebuilds from object storage
* Pipeline resumes after lease re-establishment

### Complete cluster loss

* All state is recoverable from object storage
* Source data, addresses, and session caches are restored
* Coordination state (leases, checkpoints) starts fresh
* Full re-ingestion may be required if checkpoints were not persisted

### Source data loss

* If the source store is lost, addresses cannot be rebuilt
* Object storage serves as the backup for source data
* This is the only truly unrecoverable failure mode

---

## Partition Model

Data is distributed across partitions (vbuckets) using consistent hashing:

* The source store uses a high partition count for even distribution
* The address store uses a lower partition count (addresses are more uniformly distributed)
* The coordination store uses a small partition count (low volume, high churn)
* The session store uses a moderate partition count

Partition counts are fixed at store creation and do not change. This ensures that:

* Partition keys are stable across restarts
* Data distribution is predictable
* Recovery does not require partition remapping

---

## What Is Not Persisted

The following are intentionally ephemeral:

* **Pipeline message bus state** — messages are consumed and discarded
* **In-flight processing state** — if a pod dies mid-processing, the record will be re-streamed via DCP
* **Node-level entity caches** — rebuilt from the session store on startup
* **Lease documents** — re-established on startup via CAS claims

None of these require persistence because:

* The CDC stream is the source of truth for "what needs processing"
* The session store is the source of truth for entity state
* Leases are coordination primitives, not data

---

## Consistency Model

NAM does not require strong consistency for its address store. The consistency model is:

* **Eventual consistency for addresses** — an address may take a short time to become queryable after ingest
* **Strong consistency for entity resolution** — entity IDs are assigned via CAS operations to prevent duplicates
* **At-least-once delivery** — DCP may re-deliver records after a failure; duplicate addresses are idempotent (same key = same value)

This model is sufficient because:

* Address writes are idempotent (writing the same address twice has no effect)
* Query results are deterministic once addresses are visible
* Entity resolution requires consistency only at the point of ID assignment

---

## Summary

NAM's persistence model is:

* **Derived** — all address state is rebuildable from source data
* **S3-backed** — durable storage without local disk dependencies
* **On-demand** — fast startup, efficient resource use
* **Partitioned** — consistent distribution across fixed partition counts
* **Eventually consistent** — sufficient for deterministic retrieval after convergence
* **Recoverable** — from pod restart to full cluster loss, recovery paths are defined

The persistence layer is invisible to the semantic pipeline. Encoder heads, the address builder, and the query planner do not know or care how data is stored — only that it is addressable.

> See also: [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Addressing Model](ADDRESSING_MODEL.md)
