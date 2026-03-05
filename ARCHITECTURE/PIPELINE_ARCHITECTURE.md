# Pipeline Architecture

*How data flows through NAM from ingestion to storage*

This document describes the **runtime pipeline architecture** — how records move from their source through semantic encoding to addressable storage.

The pipeline is designed around three principles:

* **Isolation** — each processing stage runs independently
* **Parallelism** — stages that can run concurrently do so
* **Locality** — coordination happens within a pod, not across the cluster

---

## Data Entry: Change Data Capture

Data enters NAM through **change data capture (CDC) streaming**, not batch import or direct API submission.

This means:

* NAM watches an upstream data store for mutations
* New or changed records are streamed to the pipeline automatically
* Deletions and expirations propagate through the same channel
* The pipeline processes records in arrival order, not priority order

CDC ensures that:

* NAM does not own or lock source data
* Ingestion is continuous, not scheduled
* The system reacts to change, not to polling

---

## Per-Pod Architecture

Each ingest replica is a **self-contained processing unit** with a pod-local message bus.

A single ingest pod contains:

| Stage | Purpose | Parallelism |
|-------|---------|-------------|
| CDC adapter | Streams mutations from the data layer | 1 per pod |
| NLP | Rule-based linguistic analysis | 1 per pod |
| Ontology classifier | Semantic type classification | 1 per pod |
| Entity head | Named entity extraction and resolution | Multiple per pod |
| Attribute head | Property and descriptor extraction | 1 per pod |
| Affordance head | Action and capability extraction | 1 per pod |
| Context head | Situational and temporal context | 1 per pod |
| Addressing | Entity-anchored bundling, address construction, storage writes | 4 per pod |

### Why per-pod isolation?

* **No cross-pod coordination for pipeline processing** — each pod handles its own partition of the data independently
* **Message bus is local** — no network hops between stages within a pod
* **Failure is contained** — if one pod's NLP stage crashes, other pods continue unaffected
* **Scaling is horizontal** — add pods to increase throughput, not threads

---

## Stage Ordering

Records flow through stages in a strict sequence:

```
CDC Stream
   |
   v
NLP (tokenization, POS tagging, lemmatization, NER, dependency parsing)
   |
   v
Ontology Classification (semantic type: person, object, location, concept, ...)
   |
   v
Encoder Head Fan-Out (parallel)
   |--- Entity Head (x N workers)
   |--- Attribute Head
   |--- Affordance Head
   |--- Context Head
   |
   v
Addressing (streaming join of all head outputs per record)
   |--- Entity-anchored bundling
   |--- Address construction
   |--- Background storage writes
```

### Key properties of this ordering:

* **NLP runs first** — all encoder heads receive the same parsed representation. NLP is never skipped.
* **Ontology runs before heads** — type classification informs entity resolution and address partitioning
* **Heads fan out in parallel** — entity, attribute, affordance, and context extraction happen concurrently
* **Addressing performs a streaming join** — it accumulates all head outputs for a given record before constructing addresses
* **Storage writes are pipelined with feedback** — the addressing stage submits batches to a pool of writer threads and collects results (success, failure, timeout) via future callbacks

---

## NLP Pipeline

The NLP stage is a **fully rule-based linguistic analysis pipeline**. It contains no neural network models and no learned weights.

It performs:

* Tokenization (whitespace and punctuation-aware)
* Part-of-speech tagging (dictionary + suffix rules)
* Lemmatization (exception lists + suffix stripping)
* Named entity recognition (gazetteers + POS heuristics)
* Dependency parsing (deterministic head-finding rules)

### Why rule-based?

* **Determinism** — identical input always produces identical parse output
* **Speed** — throughput exceeds 10,000 parses per second on a single core
* **No model drift** — behavior does not change between deployments
* **No GPU required** — runs on commodity CPU

The NLP pipeline is a critical foundation. Every downstream encoder head depends on the quality and consistency of its output. Rule-based parsing makes this dependency predictable and auditable.

---

## Encoder Head Fan-Out

After NLP and ontology classification, the parsed document is published to all encoder heads simultaneously.

Each head:

* Receives the same parsed document
* Extracts its specific semantic signal independently
* Publishes its output back to the message bus

The **entity head** is the most I/O-intensive stage (due to entity resolution against the shared entity store) and is therefore parallelized with multiple workers per pod. Other heads are single-instance per pod.

### Why fan-out?

* **Independence** — heads do not depend on each other's output
* **Isolation** — a slow or failing head does not block others
* **Extensibility** — new heads can be added without changing existing ones

---

## Addressing: The Streaming Join

The addressing stage is where all head outputs converge.

For each record, the addressing stage:

1. **Accumulates** outputs from all encoder heads (entity, attribute, affordance, context)
2. **Bundles** semantic components using dependency-parse-scoped entity anchoring
3. **Constructs** deterministic addresses from each bundle
4. **Writes** addresses to the storage layer via the async WritePipeline

### Async WritePipeline

Storage writes are submitted to a **WritePipeline** — a pool of 4 concurrent writer threads (configurable via `NAM_ADDR_WRITE_WORKERS`). Each submission returns a **WriteFuture** that tracks the batch through completion, failure, or timeout.

At the top of each processing loop, the addressing stage calls `collect()` to retrieve settled futures. This provides:

* **Pipelined throughput** — multiple batches in-flight simultaneously, overlapping TCP round-trips across writer threads
* **Failure tracking** — write errors and timeouts are reported back via future callbacks, not silently dropped
* **Timeout detection** — batches exceeding the configured write timeout are flagged without blocking the pipeline
* **Backpressure** — queue capacity prevents unbounded memory growth
* **Adaptive throttling** — configurable inter-batch delays smooth write bursts to protect the data service under high concurrency

Write failures are logged and tracked via metrics, but do not halt the pipeline.

At 500K records, write throttling was validated as the mechanism that allows sustained addressing at scale without overwhelming the data service.

---

## Lease-Based Partition Coordination

Ingest replicas divide work using **CAS-based lease claims** against the data service.

Each pod:

* Claims ownership of a set of data partitions (vbuckets) via compare-and-swap operations
* Renews its leases periodically
* Sheds excess partitions when the cluster grows
* Claims expired leases from crashed peers

### Properties:

* **No external coordinator** — lease state lives in the same data service that stores everything else
* **Crash recovery is automatic** — when a pod dies, its leases expire and are claimed by surviving pods
* **Rebalancing is proportional** — each pod targets an equal share of the total partition space
* **No consensus protocol** — CAS operations provide sufficient coordination without Raft, Paxos, or similar

---

## Health Monitoring and Recovery

Each ingest pod exposes a health endpoint. The system tracks:

* **CDC adapter health** — consecutive streaming failures trigger an exhaustion signal
* **Pipeline throughput** — records processed per second
* **Partition ownership** — how many partitions each pod owns

When the CDC adapter exceeds a failure threshold:

1. The pod reports itself as unhealthy
2. A cluster-level supervisor detects the unhealthy state
3. The pod is terminated and restarted
4. On restart, the pod re-establishes leases and resumes streaming

Container-level liveness probes serve as a safety net for cases where the application-level health check itself becomes unresponsive.

---

## Scaling Model

Throughput scales linearly with pod count:

* Each pod processes its own partition slice independently
* Adding a pod triggers automatic rebalancing (existing pods shed partitions)
* Removing a pod triggers automatic failover (leases expire and are reclaimed)

The pipeline has no global bottleneck:

* NLP is CPU-bound but fast
* Entity resolution is I/O-bound but cached
* Addressing is compute-bound but batched
* Storage writes are asynchronous and throttle-aware

At 500K records, the system produced 13.8M semantic addresses with the data service remaining stable throughout — validated on a 2-pod ingest cluster with adaptive write throttling.

---

## Summary

The NAM pipeline is:

* **Stage-ordered** — NLP before ontology before heads before addressing
* **Pod-local** — no cross-pod coordination for processing
* **Parallel where it matters** — entity extraction fans out, other stages are single-instance
* **Asynchronous at the boundary** — storage writes never block the pipeline
* **Self-healing** — lease expiry, health monitoring, and liveness probes handle failures automatically

This architecture enables deterministic, horizontally scalable semantic encoding without external coordination dependencies.

> See also: [Ingestion Model](INGESTION_MODEL.md) | [Addressing Model](ADDRESSING_MODEL.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Data Persistence](DATA_PERSISTENCE.md)
