# Neural Addressable Memory (NAM)

**Neural Addressable Memory (NAM)** is a deterministic, geometry-based retrieval system designed to support **sense-making before certainty**.

NAM is not a vector database, not a knowledge graph, and not a traditional RAG system.
It is a different approach to retrieval—one that treats meaning as **addressable structure**, not ranked similarity.

This repository contains the **public documentation** for NAM: its philosophy, architecture, capabilities, and roadmap. It intentionally excludes private implementation details.

---

## What Problem NAM Solves

Modern retrieval systems are optimized for *answers*.
NAM is optimized for *understanding*.

Most AI systems today:

* collapse meaning too early
* rely on similarity scoring instead of structure
* behave non-deterministically
* cannot explain *why* a result was retrieved
* struggle with partial, ambiguous, or exploratory queries

NAM addresses a different class of problems:

* “I don’t know exactly what I’m looking for yet”
* “I need to explore before I can ask the right question”
* “I want guarantees about what can and cannot be retrieved”
* “I need retrieval behavior to be stable, auditable, and repeatable”

---

## What NAM Is

NAM is a **geometric retrieval system** with the following defining characteristics:

* **Deterministic**
  The same input produces the same addresses, probes, and results.

* **Address-based, not similarity-based**
  Retrieval is done by navigating semantic space, not ranking neighbors.

* **Multi-dimensional**
  Information is placed into points, lines, planes, and volumes across multiple semantic axes.

* **Exploratory by design**
  Ambiguity is preserved instead of collapsed.

* **Derived, not authoritative**
  NAM is never a system of record. All state is rebuildable from source data.

---

## What NAM Is Not

To avoid confusion, NAM is explicitly **not**:

* ❌ A vector database
* ❌ A semantic search engine
* ❌ A traditional knowledge graph
* ❌ A chat history store
* ❌ A probabilistic ranking system
* ❌ A black-box retrieval layer

NAM does **not** optimize for “best answer.”
It optimizes for **structured access to meaning**.

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

  * `ARCHITECTURE/`
  * `GEOMETRIC_RETRIEVAL.md`
  * `ADDRESSING_MODEL.md`
  * `QUERY_MODEL.md`

* **Potential collaborators**
  Review:

  * `GOVERNANCE/`
  * `LICENSING_MODEL.md`

This repo is designed to be readable **without access to the private codebase**.

---

## Design Philosophy (High Level)

NAM is built around a small set of non-negotiable principles:

* **Determinism over cleverness**
* **Geometry over similarity**
* **Projection over filtering**
* **Exploration over premature answers**
* **Rebuildability over mutation**
* **Separation of training from execution**

These principles are expanded in `PHILOSOPHY/DESIGN_PRINCIPLES.md`.

---

## Current State of the Project

NAM is **actively under development**.

* The core addressing and query model is implemented and exercised against real corpora.
* The system currently operates in **exploratory mode** by default.
* Affordance-guided execution policies are being trained offline.
* Entity resolution and refinement are ongoing areas of improvement.

This is **not** a finished product, and this repo is intentionally transparent about current limitations.

See `ROADMAP/CURRENT_STATE.md` for details.

---

## Licensing & Access

NAM is **source-available**, not open source.

* Use, deployment, and redistribution require prior approval and licensing.
* Evaluation and pilot licenses may be granted on a case-by-case basis.
* The licensing model is designed to support future evolution (e.g., open-core), but no such commitment exists today.

See `GOVERNANCE/LICENSING_MODEL.md` for specifics.

---

## Why This Repo Exists

This repository exists to:

* Establish a shared mental model for NAM
* Enable GTM conversations without exposing private internals
* Align collaborators on philosophy and constraints
* Prevent mischaracterization as “just another RAG system”
* Serve as the authoritative public reference until a website exists

If you are evaluating, advising, or helping bring NAM to market, **this repo is the source of truth**.

---

## Next Steps

If you’re new:

1. Read `OVERVIEW/WHAT_IS_NAM.md`
2. Read `PHILOSOPHY/DESIGN_PRINCIPLES.md`
3. Skim `ARCHITECTURE/GEOMETRIC_RETRIEVAL.md`

If you’re considering engagement, reach out through the appropriate private channel.

---

**NAM is about building systems that can reason *before* they answer.**
