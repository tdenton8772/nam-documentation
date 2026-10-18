# Deployment Architecture

*How NAM is deployed and operated*

This document describes NAM's deployment topology, service layout and operational model.

---

## Deployment Model

NAM runs as a set of containerized services on Kubernetes. **A single Helm chart is the only deployment mechanism**: there are no raw manifests and no ad-hoc deployments. One `helm upgrade --install` creates or upgrades the whole system, and environments differ only by values files (a single-node development cluster, a multi-node production cluster), never by templates.

The chart validates itself at render time. For example, it refuses to render an ingest pipeline without a durable on-disk queue, and refuses bucket memory quotas that do not fit the data nodes.

---

## Service Topology

### Data Service (StatefulSet)

NAM's own distributed key-value store. It holds the source records (`main`), the head stores (`session`), coordination data (`metrics`) and the three indexes (`nam`, `graph`, `vector`). It runs as a multi-node tier with a replica of every vbucket. The reference deployment has three data nodes, pinned one pod per data host, and its data pods are never restarted by a template change: a running data pod picks up a new template only when deliberately replaced.

Storage is mirrored to S3-compatible object storage; the reference deployment runs an in-cluster object store on its own node.

→ See: [Data Persistence](DATA_PERSISTENCE.md)

### Ingest (StatefulSet)

Each ingest pod is a self-contained address pipeline:

* **Change-stream adapter**: streams `main` mutations for the partitions this pod leases
* **Pod-local message bus**, with durable on-disk streams on the pod's persistent volume (at least the work queue in front of addressing; every hop between stages on the reference deployment)
* **NLP workers**
* **Ontology workers**
* **Encoder head workers**: entity, attribute, affordance, context (worker counts per values file)
* **Accumulators**: join head results per record and write complete records to the durable queue
* **Addressing workers**: bundle, code and write addresses to `nam`, acknowledging after write
* **Head-store sync workers**: one single writer per head store

The StatefulSet gives each pod a stable identity for leases and a persistent volume for its queue.

**Scaling:** with autoscaling on, the supervisor sizes the tier from its backlog and the chart's autoscaling objects (KEDA) apply the count; partitions rebalance automatically as pods come and go, and a pod that is shutting down releases its partitions and drains its queues first. The reference deployment keeps at least one ingest pod.

### Vector Ingest (StatefulSet)

A separate pipeline with its own change-stream consumers on `main`, its own durable queue and its own embedding workers, which hand embedded records to the vector index writer. When a backlog builds, workers acquire a GPU from an external GPU registry for burst embedding and release it after a sustained idle period.

### Vector Index Writer (Deployment, single replica)

Receives the embedded records from every vector ingest pod and files them into the `vector` bucket. It is pinned to one replica so the vector index has exactly one writer.

### Graph Index Builder (StatefulSet)

A sharded consumer of the `nam` change stream that writes the `graph` bucket. Each shard owns the partitions whose number modulo the shard count equals its ordinal. It is off the ingest acknowledgement path, resumes from checkpoints, and can be paused and resumed without loss. With autoscaling on, its replica count follows the ingest tier's.

→ See: [Graph Index](GRAPH_INDEX.md)

### Head-Store Cache (DaemonSet)

One per compute node. It follows the `session` bucket's change stream and keeps memory-mapped copies of the four head stores (entity, attribute, affordance, context) on node-local shared memory. Head workers on the node read them read-only.

### Query Service (Deployment)

The external retrieval API: authentication, the `/v1/query` orchestrator, direct vector search and graph walks, and the admin API. Replicas are stateless and equivalent; a replica reports ready once its head-slot manifest and its view of the vector index are loaded. The chart supports autoscaling and a disruption budget. It is published through a TLS ingress.

→ See: [Query Model](QUERY_MODEL.md) | [API Reference](API_REFERENCE.md)

### Supervisor (Deployment, single replica)

Cluster lifecycle and automated maintenance:

* Waits for the data service and verifies its buckets
* Bootstraps the administrative account
* Gates readiness of downstream services
* Monitors health on a recurring loop
* Runs the **task framework**, including the decision of how large the ingest tier should be

Its admin API requires an admin JWT on every administrative route and fails closed when it has no signing secret; only the health and readiness probes are open, and the read-only scaling decision the autoscaler polls takes its own separate token.

### Remote Compaction Workers (Deployment)

Always-on workers that run RocksDB compaction jobs off the data nodes, reading inputs from object storage. Jobs are claimed exclusively. The compaction controller sizes the pool to the data nodes' backlog, and a backstop job covers the case where no worker is running.

### KV Proxy (Deployment, optional)

A JWT-authenticating TCP proxy in front of the data service's key-value port, for external clients that need direct record access. After the handshake it admits only data operations; administrative and streaming operations are refused or restricted to admins.

### Frontend (Deployment, optional)

A web dashboard: system health, pipelines, bucket browser and record editor, query workbench, graph explorer, head-slot management, task management and user administration. It talks to the query service, and to the data service's administrative REST API for bucket and record management.

### Node Tunnel Healer (DaemonSet, optional)

Repairs a known Kubernetes agent failure in which a node reports Ready while the control plane can no longer reach it. A supervisor task detects and labels the node; this per-node pod performs the restart, with guards against restart loops.

### Compute Autoscaling (optional)

On the reference deployment a cluster autoscaler adds compute nodes when pods cannot be placed and removes nodes that have emptied, and a supervisor task removes the leftover node records of servers that are gone. Together with elastic ingest, compute follows the backlog.

---

## Task Framework

The supervisor discovers task plugins at startup. Each task has a schedule (interval or cron), retries with backoff, a timeout and an execution history. Runtime overrides can change only scheduling, timeout and retry fields.

| Task | Mode | Purpose |
|---|---|---|
| `wiki_load` | Manual | Loads a test corpus into `main` through a Kubernetes Job that also monitors the pipeline and reports |
| `flush` | Manual, destructive | Flushes chosen buckets (a constant-cost column-family swap) |
| `cluster_reset` | Manual, destructive | Orchestrates a full reset of pipeline state |
| `s3_clear` | Manual, destructive | Deletes all objects in the object-storage bucket (development use) |
| `address_heal` | Manual, destructive | Re-resolves entity identifiers in stored addresses against the current entity store |
| `data_snapshot` | Manual | Copies the data tier, at a point in time, to its own prefix in object storage; writes nothing else |
| `flush_reclaim` | Scheduled | Frees the column families earlier flushes dropped |
| `compaction` | Scheduled | Merges small data files on the data nodes, for buckets not compacted remotely |
| `compaction_controller` | Scheduled | Requests compaction of what each data node reports as due, and sizes the remote compaction workers |
| `compaction_<bucket>` | Per bucket | Remote compaction of that bucket; manual where the compaction controller runs, as on the reference deployment |
| `rc_dispatch` | Scheduled | Starts a bucket's remote compaction worker when compactions are waiting and no worker is running |
| `catalog_cache` | Scheduled | Keeps a copy of each data node's compaction catalog on the supervisor's volume |
| `s3_compaction` | Scheduled | Orphan-file collection in object storage for every data node (online compaction on request) |
| `session_seed` | Scheduled | Writes head-store seed vocabulary to `session` |
| `head_health` | Scheduled, read-only | Samples stored addresses and writes a collapse report per encoder head |
| `ingest_scaler` | Scheduled | Decides the ingest tier's and graph builder's replica counts from the backlog (with autoscaling on) |
| `node_cleanup` | Scheduled | Removes the leftover node records of compute servers that no longer exist (off unless enabled) |
| `embed_pod_reaper` | Scheduled | Terminates GPU embedding pods whose owner has died |
| `node_tunnel` | Scheduled | Detects nodes whose control-plane tunnel has silently failed |

**Destructive tasks run only when triggered by hand**: never on a schedule, never by automatic retry, and the configuration API cannot change that. Task state and history never contain credentials.

Tasks are managed through the supervisor API (also proxied by the query service for admins):

* `GET /v1/admin/tasks`: every task's status
* `POST /v1/admin/tasks/{type}`: trigger a run (refused while a run of that type is executing)
* `GET /v1/admin/tasks/{type}/history`: execution history
* `POST /v1/admin/tasks/{type}/config`: override scheduling, timeout and retry fields

---

## Network Architecture

```
External traffic
       |
       v
  [ Query Service ]  <-- TLS ingress, JWT on every non-health route
  [ Frontend ]       <-- browser dashboard (query API, data-service admin API)
  [ KV Proxy ]       <-- JWT-authenticated TCP, data operations only
       |
       | (cluster-internal below this line)
       v
  [ Ingest ] [ Vector Ingest ] [ Vector Index Writer ] [ Graph Builder ]
  [ Supervisor ] [ Remote Compaction ] [ Head-Store Cache ]
  [ Data Service ] ----> [ Object Storage ]
```

* Only the query service, frontend and KV proxy accept external connections, and all require authentication.
* Internal services reach the data service directly with bucket-scoped authentication.
* Pipeline stages within an ingest pod talk over the pod-local bus.
* The chart includes ingress network policies for the data service, supervisor, ingest and query that admit only enumerated clients; they are opt-in per deployment.
* Encryption at rest for data volumes is available as an opt-in storage class.

---

## Ingest Pod Layout

```
+---------------------------------------------------------------------+
|  Ingest pod                                                         |
|                                                                     |
|  [ Change-stream adapter ]                                          |
|            |                                                        |
|            v                                                        |
|  +---------------------------------------------------------------+  |
|  |               Pod-local message bus                           |  |
|  +---------------------------------------------------------------+  |
|      |          |           |          |           |         |      |
|      v          v           v          v           v         v      |
|   [ NLP ] [ Ontology ] [ Entity xN ] [ Attribute ] [ Affordance ]   |
|                                                   [ Context ]       |
|      |                                                              |
|      v                                                              |
|  [ Accumulators ] --> [ Durable queue (persistent volume) ]         |
|                               |                                     |
|                               v                                     |
|                     [ Addressing workers ] --> nam                  |
|                                                                     |
|  [ Head-store sync x4 ] --> session                                 |
+---------------------------------------------------------------------+
```

---

## Environment Configurations

| Configuration | Purpose | Typical topology |
|---|---|---|
| Local development | Single-node cluster for testing | 1 data node, 1-2 ingest, 1 query; optional components off |
| Production | Multi-node cluster | 3 data nodes, an ingest tier sized to its backlog (never below one pod), vector ingest and a graph builder, object store on its own node |

Values files set replica and worker counts, resources, node placement, storage classes and sizes, external access, object-storage configuration and which optional components run. Secrets are supplied at deploy time or by existing Kubernetes secrets, never committed.

---

## Operational Lifecycle

### Bootstrap

1. Data nodes start and form the cluster; buckets are created.
2. The supervisor waits for the data service, verifies buckets and bootstraps the admin account.
3. The supervisor signals readiness.
4. Ingest, vector ingest, the vector index writer and graph builders start, take their partitions and begin streaming.
5. The query service starts and connects to the data service.

### Upgrade

1. Images are built for the cluster architecture and published.
2. `helm upgrade` applies the change; Kubernetes rolls each service.
3. Ingest pods resume from checkpoints and from their durable queues; leases rebalance.
4. Data pods are replaced only deliberately, one at a time.

### Reset and reload

1. An admin flushes the data buckets through the flush task. The `session` bucket is normally kept, so head stores stay warm.
2. No restart is needed: change-stream consumers observe the flush and carry on.
3. A corpus is reloaded into `main` (for example with the `wiki_load` task), and every index rebuilds from it.

### Maintenance

Compaction, orphan-file collection, flush reclaim, ingest scaling, head-health reports, GPU reaping and node repair run as scheduled tasks. Destructive operations remain manual.

### Recovery

See [Data Persistence: Recovery Model](DATA_PERSISTENCE.md#recovery-model).

---

## Monitoring and Observability

* **Health and readiness endpoints** on every service, used by Kubernetes probes.
* **Pipeline metrics** per pipeline: enqueued, dequeued and error totals, stage queue depths, head latency.
* **Ontology quality**: the rate of records abstaining to `BFO:0000001`.
* **Head health**: a scheduled collapse report per encoder head (empty rate, dominance of a few values, overlap with the type segment).
* **Partition ownership** per ingest pod, and lease/stream alignment.
* **Query traces**: every `/v1/query` response carries a per-path `access_paths` trace and reports degraded sources.
* **Prometheus metrics** from the query service.
* **System status, storage statistics and pipeline errors** through the query service admin API and the dashboard.

---

## Security Boundaries

| Boundary | Protection |
|---|---|
| External to query service | TLS ingress; JWT on every non-health route |
| Admin operations | Admin role in the JWT; supervisor API admin-only |
| External direct data access | KV proxy: JWT handshake, data operations only |
| Services to data service | Bucket-scoped authentication; optional network policies |
| Object storage | Credentials from Kubernetes secrets |
| Data volumes | Optional encryption at rest |

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Summary

NAM's deployment is:

* **One chart**: Helm is the only deployment path, validated at render time
* **Multi-node storage**: replicated data tier backed by object storage
* **Independent pipelines**: address, vector and graph consumers scale and fail separately
* **Elastic ingest**: the ingest tier is sized from its backlog
* **Durable ingest**: on-disk queues and acknowledge-after-write
* **Self-maintaining**: scheduled maintenance tasks; destructive ones manual only
* **Authenticated at the edge**: TLS and JWT on every public boundary

> See also: [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Data Persistence](DATA_PERSISTENCE.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [API Reference](API_REFERENCE.md)
