# Current State

This document describes the **current state of NAM** — what exists, what works, and what is exercised in running deployments.

It is factual. It is not a roadmap, pitch, or vision statement.

> See also: [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Near-Term Roadmap](NEAR_TERM.md)

---

## Project Maturity

NAM is in an **active production-pilot phase**.

* The full architecture is implemented and exercised end to end against real corpora
* Ingest, query, indexing and storage all scale horizontally
* The data tier runs as a multi-node cluster with one replica per partition
* The query API is authenticated; cluster lifecycle and maintenance are automated
* Deployment is a single Helm chart

---

## The Converged Index

NAM serves one query endpoint over three access paths, all built from the same source records.

### Address path (the core)

* Each record is placed at semantic addresses of the form tenant, partition, type, entity, attribute, affordance, context
* The type is a BFO/CCO class; the entity is a hash of its normalized name; the three axes are LCA codes (`lca:coarse:fine`), with `lca:0:0` for an empty axis
* Addresses are stored in a covering layout, so any combination of specified axes is a direct match on something stored. The reference deployment stores each partition's addresses together and reads a planning tier in a fixed number of batched round trips; the rotations layout (three cyclic rotations per address) is also supported
* Queries resolve entities, plan probes by type tier, fan out to neighbouring codebook entries, and read the planned regions until enough distinct records are found
* Geometry widens progressively from point to line to plane to cube

> See: [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md) | [Query Model](../ARCHITECTURE/QUERY_MODEL.md)

### Vector path

* Records and queries are embedded locally by a text-embedding model; ingest and query embed the same way, and the services refuse to start on a mismatched model
* Each record's embedding is filed into a region of a partition trained on the data; a query reads only the regions near it and ranks the candidates by cosine similarity
* Queries return the top-K records above a deployment-wide cosine floor, which a request cannot lower
* A single index writer files the records embedded by every vector ingest pod
* A read the vector path could not complete is reported as an error, never as an empty answer

> See: [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md)

### Graph path

* The graph index holds record-to-entity edges in both directions and the ontology classes each entity is stored under
* It is written by a sharded index builder that follows the address store's change stream, with blind, idempotent appends
* Queries walk record co-occurrence from up to three resolved entities in parallel and merge the walks

> See: [Graph Index](../ARCHITECTURE/GRAPH_INDEX.md)

### Fusion and tracing

* A routing policy chooses paths from cheap signals: address and vector always, plus graph when the query resolves an entity. Callers can pin paths explicitly
* A thin-result fallback consults the vector path when a routing policy has not already run it and the address path returns too few results relative to the requested limit
* Results are de-duplicated by record pointer; each row lists its `sources`; vector rows carry a `cosine`
* Every response carries an `access_paths` trace with each path's status, reason, timing, statistics and contribution
* Storage trouble is reported, not hidden: partial results are flagged with the sources that failed, and a query that could not read anything returns a service-unavailable status

---

## Ingest Pipeline

* Source records are written to the source bucket; a change-stream (DCP) adapter in each ingest pod streams them out
* Each ingest pod runs its own message bus. Records are handed off through a durable **JetStream write-ahead log** on a per-pod volume, which rejects new work when full rather than evicting; on the reference deployment every hop between stages is durable
* Stages: rule-based NLP, then the ontology head (BFO/CCO class), then the entity head and a deterministic sentence resolver, then the attribute, affordance and context heads in parallel, then addressing (bundle, LCA build, write)
* A batch is acknowledged only after its address writes land. A failed batch is redelivered; a batch that exhausts its retries is counted as an error
* Every stage publishes its queue depth; the change-stream adapter pauses when any stage nears capacity
* Ingest pods divide the source partitions using time-bounded lease documents in the coordination bucket; leases fail over automatically when a pod dies
* The supervisor sizes the ingest tier from its backlog and an autoscaler applies the count; the reference deployment keeps at least one ingest pod, and compute nodes are added and removed as the pods need them
* Separate change-stream consumers build the vector index and the graph index, each with its own leases or shards and its own checkpoints

> See: [Pipeline Architecture](../ARCHITECTURE/PIPELINE_ARCHITECTURE.md) | [Ingestion Model](../ARCHITECTURE/INGESTION_MODEL.md)

---

## Encoder Heads

* **Ontology** — classifies each sentence against a trained codebook of BFO/CCO classes, backs off to an ancestor class when uncertain and to `BFO:0000001` when nothing is supported; the record's class is the vote of its sentences. The abstention rate is tracked as a quality metric
* **Entity** — extracts candidates, normalizes them, and resolves them to hash-based identifiers through the shared entity store; generic nouns are rejected
* **Attribute** — extracts qualities asserted of entities, linked through the dependency parse
* **Affordance** — extracts the actions that define how a record is navigated, with synonym folding
* **Context** — extracts particular evidence: temporal paths, gazetteer places, institutions and fields, each tied to the sentences it came from
* Head stores live in the session bucket and are cached per node in memory-mapped LMDB files, kept current by a per-node change-stream follower
* Each head fills a slot in a manifest that ingest and query share; an admin can swap a stateless head's implementation on a running cluster. The ontology heads come from the open-source `nam-heads` package

---

## Data Service

* NAM's own distributed key-value store with a change-stream (DCP) protocol and prefix scans
* Two storage engines: an object-storage-backed RocksDB engine with one column family per partition for the source, session and coordination buckets; and a raw RocksDB engine with a single shared column family and prefix bloom filters for the address, graph and vector buckets
* Data files are mirrored to S3-compatible object storage (an in-cluster MinIO), with on-demand caching on open
* Compaction can be offloaded to remote compaction workers, scheduled by a controller from each data node's backlog; an orphan-file garbage collector reclaims object storage that the engine no longer references
* Bucket flush is a column-family swap and needs no service restart

> See: [Data Persistence](../ARCHITECTURE/DATA_PERSISTENCE.md)

---

## Operations

* **Helm is the only deployment mechanism.** Environments differ only by values files
* **Supervisor** — waits for the data tier, verifies buckets, bootstraps the admin user, gates readiness, and runs the **task framework**
* **Task framework** — pluggable, auto-discovered tasks with schedules, retries, timeouts and history: data loading, flush, compaction (local, controller-driven, bucket-level, object storage, remote dispatch), flush reclaim, data snapshots, session seeding, address healing, head-health reports, ingest scaling, node cleanup, GPU pod reaping, cluster reset and node-tunnel repair. Destructive tasks run only when triggered by hand
* **Frontend** — a web dashboard for system status, pipeline metrics, bucket browsing and record editing, queries, graph exploration, head slots and tasks, which talks to the query API and to the data service's admin REST API
* **KV proxy** — a JWT-authenticating proxy for direct key-value access by external clients, with a Python client for external consumers
* **GPU burst** — vector workers can claim a GPU from an external provider through a shared GPU registry when an embedding backlog builds, and release it after a sustained idle period

> See: [Deployment Architecture](../ARCHITECTURE/DEPLOYMENT_ARCHITECTURE.md)

---

## Security

* JWT authentication with admin and user roles on the query API, with per-user query rate limits, login rate limits and structured audit logs
* Admin JWT required on every supervisor admin route; the supervisor fails closed without a signing secret
* KV proxy validates a JWT before forwarding and then allows only data opcodes; change streams are admin-only
* TLS ingress with managed certificates for the query service
* Secrets held as Kubernetes secrets; secret fields are redacted in task state and history
* NetworkPolicies and encryption at rest are available and opt-in

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Observability

* Health endpoints on every service and a readiness gate for the cluster
* Pipeline counters (enqueued, dequeued, errors), per-stage latencies and queue depths, exposed through the query service's admin API and the dashboard
* Prometheus-format metrics on the query service
* Per-query `access_paths` traces, and an optional debug view of the address path's execution plan
* Head-health reports with axis-health metrics (empty rate, dominance, overlap with the type segment), produced on a schedule and served by the admin API

---

## Summary

Today, NAM is:

* A converged index — address, vector and graph paths behind one endpoint, with attributed, traced results
* Standards-typed — every address carries a BFO/CCO class
* Durable on ingest — write-ahead log, acknowledgement after write, checkpointed change streams
* Rebuildable — every index derives from source records
* Horizontally scalable and operationally automated
* Authenticated at its external boundaries

It is early, and the limitations are documented candidly in [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md).
