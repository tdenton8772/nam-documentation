# What NAM Cannot Do

NAM is deliberately constrained by design.
Those constraints are what keep it inspectable, rebuildable and deployable.

This document describes **systemic non-capabilities** — not temporary limitations, missing features, or roadmap items (see [Current Limitations](CURRENT_LIMITATIONS.md) for those).

If a problem fundamentally requires any of the following, **NAM is not the right tool**.

---

## 1. Opaque Relevance

NAM does contain similarity search: the vector path returns the top-K records above a cosine floor. What NAM does not do is hide *how* a result was found.

NAM will not:

* blend paths into a single unexplained relevance score
* return a result without saying which access path produced it
* present a vector match as if it were a structural match

Every fused row lists its `sources`, vector rows carry their cosine, and every response carries an `access_paths` trace. If a use case needs a black-box relevance score with no provenance, NAM is not a fit.

---

## 2. Exhaustive Similarity Search

The vector path is approximate. NAM does not guarantee:

* the globally nearest K records
* every record above a similarity threshold
* results below the deployment's cosine floor, which a request cannot lower

If a use case needs exact nearest-neighbour search over a whole corpus, use a system built for that.

---

## 3. Semantic Guessing or Intent Invention

NAM does not invent user intent.

It will not:

* infer meaning beyond what the query's words and resolved entities support
* invent entities that are not in the entity store
* reinterpret vague input to "be helpful"

Ambiguous queries remain ambiguous. NAM widens explicitly, and the trace shows it.

---

## 4. Natural Language Generation or Reasoning

NAM does not:

* generate prose
* summarize content
* rewrite text
* produce explanations in natural language

NAM retrieves **records**, not narratives. It is designed to be paired with downstream systems, LLMs and application logic.

---

## 5. Learning New Rules at Runtime

NAM does not change its navigation rules in response to queries.

It will not:

* create new ontology classes on the fly
* retrain encoder heads, the address codec or the embedding model at runtime
* adapt its addressing strategy based on query feedback

NAM accumulates *knowledge* during ingest (new addresses, new entities and surface forms). The rules that place and find knowledge stay fixed for a given set of artifacts.

---

## 6. Global Recall Guarantees

NAM does not guarantee:

* full recall
* exhaustive coverage
* "all relevant results"

Each path is bounded: the address path by its planned probes and budgets, the vector path by the bounded part of the index each query reads and by its cosine floor, the graph path by hop depth and result caps. A record outside every path's bounds is not returned, even if it is conceptually related. Bounded retrieval is a design choice: it keeps cost predictable and behaviour explainable.

---

## 7. Joins, Aggregations and Analytics

NAM does not perform:

* relational joins
* aggregations
* statistical analysis
* ad hoc schema inference

It does not behave like a data warehouse, an analytics engine, or a graph query language. The graph path walks fixed relationship types; it does not evaluate arbitrary graph queries.

---

## 8. Being a System of Record

NAM is not a source of truth.

It should not be used as:

* an authoritative datastore
* a transactional system
* a canonical record keeper

Every NAM index — addresses, vectors, graph — is derived from source records and can be rebuilt from them. NAM is a memory layer, not a ledger.

---

## 9. Universal Applicability

NAM is opinionated. It is not intended to solve every search, knowledge or AI problem. If a problem is best served by heuristics without provenance, probabilistic inference, or opaque decision-making, NAM will feel restrictive — and that is by design.

---

## Summary

NAM does **not** try to be:

* an opaque relevance engine
* a reasoning engine
* a runtime learning system
* a generative model
* a system of record

NAM exists to do one thing well:

> **Provide inspectable, attributable access to semantic knowledge — by structure first, by similarity where it helps, and always saying which.**

→ See also: [What NAM Can Do](WHAT_NAM_CAN_DO.md) | [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [Where NAM Fits](../OVERVIEW/WHERE_NAM_FITS.md)
