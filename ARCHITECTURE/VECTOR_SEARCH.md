# Vector Search

*The embedding access path*

NAM's core is the deterministic semantic address index. Alongside it, NAM maintains a **vector index** built from the same source records. The vector path answers the questions the address path cannot reach — paraphrase, topical neighbourhood, records whose wording shares nothing structural with the query.

The vector path is **explicitly approximate**. It returns the top results above a similarity floor, not a complete or exact set. That is a deliberate contract, stated here so it is never confused with the address path's guarantees.

---

## Where It Sits

```
Source records (main bucket)
   |
   +--> addressing pipeline --> nam bucket   (address path)
   |                               |
   |                               +--> index builder --> graph bucket  (graph path)
   |
   +--> vector ingest ------> vector bucket  (vector path)
```

Vector ingest is its own change-data-capture consumer of the source bucket, running in its own StatefulSet with its own partition leases. It does not depend on the addressing pipeline and does not slow it down. Like every NAM ingest pipeline, it hands records to its workers through a durable, file-backed write-ahead log (WAL) on a per-pod volume; a record leaves the WAL only after it has been handed on durably. The workers embed records, and a single index writer files them into the vector index.

At query time the vector path is one of the three paths the query orchestrator can run. It is also available directly at `POST /v1/vector/search`.

→ See: [Query Model](QUERY_MODEL.md) | [API Reference](API_REFERENCE.md)

---

## The Embedding Model

* **NAM's own model.** Records and queries are embedded by NAM's own copy of a text-embedding model, on CPU by default; for ingest, a rented GPU can add capacity (see below). No hosted AI service is called.
* **Consistent.** Ingest and query embed text the same way, by construction: a record as a document, a question as a query.
* **Text embedded:** the record's title and text.

The model is a configuration choice behind a provider interface. The vector services refuse to start when the configured model is not the one the index was built for, rather than silently searching in a different vector space. Changing the model requires re-embedding the corpus; the similarity floor (below) is calibrated per model and does not transfer between models.

---

## How the Index Behaves

Each record's embedding is filed, as it is ingested, into a region of a partition trained on the data. A query reads only the regions nearest to it and ranks the records it finds there by cosine similarity.

What that gives a caller:

* **Bounded work per query.** A query reads a bounded part of the index, never the whole corpus.
* **No duplicates.** A record that is redelivered or replayed never takes two result slots: results are de-duplicated by record.
* **Rebuildable.** The index is derived from source records and can be rebuilt by re-embedding them.
* **Tenant-scoped.** Each tenant's records are indexed and searched separately.

---

## Query: Top-K Above a Floor

Every vector query carries two things: a **cosine target** (the similarity floor) and a **limit**.

1. Embed the query.
2. Read the part of the index nearest the query.
3. Score the candidates found there by cosine similarity.
4. Keep those at or above the floor.
5. Return the top `limit`, best first.

### The floor

The cosine floor is a **relevance requirement**, enforced as a clamp by the deployment: a request can ask for a higher floor, never a lower one. The number itself is a per-model calibration.

### What the caller sees

Each result carries its data pointer and its cosine. The response also reports how much of the index the search read. Within `/v1/query`, that report appears in the vector entry of the `access_paths` trace, and every fused row the vector path contributed to carries its `cosine`.

### Read failures are errors, never empty answers

If the vector path cannot read the index, it says so: `/v1/vector/search` answers `503`, and in `/v1/query` the vector path's trace entry is an `error` while the other paths still answer. A caller can always tell "nothing cleared the floor" from "could not look."

### Readiness

A query replica reports ready only once its view of the vector index is loaded (with a bounded wait), so a rollout keeps the old replica serving instead of sending the first queries to a cold one.

---

## Embedding Capacity: CPU and GPU Tiers

Embedding is the expensive step of vector ingest.

* The **CPU tier** runs the model locally in each vector worker and handles every batch by default.
* When a backlog builds, a **GPU burst tier** takes over: the workers claim a GPU through an external GPU registry (backed by a cloud GPU provider), stream batches to it, and release it after a sustained idle period (about half an hour with no embedding work on the reference deployment). One GPU serves the whole cluster. A supervisor task reaps any GPU whose owner died before releasing it.
* **Both tiers must produce the same vectors.** They run the same model at the same precision, and the burst wrapper refuses to start if the two tiers disagree on model or dimension. A record embedded on one tier must be findable by a query embedded on the other.

The GPU tier is optional capacity, not a dependency. Without it, the CPU tier still embeds everything — more slowly.

---

## Guarantees and Non-Guarantees

The vector path **does**:

* return records at or above the configured cosine floor, ranked by cosine
* read a bounded part of the index for every query
* stay rebuildable from source records
* report the work it did on every query
* report a read failure as an error, never as an empty result

The vector path **does not**:

* guarantee completeness — a relevant record may lie in a part of the index the query did not read
* guarantee the best K overall — the K returned qualify, but need not be the global best K
* explain *why* a record is relevant beyond its cosine
* remove a record's earlier entry when its source record is updated or deleted

Those are exactly the properties the address path provides and the vector path trades away. NAM keeps both, and every fused result says which path found it.

→ See also: [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Query Model](QUERY_MODEL.md) | [Graph Index](GRAPH_INDEX.md) | [Data Persistence](DATA_PERSISTENCE.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)
