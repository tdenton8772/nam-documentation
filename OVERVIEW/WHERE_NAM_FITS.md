# Where NAM Fits

NAM does not replace your database.
It does not replace your knowledge graph.
It is not trying to be the world's best vector store.

It sits **between** your data and the systems that reason over it.

---

## The Problem With "Either / Or" Thinking

Most discussions frame systems as competitors:

* *RAG vs knowledge graphs*
* *Vector search vs SQL*
* *Graphs vs embeddings*

This framing is wrong.

These systems solve **different layers of the problem** — and fail when forced to do each other's jobs.

NAM takes a converged position: structured addressing as the core, with a vector path and a graph path beside it, all behind one query and all reporting which of them found each result.

---

## The Missing Layer in Modern Stacks

Modern AI stacks usually look like this:

```
Raw Data → Storage → Retrieval → Generation → Output
```

What's missing is **sense-making**.

NAM inserts a layer here:

```
Raw Data
   ↓
Storage (DB / Object Store / Event Stream)
   ↓
NAM (semantic addressing + vector + graph, fused and traced)
   ↓
Retrieval (targeted, explainable)
   ↓
Generation / Analytics / Decisioning
```

NAM is the system that helps decide:

* *where* to look
* *how broadly* to look
* *what dimensions matter*
* *which mechanism a result came from*

---

## NAM vs RAG

### RAG (as usually built)

* retrieves based on similarity alone
* relies on embeddings
* optimized for recall
* opaque failure modes
* difficult to debug
* fragile across model changes

### NAM

* retrieves first by semantic structure, with similarity as a labelled complement
* uses deterministic addressing at its core
* optimized for exploration
* transparent decision paths: every response traces each path
* debuggable and auditable
* the address path does not depend on an embedding model

**RAG answers questions.
NAM helps you figure out what question to ask next — and can feed RAG with results whose provenance is known.**

They can be used together — but they are not interchangeable.

---

## NAM vs Vector Databases

### Vector Databases

* operate in continuous embedding space
* rely on distance metrics
* require ranking
* struggle with explainability
* blur semantic boundaries

### NAM

* its core operates in discrete semantic space
* navigates by axes and partitions
* ranks nothing on the address path
* preserves ambiguity
* enables precise control over scope

NAM does include a vector path — but it is one path of three, it is explicitly approximate (top results above a similarity floor), and its contribution to every result is labelled. Vector search inside NAM is a primitive NAM organizes, not the organizing principle.

---

## NAM vs Knowledge Graphs

### Knowledge Graphs

* require predefined schemas
* enforce early commitments
* expensive to maintain
* brittle when domains evolve
* strong when relationships are known

### NAM

* types records with a standard upper ontology, but defers further schema commitment
* allows partial understanding
* adapts to new domains
* supports exploratory queries
* strong when meaning is unclear

NAM's graph index is not a hand-curated knowledge graph. It is derived mechanically from the addresses NAM already wrote: which records mention which entities, and which entities occur together.

Knowledge graphs model **what you already know**.
NAM helps you discover **what you don't yet understand**.

---

## NAM vs Search Engines

### Search

* keyword-based
* ranking-driven
* optimized for popularity
* context-poor
* shallow semantics

NAM:

* semantic-first
* structure-aware
* context-sensitive
* supports reasoning paths
* prioritizes meaning over popularity

Search finds documents.
NAM navigates **meaning**.

---

## NAM vs Databases (SQL / NoSQL)

Databases:

* store facts
* enforce consistency
* support transactions
* answer well-defined questions

NAM:

* does not store authoritative data
* derives all state
* supports exploration
* enables semantic navigation

**NAM is never the system of record.**

It depends on databases — it does not replace them.

---

## How NAM Is Typically Used

NAM commonly acts as:

* a semantic router for retrieval systems
* a pre-RAG exploration and grounding layer
* an AI agent memory substrate
* a reasoning scaffold for analytics
* a semantic index over heterogeneous data

It is most valuable when:

* the problem is not fully specified
* multiple interpretations are valid
* recall alone is insufficient
* trust and explainability matter

---

## What NAM Is Not Trying To Be

NAM is intentionally **not**:

* a universal knowledge store
* a ranking engine
* a probabilistic recommender
* a language model
* a graph database replacement
* a general-purpose vector database

Trying to make it those things would break its core guarantees.

---

## The Positioning in One Sentence

**NAM is a converged semantic index that sits between raw data and intelligent systems: deterministic exploration at its core, with labelled vector and graph paths, before retrieval or generation.**

---

## When NAM Is the Wrong Tool

NAM may not be appropriate when:

* the domain is simple and stable
* similarity alone is sufficient
* speed matters more than correctness
* explainability is irrelevant

NAM trades convenience for clarity — intentionally.

---

## How It Fits Together

A common pattern:

```
NAM → targeted retrieval → RAG → generation
```

NAM narrows the space and says how each result was found.
Retrieval pulls the facts.
RAG handles language.

Each does what it's good at.

→ See also: [What Is NAM](WHAT_IS_NAM.md) | [What NAM Can Do](../CAPABILITIES/WHAT_NAM_CAN_DO.md) | [What NAM Cannot Do](../CAPABILITIES/WHAT_NAM_CANNOT_DO.md) | [Vector Search](../ARCHITECTURE/VECTOR_SEARCH.md) | [Graph Index](../ARCHITECTURE/GRAPH_INDEX.md)
