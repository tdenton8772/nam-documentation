# Frequently Asked Questions (FAQ)

This FAQ answers the questions we hear most often from engineers, architects and operators encountering NAM for the first time.

If you are looking for marketing claims, this is not the right place.
If you are looking to understand what NAM *actually does*, this is.

---

## What problem is NAM solving?

> How do you retrieve meaningful, explainable knowledge from large corpora **without hiding how each result was found**?

NAM is built for environments where:
- Retrieval must be inspectable and attributable
- Structure (what a record is about, which entities it involves) matters as much as similarity
- Behaviour must be stable across runs and environments
- The index must be rebuildable from source data

It complements LLMs, search engines and databases rather than replacing them.

---

## What is NAM, in one paragraph?

NAM is a **converged semantic index**. One query endpoint runs up to three access paths over the same source records — a deterministic **address** path (BFO/CCO type, entity, and LCA-coded attribute, affordance and context axes, reached by prefix scans), an approximate **vector** path (top-K above a cosine floor), and a **graph** path (records and the entities they mention, walked from the query's entities) — and fuses their results. Every result row lists its `sources`, and every response carries an `access_paths` trace.

---

## Is NAM a vector database?

No — but it contains a vector index, and vector search is one of its three access paths.

The difference is in what the vector path is *for* and how it is reported:
- It is explicitly approximate: top-K records above a cosine floor, not a guarantee of the nearest K
- It runs alongside the structural address path, and graph when the query names an entity, not instead of them
- Its results are labelled (`sources` contains `vector`, with a `cosine` score), so they are never confused with structural matches

Inside the vector path, each record's embedding is filed into a region of a partition trained on the data; a query reads only the regions near it and ranks the candidates by cosine similarity.

→ See: [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md)

---

## Is NAM a knowledge graph?

Not in the traditional sense. NAM does not store triples or require predefined edges, and it has no graph query language.

It does maintain a **graph index**: records linked to the entities they mention, and entities to the ontology classes they are stored under, built automatically from the address store. The graph path walks it from the entities a query resolves.

→ See: [Graph Index](../ARCHITECTURE/GRAPH_INDEX.md)

---

## Is NAM a RAG system?

No — but it supports RAG systems. NAM retrieves records with payloads, scores where applicable, and a trace of how they were found. Feed those into an LLM and you get grounded, attributable context. NAM itself does not generate text.

---

## Is NAM deterministic?

The address and graph paths are deterministic: given the same inputs and artifacts, they produce the same addresses, plans, probes and results.

The vector path is approximate by design, and its results are always labelled as vector results, with their cosine. The fused response tells you exactly which path contributed what.

---

## What is the address made of?

```
tenant | partition | type | entity | attribute | affordance | context
```

- **type** — a BFO/CCO class such as a CCO person or organization class, or `BFO:0000001` when the ontology head could not classify
- **entity** — a hash of the entity's normalized name, independent of type
- **attribute, affordance, context** — LCA codes of the form `lca:coarse:fine`; `lca:0:0` marks an empty axis
- **storage** — each address is stored in a covering layout so that any set of specified axes is a direct match on something stored; the rotations layout does this with three cyclic rotations of the axes

→ See: [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md)

---

## What does `BFO:0000001` mean in a key?

It is BFO's root class, *entity*. The ontology head assigns it when the evidence supports nothing more specific. NAM treats it as a classification miss, not a category: the abstention rate is tracked as a quality metric, and the query planner always probes this partition as a safety net so such records stay reachable.

---

## How does NAM decide which access paths to run?

A routing policy uses cheap signals from the query:
- No resolved entity: address and vector
- At least one resolved entity: address, vector and graph (graph is seeded from up to three resolved entities)

Callers can pin paths explicitly with the `paths` field. If a routing policy leaves the vector path out and the address path returns too few results relative to the requested limit, the vector path is consulted as a fallback. Every decision and its reason appears in `access_paths`.

→ See: [Query Model](../ARCHITECTURE/QUERY_MODEL.md)

---

## How are results ordered?

Results are de-duplicated by record pointer. Address results come first, then vector-only and graph-only results, and any record whose key names a resolved entity is moved to the front. Vector rows carry a `cosine` score; address rows carry their match metadata.

---

## Does NAM "understand" language?

NAM does not claim human-like understanding. It performs deterministic linguistic parsing (rule-based NLP), standards-based classification (BFO/CCO), entity resolution and structured address assignment, and on the vector path it uses a text-embedding model. Understanding, in NAM, means consistent semantic placement, not generative reasoning.

---

## Does NAM learn in real time?

It accumulates knowledge, not rules. During ingest it:
- Adds new addresses, vectors and graph records
- Registers new entities and surface forms in the entity store

It does **not** change its encoder heads, address codec, ontology codebook or embedding model at runtime. An admin can select a different head implementation for a slot; NAM never changes one by itself.

---

## What happens when NAM cannot answer a question?

It returns fewer results and says why in the trace: which paths ran, what each found, and whether a fallback fired. If storage reads failed, the response is flagged as partial and names the failing sources — "could not look" is never reported as "found nothing".

---

## Can NAM be trained on my domain?

Yes. The ontology codebook, the LCA codec, head vocabularies and capability maps are offline, versioned artifacts that can be trained on domain corpora. The embedding model is swappable by reindexing from source records.

---

## Does NAM require an LLM?

No. LLMs can be used for labeling during training, or as downstream consumers, but NAM does not depend on them at runtime.

---

## Does NAM need a GPU?

No. Every stage runs on CPU. The vector pipeline can optionally burst embedding work to a GPU claimed from an external provider when a backlog builds, and falls back to CPU otherwise.

---

## How does data get into NAM?

Source documents are written to NAM's source bucket (directly, through the authenticated KV proxy, or by a loader task). A change stream (DCP) carries every mutation to the ingest pipeline, which holds records in a durable per-pod write-ahead log and acknowledges them only after their address writes land. Separate change-stream consumers build the vector index and the graph index.

→ See: [Pipeline Architecture](../ARCHITECTURE/PIPELINE_ARCHITECTURE.md)

---

## Can NAM scale?

Yes, horizontally:
- **Ingest** — pods divide the source partitions with time-bounded leases and fail over automatically, and the tier sizes itself to its backlog
- **Indexing** — vector ingest scales on its own; graph builder shards follow the ingest tier when autoscaling is on
- **Query** — query replicas are stateless
- **Storage** — the data service runs as a multi-node cluster with replicated partitions

Ingest throughput is bound by the CPU available to pipeline workers, so it scales with compute capacity; on the reference deployment compute nodes are added as the ingest pods need them.

---

## How does NAM persist data?

In NAM's own distributed key-value store. Source, session and coordination buckets use an object-storage-backed RocksDB engine; the address, graph and vector buckets use a raw RocksDB engine with prefix bloom filters. Data files are mirrored to S3-compatible object storage, with on-demand caching on open. Every index is derived and can be rebuilt from source records.

→ See: [Data Persistence](../ARCHITECTURE/DATA_PERSISTENCE.md)

---

## How does NAM handle failures?

- **Pod failure** — leases expire, surviving pods take over partitions, change streams resume from checkpoints, and records still in the write-ahead log are processed after restart
- **Write failure** — batches are redelivered; a batch that exhausts its retries is counted as an error, never silently dropped
- **Index builder interruption** — the graph builder resumes from its checkpoints and retries transient errors
- **Storage trouble at query time** — reported as partial or unavailable, never as an empty answer

---

## How does NAM handle security?

- JWT authentication with roles, rate limits and audit logging on the query API
- Admin JWT on every supervisor administrative route, failing closed without a signing secret
- A KV proxy that validates a JWT and then allows only data operations; change streams are admin-only
- TLS for the query service
- Secrets in Kubernetes secrets, redacted in task state and history
- Destructive maintenance only by hand
- Opt-in network policies and encryption at rest

→ See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Is NAM production-ready today?

NAM is in an active production-pilot phase. It is suitable for controlled pilots, internal deployments, domain-specific evaluations and partner integrations with hands-on support. It is not yet a fully managed platform.

→ See: [Current State](../ROADMAP/CURRENT_STATE.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md)

---

## What kinds of problems is NAM *not* good at?

- Open-ended text generation and creative writing
- Exhaustive nearest-neighbour search with recall guarantees
- Joins, aggregations and analytics
- Serving as a system of record

→ See: [What NAM Cannot Do](../CAPABILITIES/WHAT_NAM_CANNOT_DO.md)

---

## Where should I start next?

- Conceptual clarity → [High-Level Systems](../ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md)
- Practical constraints → [What NAM Can Do](../CAPABILITIES/WHAT_NAM_CAN_DO.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md)
- Current status → [Current State](../ROADMAP/CURRENT_STATE.md)
- Future direction → [Near-Term](../ROADMAP/NEAR_TERM.md) | [Long-Term](../ROADMAP/LONG_TERM.md)
- Definitions → [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [Glossary](GLOSSARY.md)

→ See also: [What Is NAM](../OVERVIEW/WHAT_IS_NAM.md)
