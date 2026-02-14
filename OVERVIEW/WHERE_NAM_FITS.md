# Where NAM Fits

NAM does not replace your database.
It does not replace your vector store.
It does not replace your knowledge graph.

It sits **between** them — and the systems that reason over them.

---

## The Problem With “Either / Or” Thinking

Most discussions frame systems as competitors:

* *RAG vs knowledge graphs*
* *Vector search vs SQL*
* *Graphs vs embeddings*

This framing is wrong.

These systems solve **different layers of the problem** — and fail when forced to do each other’s jobs.

NAM exists to occupy the layer that none of them handle well.

---

## The Missing Layer in Modern Stacks

Modern AI stacks usually look like this:

```
Raw Data → Storage → Retrieval → Generation → Output
```

What’s missing is **sense-making**.

NAM inserts a layer here:

```
Raw Data
   ↓
Storage (DB / Object Store / Event Stream)
   ↓
NAM (semantic addressing + exploration)
   ↓
Retrieval (targeted, deterministic)
   ↓
Generation / Analytics / Decisioning
```

NAM is the system that helps decide:

* *where* to look
* *how broadly* to look
* *what dimensions matter*
* *what to ignore for now*

---

## NAM vs RAG

### RAG

* retrieves based on similarity
* relies on embeddings
* optimized for recall
* opaque failure modes
* difficult to debug
* fragile across model changes

### NAM

* retrieves based on semantic structure
* uses deterministic addressing
* optimized for exploration
* transparent decision paths
* debuggable and auditable
* stable across model changes

**RAG answers questions.
NAM helps you figure out what question to ask next.**

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

* operates in discrete semantic space
* navigates by axes and partitions
* avoids ranking entirely
* preserves ambiguity
* enables precise control over scope

Vector databases are excellent **primitives**.
NAM is an **organizing system**.

---

## NAM vs Knowledge Graphs

### Knowledge Graphs

* require predefined schemas
* enforce early commitments
* expensive to maintain
* brittle when domains evolve
* strong when relationships are known

### NAM

* defers schema commitment
* allows partial understanding
* adapts to new domains
* supports exploratory queries
* strong when meaning is unclear

Knowledge graphs model **what you already know**.
NAM helps you discover **what you don’t yet understand**.

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
* a pre-RAG exploration layer
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

Trying to make it those things would break its core guarantees.

---

## The Positioning in One Sentence

**NAM is a semantic sense-making layer that sits between raw data and intelligent systems, enabling deterministic exploration before retrieval or generation.**

---

## When NAM Is the Wrong Tool

NAM may not be appropriate when:

* the domain is simple and stable
* similarity is sufficient
* speed matters more than correctness
* explainability is irrelevant

NAM trades convenience for clarity — intentionally.

---

## How It Fits Together

A common pattern:

```
NAM → targeted retrieval → RAG → generation
```

NAM narrows the space.
Retrieval pulls the facts.
RAG handles language.

Each does what it's good at.

→ See also: [What Is NAM](WHAT_IS_NAM.md) | [What NAM Can Do](../CAPABILITIES/WHAT_NAM_CAN_DO.md) | [What NAM Cannot Do](../CAPABILITIES/WHAT_NAM_CANNOT_DO.md)