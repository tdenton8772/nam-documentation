# What NAM Cannot Do

NAM is deliberately constrained by design.
Those constraints are what make it deterministic, inspectable, and deployable.

This document describes **systemic non-capabilities** — not temporary limitations, missing features, or future roadmap items.

If a problem fundamentally requires any of the following, **NAM is not the right tool**.

---

## 1. Probabilistic or Best-Guess Search

NAM does not perform:

* similarity ranking
* fuzzy matching
* “top-K” retrieval based on embeddings
* relevance scoring without semantic justification

There is no concept of:

* “closest match”
* “most similar”
* “probably relevant”

If a use case depends on:

* approximate answers
* recall-first discovery
* “something like this” behavior

NAM is not a fit.

---

## 2. Semantic Guessing or Intent Inference

NAM does not guess user intent.

It will not:

* infer meaning beyond what the query activates
* hallucinate affordances
* reinterpret vague input to “be helpful”

Ambiguous queries remain ambiguous.

This is intentional.

If a system must:

* always return *something*
* reinterpret unclear questions
* smooth over underspecified intent

NAM will feel rigid — because it is.

---

## 3. Natural Language Generation or Creative Reasoning

NAM does not:

* generate prose
* summarize content
* rewrite text
* create explanations in natural language

NAM retrieves **knowledge**, not narratives.

It is designed to be paired with:

* downstream systems
* LLMs
* application logic

NAM provides **what is known** — not how to phrase it.

---

## 4. Learning New Reasoning Strategies at Runtime

NAM does not evolve its reasoning rules dynamically.

It will not:

* invent new affordances
* create new ontologies on the fly
* adapt its addressing strategy in response to queries

While NAM accumulates more knowledge over time, the *rules of navigation* remain fixed.

If a system requires:

* adaptive inference strategies
* reinforcement learning loops
* self-modifying reasoning

NAM is not designed for that role.

---

## 5. Global Recall Guarantees

NAM does not guarantee:

* full recall
* exhaustive coverage
* “all relevant results”

Retrieval is constrained by:

* geometry
* addressing choices
* probe shape

If a record lies outside the probed region, it will not be returned — even if it is conceptually related.

This is a feature, not a bug.

If a use case demands:

* “never miss anything”
* full-corpus scans by default
* recall-first guarantees

NAM is not appropriate.

---

## 6. Implicit Joins or Schema-Free Analytics

NAM does not perform:

* relational joins
* aggregations
* statistical analysis
* ad hoc schema inference

It does not behave like:

* a data warehouse
* an analytics engine
* a graph query language

NAM retrieves **records**, not computed results.

---

## 7. Being a System of Record

NAM is not a source of truth.

It should not be used as:

* an authoritative datastore
* a transactional system
* a canonical record keeper

NAM assumes:

* upstream systems own correctness
* data can be re-ingested
* addresses can be rebuilt

It is a memory layer, not a ledger.

---

## 8. Universal Applicability

NAM is not intended to solve:

* every search problem
* every knowledge problem
* every AI problem

It is intentionally opinionated.

If a problem benefits from:

* ambiguity
* heuristics
* probabilistic inference
* opaque decision-making

NAM will feel restrictive — and that is by design.

---

## Summary

NAM does **not** try to be:

* a search engine
* a reasoning engine
* a learning system
* a generative model
* a universal database

NAM exists to do one thing well:

> **Provide deterministic, inspectable access to semantic knowledge through geometry.**

If a problem fits that shape, NAM excels.
If it doesn't, NAM will resist — and should.

→ See also: [What NAM Can Do](WHAT_NAM_CAN_DO.md) | [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [Where NAM Fits](../OVERVIEW/WHERE_NAM_FITS.md)
