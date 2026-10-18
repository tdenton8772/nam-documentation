# What NAM Can Do

NAM is a **converged semantic index**: one query endpoint over three access paths built from the same source records.

* The **address path** — a deterministic semantic address index. Each record is placed at addresses made of a BFO/CCO type, an entity, and attribute, affordance and context axes, and queries reach them by prefix scans.
* The **vector path** — an embedding index that returns the top-K records above a cosine floor. It is explicitly approximate.
* The **graph path** — a graph of records and the entities they mention, walked from the entities a query names.

NAM fuses the paths into one result list. Every result row says which paths found it (`sources`), and every response carries an `access_paths` trace of what each path did. NAM is derived from source data and is never the system of record.

This document describes the **classes of problems NAM is built to solve**.

---

## 1. Explainable Semantic Retrieval

When the address path returns a record, NAM can show:

* which type partition, entity and axis values were probed
* at which geometry the match happened (point, line, plane or cube)
* which address of the record matched

When the vector path returns a record, it carries its cosine score. When the graph path returns a record, the trace names the seed entities and the walk statistics.

This enables:

* auditability of results, path by path
* reproducible behaviour on the address and graph paths
* a clear distinction between "found by structure" and "found by similarity"

---

## 2. Entity-Anchored Questions

NAM resolves the entities a question names against the same entity store ingest built, and uses them three ways:

* the address path probes the type partitions each entity is actually stored under first
* the graph path walks from each resolved entity (several entities are walked in parallel, and records reached from more of them rank first)
* records whose key *is* a resolved entity move to the front of the fused list

This makes NAM effective for questions such as:

* "Who was X?"
* "What is known about X and Y?"
* "Which records involve X in this context?"

---

## 3. Free-Text and Exploratory Questions

When a question names no known entity, NAM still answers:

* the address path explores type tiers in a fixed order, widening from specific to general geometry
* the vector path finds records by embedding similarity above the cosine floor, alongside the address path

Exploration remains traceable: the trace says which paths ran, why, and what each contributed.

---

## 4. Typed, Standards-Based Classification

Every address carries a class from **BFO 2020 and CCO**, the formal upper- and mid-level ontologies. This gives:

* a shared, standard vocabulary for what a record is about (person, organization, geospatial region, act, information content entity, and so on)
* subsumption-aware querying: a query can reach classes related to a hint by ancestry
* honest abstention: when evidence is thin a record is classified at a broader class, or at `BFO:0000001`, rather than forced into a wrong one

---

## 5. Structured Retrieval Over Unstructured Text

NAM can ingest natural-language text, summaries, records, logs, documentation and other knowledge artifacts, and answer questions over them without:

* schema-first modeling
* predefined joins
* hand-crafted ontologies per dataset

Structure comes from **addressing at ingest**, not preprocessing.

---

## 6. Relationship Discovery

The graph index connects records to the entities they mention, and entities to the ontology classes they are stored under. With it, NAM can:

* list the records that mention an entity
* find entities that co-occur with an entity across records
* walk outward a configurable number of hops

All of this is key-value lookups and prefix scans. No similarity is computed on the graph path.

---

## 7. Continuous Ingestion

Data enters NAM through a change stream from its source bucket. NAM:

* picks up new and changed records automatically
* holds every handed-off record in a durable per-pod write-ahead log until its writes have landed
* resumes from checkpoints after restarts
* rebuilds every derived index from the source records if needed

---

## 8. Incremental Knowledge Accumulation

NAM answers more questions as more data is ingested:

* more addresses fill more of the semantic space
* the entity store learns new entities and surface forms
* the graph gains records and edges

This is **not** runtime learning of new rules. The *space* becomes richer while the *navigation rules* stay fixed.

---

## 9. Domain-Specific Deployment

NAM is designed to be:

* trained on customer-specific domains through offline artifacts (ontology codebook, LCA codec, head vocabularies)
* extended with pluggable encoder heads, selected per slot and swappable on a running cluster for heads that keep no state (implementations developed outside NAM's tree are published in the open-source `nam-heads` package)
* deployed as separate instances with different semantic profiles

All while preserving determinism on the address path, auditability, and isolation between deployments.

---

## 10. Pairing With Downstream Systems

NAM returns records with their payloads, scores where applicable, and a full access trace. It is designed to feed:

* LLMs and RAG pipelines that need grounded, attributable context
* applications that must justify what they retrieved
* analysts who want to explore before they ask a precise question

---

## Summary

NAM is well suited to problems that need:

* **explainable retrieval**, with every result attributed to the path that found it
* **entity-anchored and typed** access to knowledge
* **similarity search where it helps**, clearly labelled as approximate
* **relationship discovery** without a graph database
* **continuous, rebuildable** indexing of an evolving corpus

It is not a general-purpose search engine. It is a **semantic memory layer** designed to support sense-making at scale.

→ See also: [What NAM Cannot Do](WHAT_NAM_CANNOT_DO.md) | [Current Limitations](CURRENT_LIMITATIONS.md) | [Query Model](../ARCHITECTURE/QUERY_MODEL.md) | [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md) | [Graph Index](../ARCHITECTURE/GRAPH_INDEX.md)
