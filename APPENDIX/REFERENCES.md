# References & Lineage

This document lists **ideas, systems, and prior art** that influenced the design of Neural Addressable Memory (NAM).

These references are **conceptual inspirations**, not dependencies, endorsements, or claims of equivalence.
NAM intentionally diverges from many of these approaches.

---

## Knowledge Representation & Memory

### Addressable Memory Systems

* **Associative Memory (Hopfield Networks)**
  Early exploration of content-addressable memory.
  NAM shares the idea of retrieval by structure, not by ID — but avoids learned weights and stochastic recall.

* **Sparse Distributed Memory (Kanerva)**
  A foundational model for high-dimensional memory addressing.
  Influenced NAM’s geometric framing, but NAM replaces probabilistic distance with deterministic structure.

### Cognitive & Symbolic Systems

* **Frames and Scripts (Minsky)**
  Early attempts to structure meaning explicitly.
  NAM rejects hard-coded semantic frames in favor of emergent address families.

* **Symbolic AI & Expert Systems**
  Demonstrated the power — and brittleness — of rule-driven reasoning.
  NAM intentionally avoids rule execution at query time.

---

## Databases & Retrieval Systems

### Graph & Knowledge Systems

* **RDF / SPARQL / Semantic Web**
  Strong influence as a counterexample.
  NAM rejects explicit triples, joins, and query languages that require prior schema agreement.

* **Knowledge Graphs (Neo4j, property graphs)**
  Influenced NAM’s respect for relationships, but NAM avoids traversal, edges, and mutable graph state.

### Search & Indexing

* **Inverted Indexes (Lucene / Elasticsearch)**
  Showed the power of indexing, but rely on token matching and ranking heuristics.

* **OLAP Cubes**
  Influenced the geometric intuition (dimensions, slicing).
  NAM generalizes this idea beyond numeric aggregation.

---

## Vector Search & Embeddings

### Dense Vector Retrieval

* **Word2Vec / GloVe / FastText**
  Demonstrated semantic proximity via learned embeddings.
  NAM rejects similarity as a primitive.

* **Approximate Nearest Neighbor (HNSW, FAISS)**
  Influential in modern retrieval pipelines.
  NAM avoids probabilistic neighbors entirely.

### RAG Architectures

* **Retrieval-Augmented Generation (RAG)**
  Acknowledged as the dominant pattern today.
  NAM exists specifically to solve RAG’s determinism, explainability, and control problems.

---

## Distributed Systems & Design Philosophy

### Deterministic Systems

* **Event-sourced systems**
  Influenced NAM’s immutability and rebuildability principles.

* **Functional programming models**
  Inspired NAM’s insistence on pure functions for address generation.

### Observability & Auditability

* **Operational telemetry systems (e.g., OTEL)**
  Reinforced the importance of traceability and replay.

---

## Philosophy & Systems Thinking

* **“Simple vs Easy” (Rich Hickey)**
  A guiding principle in resisting convenience abstractions.

* **Second-order effects in system design**
  NAM prioritizes long-term system integrity over short-term capability wins.

---

## Explicit Non-Influences

For clarity, NAM intentionally does **not** draw from:

* Neural network fine-tuning
* End-to-end learned reasoning systems
* Probabilistic inference engines
* Autonomous agent frameworks

These omissions are deliberate.

---

## Closing Note

NAM is not an incremental improvement on an existing category.
It is an attempt to re-establish **structure, determinism, and geometry**
as first-class primitives in knowledge systems.

The references above represent starting points — not destinations.

→ See also: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [What Is NAM](../OVERVIEW/WHAT_IS_NAM.md)

