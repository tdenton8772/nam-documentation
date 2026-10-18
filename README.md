# Neural Addressable Memory (NAM)

**Neural Addressable Memory (NAM)** is a converged semantic index designed to support **sense-making before certainty**.

NAM gives one query endpoint over three access paths, all built from the same source records:

1. **The semantic address index** — the core. Records are projected into a deterministic, multi-dimensional address space and retrieved by geometric prefix scans. No similarity scoring, no ranking model.
2. **The vector index** — an embedding index for paraphrase and topical neighbourhood. Explicitly approximate: top results above a similarity floor.
3. **The graph index** — relationships between records and the entities they mention, for walking outward from what a query names.

Results from the paths are fused. Every result row lists the `sources` (paths) that found it, and every response carries an `access_paths` trace of what each path did, why, and what it cost.

NAM is not a vector database, not a knowledge graph, and not a traditional RAG system — though it contains a vector path and a graph path. Its defining idea is that meaning can be treated as **addressable structure**, with similarity search kept as a clearly labelled complement rather than the foundation.

This repository contains the **public documentation** for NAM: its philosophy, architecture, capabilities, and roadmap. It intentionally excludes private implementation details.

---

## What Problem NAM Solves

Modern retrieval systems are optimized for *answers*.
NAM is optimized for *understanding*.

Most AI systems today:

* collapse meaning too early
* rely on similarity scoring as their only retrieval mechanism
* cannot explain *why* a result was retrieved
* struggle with partial, ambiguous, or exploratory queries

NAM addresses a different class of problems:

* "I don't know exactly what I'm looking for yet"
* "I need to explore before I can ask the right question"
* "I want to know which mechanism found each result, and why"
* "I need the core retrieval behavior to be stable, auditable, and repeatable"

---

## What NAM Is

NAM has the following defining characteristics:

* **Deterministic at the core**
  The address path produces the same addresses, scans, and results for the same input and data.

* **Address-based first**
  The core retrieval mechanism navigates semantic space rather than ranking neighbours.

* **Multi-dimensional**
  Records are placed into points, lines, planes, and volumes across attribute, affordance, and context axes, partitioned by ontology class and entity.

* **Converged and traceable**
  Address, vector, and graph paths run behind one endpoint; each result says which paths found it.

* **Exploratory by design**
  Ambiguity is preserved instead of collapsed.

* **Derived, not authoritative**
  NAM is never a system of record. All of its state is rebuildable from source data.

---

## What NAM Is Not

To avoid confusion, NAM is explicitly **not**:

* A vector database (it has a vector path, but that is one of three, and it is labelled approximate)
* A semantic search engine
* A traditional knowledge graph
* A chat history store
* A probabilistic ranking system
* A black-box retrieval layer

NAM does **not** optimize for "best answer."
It optimizes for **structured, explainable access to meaning**.

---

## How to Read This Documentation

This documentation is organized for different audiences:

* **Non-technical / GTM / Advisors**
  Start with:

  * `OVERVIEW/`
  * `PHILOSOPHY/`
  * `CAPABILITIES/`
  * `ROADMAP/`

* **Technical but non-implementers**
  Focus on:

  * [High-Level Systems](ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md)
  * [Geometric Retrieval](ARCHITECTURE/GEOMETRIC_RETRIEVAL.md)
  * [Addressing Model](ARCHITECTURE/ADDRESSING_MODEL.md)
  * [Query Model](ARCHITECTURE/QUERY_MODEL.md)
  * [Vector Search](ARCHITECTURE/VECTOR_SEARCH.md)
  * [Graph Index](ARCHITECTURE/GRAPH_INDEX.md)
  * [Pipeline Architecture](ARCHITECTURE/PIPELINE_ARCHITECTURE.md)
  * [Data Persistence](ARCHITECTURE/DATA_PERSISTENCE.md)

* **SDK developers and integrators**
  Focus on:

  * [API Reference](ARCHITECTURE/API_REFERENCE.md)
  * [Query Model](ARCHITECTURE/QUERY_MODEL.md)
  * [Security and Trust](GOVERNANCE/SECURITY_AND_TRUST.md)

* **Security and infrastructure reviewers**
  Focus on:

  * [Security and Trust](GOVERNANCE/SECURITY_AND_TRUST.md)
  * [Deployment Architecture](ARCHITECTURE/DEPLOYMENT_ARCHITECTURE.md)
  * [High-Level Systems](ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md)
  * [Design Principles](PHILOSOPHY/DESIGN_PRINCIPLES.md)

* **Potential collaborators**
  Review:

  * `GOVERNANCE/`
  * [Licensing Model](GOVERNANCE/LICENSING_MODEL.md)

This repo is designed to be readable **without access to the private codebase**.

---

## Design Philosophy (High Level)

NAM is built around a small set of non-negotiable principles:

* **Determinism over cleverness**
* **Geometry first, similarity labelled**
* **Projection over filtering**
* **Exploration over premature answers**
* **Rebuildability over mutation**
* **Separation of training from execution**
* **Every result explains its provenance**

These principles are expanded in [Design Principles](PHILOSOPHY/DESIGN_PRINCIPLES.md).

---

## Current State of the Project

NAM is in an **active production-pilot phase**.

* The ingest pipeline — rule-based NLP, BFO/CCO ontology classification, entity, attribute, affordance and context heads, entity-anchored bundling, learned-codec (LCA) addressing — runs continuously from a change data capture stream, with durable on-disk streams between stages, and sizes itself to its backlog
* Encoder heads are pluggable per slot; implementations developed outside NAM's own tree are published in the open-source `nam-heads` package
* The query service routes each query across the address, vector, and graph paths and fuses the results, with a per-path trace on every response
* The vector index returns the top records above a similarity floor, ranked by cosine; the graph index is built by a sharded change-stream consumer off the ingest critical path
* Storage is a multi-node key-value data service backed by S3-compatible object storage
* Authentication, transport security, and automated lifecycle management (bootstrap, health monitoring, maintenance tasks) are operational
* Deployment is a single templated chart

This is **not** a finished product, and this repo is intentionally transparent about current limitations.

> See: [Current State](ROADMAP/CURRENT_STATE.md) | [Current Limitations](CAPABILITIES/CURRENT_LIMITATIONS.md)

---

## Licensing & Access

NAM is **source-available**, not open source.

* Use, deployment, and redistribution require prior approval and licensing.
* Evaluation and pilot licenses may be granted on a case-by-case basis.
* The licensing model is designed to support future evolution (e.g., open-core), but no such commitment exists today.

See [Licensing Model](GOVERNANCE/LICENSING_MODEL.md) for specifics.

---

## Why This Repo Exists

This repository exists to:

* Establish a shared mental model for NAM
* Enable GTM conversations without exposing private internals
* Align collaborators on philosophy and constraints
* Prevent mischaracterization as "just another RAG system"
* Serve as the authoritative public reference until a website exists

If you are evaluating, advising, or helping bring NAM to market, **this repo is the source of truth**.

---

## Next Steps

If you're new:

1. Read [What Is NAM](OVERVIEW/WHAT_IS_NAM.md)
2. Read [Design Principles](PHILOSOPHY/DESIGN_PRINCIPLES.md)
3. Skim [Geometric Retrieval](ARCHITECTURE/GEOMETRIC_RETRIEVAL.md) and [Query Model](ARCHITECTURE/QUERY_MODEL.md)

If you're considering engagement, reach out through the appropriate private channel.

---

**NAM is about building systems that can reason *before* they answer.**
