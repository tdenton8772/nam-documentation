# Deployment Architecture

*How NAM is deployed and operated*

This document describes NAM's deployment topology, service layout, and operational model.

---

## Deployment Model

NAM is deployed as a set of containerized services orchestrated by Kubernetes. All deployment is managed through a single templated manifest system (Helm chart) — there are no raw manifests or ad-hoc deployments.

A single deployment command creates, upgrades, or replaces the entire system. Environment-specific configurations (development, staging, production) are handled through value overrides, not separate templates.

---

## Service Topology

A NAM deployment consists of the following services:

### Ingest Pods (StatefulSet)

Each ingest pod is a self-contained processing unit that runs the full ingestion pipeline:

* **CDC adapter** — streams mutations from the data service
* **NLP worker** — rule-based linguistic analysis
* **Ontology worker** — semantic type classification
* **Encoder head workers** — entity (6 workers), attribute (LMDB L2 cache), affordance (LMDB L2 cache), context
* **Addressing workers (4)** — entity-anchored bundling, LCA encoding (ONNX Runtime), async WritePipeline
* **LMDB sync workers** — attribute-lmdb-sync and affordance-lmdb-sync relay NATS writes to per-node LMDB
* **Pod-local message bus** — stage-to-stage communication within the pod

Ingest pods are stateful: each pod owns a set of data partitions via CAS-based leases. The StatefulSet ensures stable pod identities for lease management.

**Scaling:** Add or remove replicas. Partitions are automatically redistributed.

### Query Service (Deployment)

A stateless HTTP service that handles:

* Semantic queries (exploratory and affordance modes)
* Authentication and user management
* System administration APIs
* Document browsing

The query service is the only externally-facing API. It connects to the data service for address lookups and document retrieval.

**Scaling:** Standard horizontal scaling. All replicas are equivalent.

### Data Service (StatefulSet)

The distributed key-value store that persists all NAM data:

* Source documents (raw records)
* Semantic addresses (the address index)
* Entity and vocabulary caches (session state)
* Coordination documents (leases, checkpoints)

The data service uses a custom storage engine backed by S3-compatible object storage for durability. On-demand caching enables fast cold starts.

**Scaling:** Replicas provide redundancy. Data is partitioned across vbuckets.

### Supervisor (Deployment)

A single-instance service that handles cluster lifecycle and automated maintenance:

* Waits for the data service to become available
* Creates storage structures (buckets) during bootstrap
* Monitors cluster health on a recurring basis
* Coordinates pipeline resets and data loading
* Gates readiness for downstream services
* Runs the **task framework** — a pluggable system for scheduled and manual maintenance tasks

#### Task Framework

The supervisor includes an auto-discovery task framework (inspired by Apache Pinot's Minion architecture) that manages operational tasks. Tasks are defined as Generator + Executor pairs and are automatically discovered at startup.

| Task | Mode | Description |
|------|------|-------------|
| `compaction` | Scheduled | In-memory compaction on live data service (local SSTs only) |
| `flush` | Scheduled | Periodic memtable flush to ensure data reaches S3 |
| `s3_compaction` | Manual | Online S3 SST compaction — merges accumulated SSTs without downtime |
| `s3_clear` | Manual | Wipes an S3 prefix (destructive, for development use) |
| `wiki_load` | Manual | Loads Wikipedia test data via a Kubernetes Job |
| `session_seed` | Manual | Pre-seeds session store caches |
| `cluster_reset` | Manual | Orchestrates full pipeline reset (flush, clear, restart) |

Tasks support cron scheduling, retry with exponential backoff, configurable timeouts, and execution history. All tasks are managed via the supervisor's REST API and can be triggered from the frontend dashboard.

**S3 compaction** is the most significant maintenance task. Over time, the storage engine accumulates many small SST files in object storage (the result of periodic flushes). S3 compaction runs a dedicated Kubernetes Job that opens the database directly from S3, runs a full merge compaction, and deletes orphaned SST files — all while the data service remains live and serving traffic. An age-based safety filter ensures only SSTs written before the compaction began are eligible for cleanup.

### Frontend (Deployment, optional)

A web-based dashboard providing:

* System health overview
* Query interface
* Document browser
* User and access management
* Encoder head monitoring
* Data replication management

The frontend communicates exclusively through the query service API.

### KV Proxy (DaemonSet, optional)

A JWT-authenticating TCP proxy that enables direct key-value access for SDK clients. Runs on each node to provide low-latency access.

### LMDB Cache DaemonSet (optional)

A per-node shared memory cache layer that populates three LMDB databases on tmpfs from the session store via DCP:

* **Entity LMDB** (`/dev/shm/nam-entity-lmdb`) — surface form and candidate mappings (`surf::`/`cand::` keys)
* **Attribute LMDB** (`/dev/shm/nam-attribute-lmdb`) — attribute category cache (`acat::` keys)
* **Affordance LMDB** (`/dev/shm/nam-affordance-lmdb`) — affordance category cache (`vcat::` keys)

Entity, attribute, and affordance workers mount their respective LMDB databases read-only for ~1-2μs zero-copy lookups, with fallback to the session store KV for L3 misses.

---

## Network Architecture

```
External Traffic
       |
       v
  [ Query Service ]  <-- JWT-authenticated HTTP API
  [ Frontend ]       <-- Browser-based dashboard
  [ KV Proxy ]       <-- JWT-authenticated TCP (SDK access)
       |
       | (cluster-internal only below this line)
       v
  [ Ingest Pods ]    <-- CDC streaming, pipeline processing
  [ Supervisor ]     <-- Lifecycle management
  [ Data Service ]   <-- KV storage, S3 replication
```

### External boundaries

* Only the query service, frontend, and KV proxy accept external connections
* All external access requires authentication (JWT tokens)
* Internal services are not exposed to external networks

### Internal communication

* Ingest pods communicate with the data service for CDC streaming and storage writes
* The query service communicates with the data service for address lookups
* The supervisor communicates with the data service for health monitoring
* Pod-internal communication uses the local message bus (no network hops)

---

## Container Layout (Per Ingest Pod)

Each ingest pod runs multiple containers in a sidecar pattern:

```
+----------------------------------------------------------+
|  Ingest Pod                                              |
|                                                          |
|  +----------+  +-----+  +----------+                    |
|  | CDC      |  | NLP |  | Ontology |                    |
|  | Adapter  |  |     |  |          |                    |
|  +----+-----+  +--+--+  +----+-----+                    |
|       |           |           |                          |
|       v           v           v                          |
|  +------------------------------------------+           |
|  |         Pod-Local Message Bus             |           |
|  +------------------------------------------+           |
|       |           |           |          |               |
|       v           v           v          v               |
|  +--------+ +--------+ +--------+ +--------+            |
|  | Entity | | Entity | | Attrib | | Afford |            |
|  | Head 1 | | Head N | | Head   | | Head   |            |
|  +--------+ +--------+ +--------+ +--------+            |
|       |           |           |          |               |
|       +-----+-----+-----+----+----------+               |
|             |                                            |
|             v                                            |
|       +------------+                                     |
|       | Addressing |  --> Background writes to           |
|       | Worker     |      Data Service                   |
|       +------------+                                     |
|                                                          |
+----------------------------------------------------------+
```

---

## Storage Layout

```
+-------------------+
|   Data Service    |
|                   |
|  +-------------+  |       +-----------+
|  | Source Store |--+------>|    S3     |
|  | (default)   |  |       | (durable) |
|  +-------------+  |       +-----------+
|                   |            ^
|  +-------------+  |            |
|  | Address     |--+------------+
|  | Store (nam) |  |
|  +-------------+  |
|                   |
|  +-------------+  |
|  | Session     |--+------------> S3
|  | Store       |  |
|  +-------------+  |
|                   |
|  +-------------+  |
|  | Coordination|  |  (ephemeral, not S3-backed)
|  | Store       |  |
|  +-------------+  |
+-------------------+
```

---

## Environment Configurations

NAM supports multiple deployment targets through value overrides:

| Configuration | Purpose | Typical Topology |
|---------------|---------|------------------|
| Local development | Single-node testing | 1 data, 1-2 ingest, 1 query |
| Cloud production | Multi-node deployment | 1-3 data, 2-4 ingest, 1-2 query |

Environment-specific settings include:

* Replica counts for each service
* Resource limits (CPU, memory)
* Storage class and volume sizes
* External access (NodePort, LoadBalancer, Ingress)
* S3 configuration (bucket, region, credentials)
* Authentication secrets
* Write throttling parameters (writer thread count, inter-batch delay, timeouts)

---

## Operational Lifecycle

### Bootstrap

1. Data service starts and initializes storage
2. Supervisor waits for data service readiness
3. Supervisor creates storage structures (buckets, indexes)
4. Supervisor signals readiness
5. Ingest pods start, claim partition leases, begin CDC streaming
6. Query service starts, connects to data service

### Upgrade

1. New container images are built and published
2. Helm upgrade applies the new configuration
3. Kubernetes performs rolling updates per service
4. Ingest pods shed leases before termination, reclaim on restart
5. No downtime for queries during rolling updates

### Reset

1. Admin triggers reset via supervisor API
2. Data stores are flushed (except session store, which preserves entity caches)
3. Ingest pods are restarted to re-establish leases
4. Pipeline resumes from the beginning of the CDC stream

### Maintenance

The supervisor's task framework automates recurring maintenance:

1. **Scheduled tasks** (compaction, flush) run on configurable intervals or cron expressions
2. **Manual tasks** (S3 compaction, cluster reset) are triggered via the admin API or frontend
3. Task execution is tracked with history, retry counts, and error reporting
4. Environment variable overrides allow per-task tuning without redeployment

Tasks can be monitored and triggered through:
* `GET /v1/admin/tasks` — list all task types and their status
* `POST /v1/admin/tasks/{type}` — trigger a manual task execution
* `GET /v1/admin/tasks/{type}/history` — view execution history

### Recovery

See [Data Persistence — Recovery Model](DATA_PERSISTENCE.md) for the full recovery matrix.

---

## Monitoring and Observability

NAM exposes health and status endpoints on every service:

* **Health checks** — liveness and readiness probes for Kubernetes
* **Pipeline metrics** — throughput, error rates, queue depths
* **Partition ownership** — which pod owns which partitions
* **DCP health** — CDC adapter status, failure counts, exhaustion state
* **System status** — aggregated view from the query service admin API

### Health Recovery

* DCP adapter failures trigger automatic pod restart via exhaustion detection
* Container-level liveness probes serve as a safety net
* Lease expiry provides automatic failover for crashed pods
* The supervisor monitors and re-joins dropped data service nodes

---

## Security Boundaries

| Boundary | Protection |
|----------|-----------|
| External to cluster | JWT authentication on all APIs |
| Cluster to data service | SASL bucket authentication |
| Pod to pod | Network isolation (no cross-pod pipeline traffic) |
| S3 access | IAM credentials (mounted as secrets) |
| Admin operations | Role-based access control (admin JWT claim) |
| KV proxy | JWT validation before forwarding |

Secrets are managed as Kubernetes secrets, never embedded in images or templates.

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Summary

NAM's deployment architecture is:

* **Templated** — single Helm chart for all environments
* **Containerized** — all services run as containers on Kubernetes
* **Per-pod isolated** — pipeline processing stays within a pod
* **Self-healing** — lease expiry, health monitoring, and liveness probes handle failures
* **S3-backed** — durable storage without local disk dependencies
* **Externally authenticated** — JWT on all public boundaries
* **Horizontally scalable** — add pods to increase throughput

> See also: [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Data Persistence](DATA_PERSISTENCE.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [API Reference](API_REFERENCE.md)
