# High-Level System Architecture

*A conceptual view of NAM*

This document describes NAM at the **system level**: how information flows through it, how responsibilities are separated, and how the major components interact.

It is not an implementation guide. Where it names a storage bucket or a protocol, it does so to make the shape of the system concrete, not to document an interface.

---

## System Overview

NAM is a **converged semantic index**. It reads records from a source store and builds three derived indexes over them, all served through one query endpoint:

1. **The semantic address index** (the core). Every record is projected into deterministic addresses: an ontology type, an entity anchor, and three coded semantic axes (attribute, affordance, context). Queries are answered by key lookups and prefix matches over those addresses.
2. **The vector index.** Every record is also embedded and filed in a vector index. Queries are answered by reading the part of the index nearest the question and ranking the candidates by cosine similarity. This path is explicitly approximate: it returns the top results above a similarity floor.
3. **The graph index.** Records, the entities they mention, and the ontology classes each entity is stored under are linked by small graph records derived from the addresses. Queries are answered by walking outward from the entities a question names.

A query runs on one or more of these paths. The results are fused into one list; every row lists the `sources` (paths) that found it, and every response carries an `access_paths` trace of what each path did and why.

NAM is **derived state**. The source store is the system of record; every index can be rebuilt from it.

---

## The Two Directions

| | Ingest | Query |
|---|---|---|
| Entry point | Change stream from the source store | One HTTP endpoint |
| Address index | NLP, ontology, encoder heads, bundling, address construction, write | Same NLP and heads, same bundling, then planning and probes |
| Vector index | Embed the record and file it in the index | Embed the question, read the part of the index near it, rank by cosine |
| Graph index | A separate builder follows address writes and writes graph records | Walk from the entities resolved in the question |

The address path is **symmetric by construction**: ingest and query use the same parser, the same encoder heads, the same entity resolver and the same coordinate codec. That symmetry is what makes deterministic retrieval possible.

---

## Core Components

1. **Source store and change stream**
2. **Ingest pipeline** (per ingest replica)
3. **NLP stage**
4. **Ontology head**
5. **Encoder heads** (entity, attribute, affordance, context)
6. **Address builder**
7. **Graph index builder**
8. **Vector ingest** (and its index writer)
9. **Data service** (storage)
10. **Query service** (orchestrator, address planner and runner, vector search, graph walker)
11. **Head-store cache** (per node)
12. **Supervisor** (lifecycle and task framework)
13. **Access control**

Each has a single responsibility and evolves independently.

---

## 1. Source Store and Change Stream

Records enter NAM by being written to the **source store** (the `main` bucket of the data service). NAM does not take records through an import API.

Each consumer of the source store follows its **change stream**: a per-partition, ordered stream of mutations with resumable checkpoints. The address pipeline, the vector pipeline and the graph builder are independent consumers, each with its own checkpoints, so one can lag or restart without affecting the others.

---

## 2. Ingest Pipeline

Each ingest replica is a self-contained processing unit with a pod-local message bus. It claims a share of the source partitions, streams their changes, and runs every record through the staged pipeline: NLP, ontology, encoder heads, a join of all head results per record, and addressing.

Two properties matter at system level:

* **Durable hand-off.** A record whose head results are complete is written to a durable on-disk queue before it is addressed. The queue entry is acknowledged only after the record's address writes have landed. A failed write puts the entry back for redelivery. On the reference deployment every earlier hop between stages is a durable on-disk stream too, and each stage acknowledges its input only after its output has been handed on.
* **Backpressure.** Every stage publishes its queue depth. When a stage fills, the change-stream reader slows down, and flow control propagates back to the data service.

→ See: [Pipeline Architecture](PIPELINE_ARCHITECTURE.md)

---

## 3. NLP Stage

A **rule-based** linguistic analysis stage: tokenization, part-of-speech tagging, lemmatization, named entity recognition and dependency parsing.

* **Deterministic.** Identical input always produces identical output.
* **No learned weights and no GPU.**
* **Mandatory.** Every encoder head reads the same shared parse. A record that reaches a head without one is parsed there with the same rule parser, so no head runs without a parse.

---

## 4. Ontology Head

The ontology head answers *"what kind of thing is this record about?"* with a class from **BFO 2020 and the Common Core Ontologies (CCO v2.2)**, a certified, standards-based class hierarchy.

* Each sentence is scored against a trained codebook of class centroids (the default implementation of the ontology slot).
* An uncertain sentence backs off to the deepest ancestor class the evidence supports.
* When nothing beyond existence can be asserted, the head abstains with the root class, `BFO:0000001` (entity). The abstain rate is tracked as a quality metric.
* The record's class is the vote of its sentences.

The record's class becomes the **type segment of every address the record emits**. All of a biography's addresses, including one anchored on an award the person won, carry the class of the person.

---

## 5. Encoder Heads

Encoder heads are pluggable, deterministic projectors from the shared parse to symbolic coordinates. Each head fills a **slot**; a slot manifest that ingest and query both read records which implementation fills each slot, and an admin can swap a stateless head's implementation on a running cluster. Head implementations developed outside NAM's own tree are published in the open-source `nam-heads` package.

| Head | Answers | Feeds |
|---|---|---|
| **Entity** | Which specific things are referred to? | The entity anchor (a hash of the canonical name) |
| **Attribute** | What qualities are asserted? | Axis x |
| **Affordance** | What action or interaction is described? (synonyms folded to canonical verbs) | Axis y |
| **Context** | Under what circumstances does it apply? (time periods, places, institutions, fields) | Axis z |

On the reference deployment the entity head runs first; a deterministic sentence resolver then ties each sentence's statements to the resolved entities, and the attribute, affordance and context heads run in parallel on the result.

Heads are trained or curated offline and fixed at runtime. Their vocabularies (entity surface forms, attribute and affordance categories, context values) live in **head stores** in the `session` bucket, which grow as ingest sees new terms and act as a warm cache across reloads.

The entity head also provides the **one entity resolver** the system uses: the query service resolves the entities in a question with the same code and the same store that ingest used to anchor addresses.

---

## 6. Address Builder

The address builder turns head output into keys.

* **Entity-anchored bundling.** The dependency parse decides which attributes and actions belong to which entity, so a record emits addresses for what its text actually says, not for every combination of its parts.
* **Coordinate coding (LCA).** Each axis value is encoded by its axis head's learned codec (a character CNN with two-stage residual quantization) into a code pair `lca:coarse:fine`. `lca:0:0` marks an empty axis. Similar strings get nearby codes, which the query planner exploits.
* **A covering index.** Addresses are stored so that any set of specified axes is a direct match on something stored, with no wildcard copies and nothing derived at read time. The reference deployment stores each partition's (tenant, partition, type, entity) addresses together and reads a planning tier in a fixed number of batched round trips. The alternative **rotations layout** stores each address under three cyclic rotations of its axes.

Address shape:

```
tenant | partition | TYPE | entity | attribute | affordance | context
```

Address construction is deterministic, stateless and shared by ingest and query.

→ See: [Addressing Model](ADDRESSING_MODEL.md) | [Ingestion Model](INGESTION_MODEL.md)

---

## 7. Graph Index Builder

A separate service that follows the change stream of the **address** store and writes the graph index: record-entity edges in both directions, and the ontology classes each entity is stored under. The builder shards by partition.

The graph builder is off the ingest acknowledgement path. The graph index therefore **lags** the address index slightly and never slows addressing down.

→ See: [Graph Index](GRAPH_INDEX.md)

---

## 8. Vector Ingest

A separate pipeline with its own change-stream consumers and its own durable queue. Its workers embed each record, and a single index writer files the embeddings into the vector index.

Embedding runs on CPU by default. When a backlog builds, a GPU burst tier is brought up through an external GPU registry and released after a period with no work.

→ See: [Vector Search](VECTOR_SEARCH.md)

---

## 9. Data Service

NAM's own distributed key-value store with an ordered change stream, prefix scans and append. It runs as a multi-node tier with partition (vbucket) replicas.

| Bucket | Holds | Engine |
|---|---|---|
| `main` | Source records | RocksDB-Cloud, one column family per partition |
| `session` | Head stores (entity, attribute, affordance, context vocabularies) | RocksDB-Cloud |
| `metrics` | Leases, checkpoints, pipeline counters, user accounts | RocksDB-Cloud |
| `nam` | Semantic addresses | Raw RocksDB, one shared column family, prefix bloom filters |
| `graph` | Graph index records | Raw RocksDB |
| `vector` | Vector index records | Raw RocksDB |

Storage is mirrored to S3-compatible object storage.

→ See: [Data Persistence](DATA_PERSISTENCE.md)

---

## 10. Query Service

The only external API for retrieval. Its **orchestrator** routes each query across the three paths and fuses the results.

* **Routing.** A caller may pin paths explicitly. Otherwise a routing policy decides from cheap signals: the address and vector paths always run; the graph path runs when the question names an entity the store knows.
* **Address path.** Canonicalize the question, resolve its entities, encode it with the same heads as ingest, bundle, then plan probes of address regions: first the classes each resolved entity is actually stored under, then ontology tiers by BFO branch, with a probe of the unclassified class as a safety net. Codebook neighbours widen each axis. Regions widen from points to lines, planes and cubes until enough distinct records are found.
* **Vector path.** Embed the question and return the top results above a cosine floor.
* **Graph path.** Seed a walk from each resolved entity (up to three) and walk them in parallel.
* **Fusion.** Rows are de-duplicated by record pointer, each row lists its `sources`, and a record that *is* one of the resolved entities moves to the front.

Partial failures are reported, never hidden: a response says which paths could not read storage.

→ See: [Query Model](QUERY_MODEL.md) | [API Reference](API_REFERENCE.md)

---

## 11. Head-Store Cache

Every compute node runs a cache process that follows the `session` bucket's change stream and keeps memory-mapped copies of the four head stores on node-local shared memory. Encoder head workers read them without a network round trip. Writes made by heads during ingest are relayed to the store through a single writer per head, so each cache has exactly one writer.

The cache is disposable: it rebuilds from the change stream.

---

## 12. Supervisor

A single-instance service that gates readiness during bootstrap, monitors health, and runs the **task framework**: loads, flushes, compactions, object-storage maintenance, ingest scaling, head-health reports, GPU reaping and node repair. Destructive tasks run only when triggered by hand.

→ See: [Deployment Architecture](DEPLOYMENT_ARCHITECTURE.md)

---

## 13. Access Control

* The query service requires a signed token (JWT) on every endpoint other than login, password reset by token, health, readiness and metrics; admin endpoints require the admin role.
* The supervisor's admin API requires an admin token and refuses to run without its signing secret.
* Direct key-value access goes through a JWT-authenticating proxy that admits only data operations.
* The query endpoint is served over TLS.

→ See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## End-to-End Flow (Ingest)

1. A record is written to the source store.
2. An ingest replica that owns its partition streams the change.
3. NLP parses it once; the ontology head types it; the entity head resolves its entities, and the attribute, affordance and context heads run in parallel.
4. Head results are joined per record and queued durably.
5. The addressing stage bundles, codes and writes the addresses, then acknowledges the queue entry.
6. Independently, the graph builder follows the address writes, and the vector pipeline embeds the record.

## End-to-End Flow (Query)

1. A question arrives at the query service.
2. The orchestrator gathers routing signals (resolved entities, intent) and starts the chosen paths in parallel.
3. Each path returns records; the orchestrator fuses them and records a trace per path.
4. Payloads are attached from the source store, and the response returns rows, sources and the trace.

Queries never mutate stored state.

---

## Architectural Guarantees

* The address path is deterministic: the same input and artifacts produce the same addresses and the same scans.
* The vector path is honest about being approximate: it returns what clears a similarity floor, with the work it did reported.
* Every result names the path that produced it.
* Every index is rebuildable from the source store.
* Records handed off to the address pipeline are not lost to a pod restart.
* All external access is authenticated.

---

## What NAM Is Not

* Not a system of record.
* Not a vector database: embeddings are one access path among three, not the model of meaning.
* Not a knowledge graph engine: the graph index is derived from addresses, not curated.
* Not a generator: NAM retrieves; it does not summarize or answer.

→ See also: [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [Ingestion Model](INGESTION_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Data Persistence](DATA_PERSISTENCE.md) | [Vector Search](VECTOR_SEARCH.md) | [Graph Index](GRAPH_INDEX.md) | [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md)
