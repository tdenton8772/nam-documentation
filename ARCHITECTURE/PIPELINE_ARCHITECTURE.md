# Pipeline Architecture

*How records flow from the source store into NAM's indexes*

This document describes the **runtime ingest architecture**: how records move from the source store through semantic encoding into the address index, and how the two companion pipelines (graph and vector) follow alongside.

The pipeline is designed around four principles:

* **Isolation**: each stage runs as its own process
* **Locality**: stages of one replica talk over a pod-local message bus
* **Durability at the hand-off**: a record is not acknowledged until its writes have landed
* **Backpressure, not buffering**: when a stage fills, intake slows

---

## Data Entry: The Change Stream

Records enter NAM by being written to the source store (the `main` bucket). NAM follows the source store's **change stream**: an ordered, per-partition stream of mutations with resumable checkpoints.

This means:

* NAM never owns or locks source data
* Ingestion is continuous, not scheduled
* A consumer that restarts resumes from its checkpoint

Three independent consumers follow change streams:

| Consumer | Follows | Writes |
|---|---|---|
| Address pipeline (ingest replicas) | `main` | `nam` |
| Vector pipeline (vector ingest replicas) | `main` | `vector` |
| Graph index builder | `nam` | `graph` |

Each has its own checkpoints and its own partition ownership, so one can lag, pause or restart without affecting the others.

---

## Per-Replica Architecture

Each ingest replica is a **self-contained processing unit**: a change-stream adapter, a pod-local message bus, and one process per stage.

| Stage | Purpose | Parallelism |
|---|---|---|
| Change-stream adapter | Streams mutations for the partitions this replica owns | 1 per replica |
| NLP | Rule-based parse, shared by every head | Configurable worker group |
| Ontology | BFO/CCO class per sentence and per record | Configurable |
| Entity head | Entity extraction and resolution | Configurable (usually several) |
| Attribute head | Qualities asserted in the text | Configurable |
| Affordance head | Actions and interactions | Configurable |
| Context head | Time, place, institution and field | Configurable |
| Accumulator | Joins all head results per record; writes complete records to the durable queue | Per addressing partition |
| Addressing | Bundling, coordinate coding, address construction, writes | Per addressing partition |
| Head-store sync (one per head store) | Single writer relaying new head-store entries to the `session` bucket | 1 each |

Head results are routed by record so that all results for one record reach the same accumulator.

### Why per-replica isolation?

* **No cross-replica coordination for processing**: each replica handles its own partition slice
* **Local message bus**: no network hops between stages
* **Contained failure**: one replica's stage failing does not stop another replica
* **Horizontal scaling**: add replicas to add throughput

---

## Stage Ordering

```
Change stream (main)
   |
   v
NLP (tokens, POS, lemmas, NER, dependency parse, sentence skeleton)
   |
   v
Ontology head (BFO/CCO class; abstains to BFO:0000001)
   |
   v
Entity head, then sentence resolution
   |
   v
Axis heads, in parallel
   |--- Attribute
   |--- Affordance
   |--- Context
   |
   v
Accumulator (join per record)
   |
   v
Durable queue (on-disk, per replica)
   |
   v
Addressing
   |--- Entity-anchored bundling
   |--- LCA coordinate coding
   |--- Address construction (covering-index entries)
   |--- Write to nam
   |--- Acknowledge the queue entry
```

This is the order the reference deployment runs. The other supported order runs all four encoder heads in parallel straight after the ontology head; ingest and query are always configured with the same order.

Key properties:

* **NLP runs first and runs once.** Every head reads the same parse. NLP is never skipped: a record that reaches a head without a parse is parsed there with the same rule parser, so no head runs without one.
* **Ontology runs before the heads.** Heads use the record's class to weight their own choices.
* **Entities before axes.** The entity head resolves the record's entities, and a deterministic sentence resolver ties each sentence's statements to them, before the axis heads read the sentences.
* **Axis heads fan out in parallel.** A slow head delays only the records waiting on it.
* **The accumulator is a streaming join.** A record waits until every head has reported. A record whose head result never arrives is evicted after a timeout and counted as an error, not held forever.

---

## The NLP Stage

A fully rule-based linguistic analysis stage with no learned weights:

* Tokenization
* Part-of-speech tagging (dictionary and suffix rules)
* Lemmatization (exception lists and suffix stripping)
* Named entity recognition (gazetteers and POS heuristics)
* Dependency parsing (deterministic head-finding rules)

On the reference deployment it also builds a sentence skeleton, which the sentence resolver completes once the entity head has resolved the record's entities.

Rule-based parsing gives determinism (identical input, identical parse), no model drift between deployments, and no GPU requirement. Because every head depends on it, its consistency is the foundation of address determinism.

---

## The Ontology Head

The ontology head types text with a class from BFO 2020 and CCO v2.2. Each sentence is turned into a cue vector and scored against a trained codebook of class centroids; an uncertain sentence backs off to the deepest ancestor class the probability supports, and then to the root class `BFO:0000001`. The record's class is the vote of its sentences. The head refuses to start without a valid codebook that matches the parser.

The record's class becomes the type segment of every address the record emits. Records that abstain are still addressed under `BFO:0000001`, and the abstain rate is published as a quality metric.

---

## Encoder Heads and Head Stores

Each head reads the shared parse and the record's class, and emits values for its axis. Heads that learn vocabulary (new entities, new attribute or affordance categories, new context values) write to **head stores** in the `session` bucket.

Head-store reads go through three levels:

1. An in-process cache
2. A memory-mapped node-local copy maintained by the per-node cache process, which follows the `session` change stream
3. The `session` bucket itself

New entries a head creates are published to its sync stage, the single writer for that store, which writes them to `session`; the change stream then carries them to every node's cache.

The entity head is the most expensive stage and usually runs several workers per replica.

---

## Addressing and the Durable Queue

### Durable hand-off

Once the accumulator has every head result for a record, it publishes the complete record to a **durable, on-disk work queue** on the replica's persistent volume. The queue:

* Survives a pod restart (file storage, never memory)
* Never evicts: when full it rejects new entries, and the publisher retries while intake throttles

On the reference deployment every earlier hop is durable in the same way. The change-stream adapter advances its checkpoint only once a batch is durably on the first stream, and each stage acknowledges its input only after its output is durably on the next, so a record is never only in memory once the change stream has moved past it. A deployment that runs the head stages on the lightweight bus keeps this guarantee from the accumulator's queue onward, and counts a record lost before it as an error.

### Acknowledge after write

The addressing stage pulls batches from the queue and, for each batch:

1. Bundles, codes and builds the addresses
2. Writes them to the `nam` bucket through a pool of concurrent writers
3. Waits for every write in the batch to report success
4. Acknowledges the batch

If any write fails, the batch is returned to the queue for redelivery after a growing delay. By default a batch is retried until it lands; when a redelivery limit is configured, the final failed delivery is removed and its records are counted in the pipeline's error total, so loss is never silent.

Address values are appended lines. A redelivered batch can append a line a second time; readers de-duplicate lines by record pointer, so redelivery costs space, not correctness.

Writes are protected by per-batch timeouts and a circuit breaker that pauses writing after repeated failures.

---

## Backpressure

Every stage publishes its queue depth (the backlog waiting for it). The change-stream adapter watches the stages' depths and how full each durable stream on its pod is:

* When any watched stage passes a high watermark, the adapter stops reading from the change stream.
* It resumes only when all watched stages drop below a low watermark.
* While it is paused, the stream's own flow control pushes back to the data service.

A stage that stops publishing is ignored after a short staleness window, so a crashed stage can neither hold the throttle shut nor block recovery. All internal queues are bounded.

---

## Partition Ownership: Leases

Replicas divide the source partitions among themselves with **TTL-based lease documents** in the `metrics` bucket.

* A replica claims a partition by writing its lease document; the document carries a server-side expiry.
* It renews by refreshing the expiry, one lightweight operation per partition.
* A partition whose lease has expired is claimable by any replica.
* Each replica targets an equal share; an over-full replica sheds partitions to its peers.

Properties:

* **No external coordinator**: lease state lives in the data service
* **Automatic failover**: a dead replica's leases expire and are claimed by survivors
* **Proportional rebalancing**: adding a replica causes existing ones to shed
* **Clean scale-down**: ownership follows the ready replicas, and a replica that is shutting down releases its leases and drains its queues first

The vector pipeline and the graph builder use their own lease or shard namespaces, so their ownership is independent of the address pipeline's.

---

## Companion Pipelines

### Graph index builder

A sharded service that follows the `nam` change stream. Each shard owns the partitions whose number modulo the shard count equals its ordinal. For every address it writes the record-entity edges and the entity's ontology class, as blind appends that readers de-duplicate. It retries transient errors rather than dropping items and reports itself unhealthy when stalled.

Because it is off the addressing acknowledgement path, graph lags the address index and never slows it.

→ See: [Graph Index](GRAPH_INDEX.md)

### Vector pipeline

A separate set of replicas with their own change-stream consumers on `main`, their own durable queue, and their own workers. The workers embed records and hand them, over a durable stream, to a single index writer that files them into the vector index.

Embedding normally runs on CPU. When the backlog crosses a watermark, the worker acquires a GPU through an external GPU registry and streams batches to it; once the backlog has stayed low and no embeddings have been sent for about half an hour, the claim is released. A supervisor task reaps any GPU whose owner died before reporting it.

→ See: [Vector Search](VECTOR_SEARCH.md)

---

## Health Monitoring and Recovery

Each replica exposes a health endpoint that reports change-stream adapter health, partition ownership and throughput. The query service aggregates enqueued, dequeued and error totals per pipeline.

* An adapter that exhausts its failure budget reports unhealthy and is restarted; on restart it reclaims leases and resumes from its checkpoints.
* Stage processes that die are detected and restarted by their supervisor loop.
* Container liveness probes are a backstop.

---

## Scaling Model

* Each replica processes its own partition slice, so adding replicas adds throughput.
* **Elastic ingest.** The supervisor sizes the ingest tier, and the graph builder with it, from the backlog: records not yet streamed plus records still in flight. An autoscaler applies the count, up at once and down only after a quiet period. On the reference deployment the tier never goes below one ingest pod, and compute nodes are added and removed as the pods need them.
* Stage worker counts are tuned per deployment; the expensive stages (NLP, entity, addressing) can run several workers per replica.
* The vector and graph pipelines scale independently of the address pipeline.

---

## Summary

The NAM ingest pipeline is:

* **Stage-ordered**: NLP, ontology, heads, join, durable queue, addressing
* **Pod-local**: no cross-replica coordination for processing
* **Durable at the hand-off**: acknowledged only after the address writes land
* **Backpressured**: intake slows when the write side fills
* **Self-healing**: lease expiry, redelivery, health checks and probes handle failures
* **Accompanied**: graph and vector indexes are built by independent consumers

> See also: [Ingestion Model](INGESTION_MODEL.md) | [Addressing Model](ADDRESSING_MODEL.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Data Persistence](DATA_PERSISTENCE.md)
