# References & Lineage

This document lists **ideas, systems, and prior art** that influenced the design of Neural Addressable Memory (NAM).

These references are **conceptual inspirations**, not dependencies, endorsements, or claims of equivalence.
NAM intentionally diverges from many of these approaches.

---

## Knowledge Representation & Memory

### Addressable Memory Systems

* **Associative Memory (Hopfield Networks)**
  Early exploration of content-addressable memory.
  NAM shares the idea of retrieval by content rather than by ID, without stochastic recall.

* **Sparse Distributed Memory (Kanerva)**
  A foundational model for high-dimensional memory addressing.
  Influenced NAM’s geometric framing; NAM’s address path uses deterministic structure in place of probabilistic distance.

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
  Influenced NAM’s respect for relationships. NAM’s graph index is derived — records, the entities they mention and the classes those entities are stored under, written by an index builder — rather than a mutable property graph with a query language.

### Search & Indexing

* **Inverted Indexes (Lucene / Elasticsearch)**
  Showed the power of indexing, but rely on token matching and ranking heuristics.

* **OLAP Cubes**
  Influenced the geometric intuition (dimensions, slicing).
  NAM generalizes this idea beyond numeric aggregation.

---

## Formal Ontology

* **Basic Formal Ontology (BFO 2020, ISO/IEC 21838-2)**
  The upper-level ontology behind the type segment of every NAM address.

* **Common Core Ontologies (CCO)**
  The mid-level ontology whose classes (persons, organizations, acts, information content entities and so on) NAM’s ontology head assigns.

---

## Vector Search & Embeddings

### Dense Vector Retrieval

* **Word2Vec / GloVe / FastText**
  Demonstrated semantic proximity via learned embeddings.
  NAM uses embeddings as one access path, never as the only explanation of a result.

* **Approximate Nearest Neighbor (HNSW, FAISS, inverted-file partitioning)**
  Influential in modern retrieval pipelines.
  NAM’s vector path keeps its index in NAM’s own key-value store, reads only the part of it near a query, and labels every result with its cosine.

* **Residual vector quantization**
  Underlies the LCA address codec, which maps an axis value to a coarse and a fine code.

### RAG Architectures

* **Retrieval-Augmented Generation (RAG)**
  Acknowledged as the dominant pattern today.
  NAM exists to give RAG pipelines attributable, inspectable retrieval.

---

## Distributed Systems & Design Philosophy

### Deterministic Systems

* **Event-sourced systems and change data capture**
  Influenced NAM’s rebuildability: every index is derived from a change stream over source records.

* **Log-structured merge trees (RocksDB)**
  The storage substrate whose ordered keys make prefix scans the natural retrieval primitive.

* **Durable message logs (NATS JetStream)**
  The write-ahead log pattern NAM uses to acknowledge records only after their writes land.

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
It is an attempt to make **structure, determinism and attribution**
first-class primitives in knowledge systems, with similarity as a labelled complement rather than the foundation.

The references above represent starting points — not destinations.

→ See also: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [What Is NAM](../OVERVIEW/WHAT_IS_NAM.md)

