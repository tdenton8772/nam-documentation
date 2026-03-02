# The NAM Query Model

*How questions become probes*

This document explains how a natural-language question is transformed into a deterministic set of storage probes in NAM.

NAM does not “search” in the traditional sense.
It **plans** a query and **executes probes** against known semantic locations.

---

## Queries Are Not Requests for Answers

In NAM, a query is **not** a request for a ranked list of documents.

A query is:

> A request to explore a specific region of semantic space.

NAM’s job is not to decide *what the answer is*.
Its job is to decide *where meaningful answers could exist* — and retrieve what is already there.

---

## High-Level Query Flow

At a conceptual level, every query follows this flow:

1. **Parse the question**
2. **Encode meaning using the same heads as ingest**
3. **Determine query mode**
4. **Plan address probes**
5. **Execute probes**
6. **Return records (unranked or lightly ordered)**

No step is optional.
No step is probabilistic.

---

## Step 1: Query Parsing (Input Normalization)

The raw query text is accepted as-is.

NAM does not:

* Rewrite the question
* Simplify grammar
* Ask clarifying questions
* Hallucinate intent

The system assumes:

> The user’s ambiguity is meaningful.

Ambiguity is preserved, not eliminated.

---

## Step 2: Encoder Heads Run in Query Mode

The same encoder heads used at ingest are invoked again — but in **query mode**.

Examples:

* Entity head extracts candidate entities
* Ontology head activates possible semantic domains
* Affordance head identifies what kind of action or intent is implied
* Context head supplies environmental or situational signals

Entity resolution at query time is type-scoped: a person lookup for "Einstein" only searches within person-type entities, preventing cross-type semantic bleed. Verbs are treated as first-class entities — "study" resolves through the same entity store as "Einstein," with synonym folding to match canonical forms established at ingest.

**Critical rule:**
Query encoders must obey the same axis definitions and vocabularies as ingest.

No head may invent:

* New axes
* New axis values
* New semantics

---

## Step 3: Query Mode Determination

NAM does not treat all queries equally.

Based on encoder output, the planner selects one of two **query modes**:

### Exploratory Mode (default)

Used when no specific affordance verb is recognized in the query. The system explores broadly across ontology tiers.

* Ontology tiers are probed in a predefined order, with tiers containing the bundler's ontology **hints** probed first
* Each tier groups related ontologies (e.g., Tier 1: person, object, location, concept; Tier 2: time, event, action; etc.)
* The system probes progressively from most-specific addresses to least-specific within each tier

### Affordance Mode

Used when an affordance verb is recognized (e.g., "Where is...", "What did... do", "Who leads..."). The affordance selects specific ontology tiers and execution strategies.

* Affordance verbs undergo synonym folding to match canonical forms established at ingest
* Each affordance maps to a set of ontology-role capabilities (pre-computed)
* Probing is narrower and more targeted than exploratory mode

The mode is **derived**, not specified by the user.

---

## Step 4: Ontology Selection (Domains to Probe)

The ontology head activates a *set* of candidate domains.

Example:

```
['person', 'location', 'object', 'activity']
```

Important rules:

* Ontologies are allowed to over-activate
* Over-activation is safer than exclusion
* Ontology selection never filters records directly

Ontologies only define *where probes may be constructed*.

---

## Step 5: Address Planning

This is the core of the query model.

The planner:

* Takes ontology + entity + axis signals from encoding
* Captures **ontology hints** from the bundler (what types the bundler actually observed)
* Reorders ontology tiers so that hinted tiers are probed first
* Constructs a **finite, budget-bounded set of concrete addresses**
* Orders them from most specific to most general within each tier

Each planned address is a valid KV lookup.

Examples:

* Point probes (all axes populated: entity + attribute + affordance + context)
* Line probes (one axis wildcarded)
* Plane probes (two axes wildcarded)
* Volume probes (only entity specified, all other axes wildcarded)

No “contains”, no “similar to”, no post-filtering.

Planning is **budget-aware**: the total number of planned addresses is capped to prevent unbounded exploration.

---

## Step 6: Probe Execution (Progressive Fan-Out)

Each planned address becomes a **direct storage probe**, but probes are executed using a **progressive fan-out** strategy:

1. Within each ontology tier, addresses are sorted by **specificity** (how many axes are populated)
2. The most specific addresses are probed first (specificity 3: all axes set)
3. Then less specific (specificity 2, 1, 0)
4. After completing a tier, if enough matches have been found (above a **satisfaction threshold**), remaining tiers are skipped

For each probe:

* A partition key is computed
* The storage backend is queried directly
* Matching records are collected and deduplicated

### Early termination

Progressive fan-out enables **early termination**: if a tier yields sufficient results, the system stops probing rather than exhaustively scanning all planned addresses. This bounds query latency while preserving determinism — the same query with the same data always terminates at the same point.

### Budget limits

Execution is bounded by:

* A maximum number of address probes
* A maximum number of document fetches
* A satisfaction threshold per tier

These limits ensure that even highly ambiguous queries complete in bounded time.

---

## Step 7: Controlled Widening (Not Guessing)

NAM does not “broaden” queries by:

* Removing constraints arbitrarily
* Dropping entities
* Falling back to similarity

Widening occurs through structured mechanisms:

* **Specificity descent** — within a tier, probes progress from fully-specified (point) to partially-specified (line, plane)
* **Ontology tier expansion** — when a tier is exhausted without satisfaction, the next tier is probed
* **Hint-prioritized ordering** — tiers containing ontologies observed by the bundler are probed before others
* **Explicit wildcard axes** — `__null__` coordinates widen the geometric region
* **Codebook neighborhood expansion** — when using LCA byte-code coordinates, each axis's coarse code is expanded to include its k nearest codebook neighbors (by cosine distance). This probes semantically adjacent regions of the address space without abandoning the axis entirely

Every widening step is explainable: the system can report which tier it probed, at what specificity, and whether satisfaction was reached.

---

## Results Are Not Ranked by Similarity

NAM returns:

* Records that exist at probed addresses

Ordering (if any) reflects:

* Address specificity
* Probe order
* Retrieval time

NAM does **not** compute semantic similarity scores at query time.

Any ranking beyond this is the responsibility of:

* The client
* Or a downstream system

---

## Why Queries Can Return “Unexpected” Results

This is a feature, not a flaw.

Because:

* Ontologies may be broad
* Queries may be exploratory
* Addresses may intersect unexpectedly

NAM may surface records the user did not anticipate — but which are semantically adjacent in the space they asked to explore.

This is how **sense-making precedes certainty**.

---

## Failure Is Explicit and Informative

If a query returns no results, it means one of the following is true:

* No records were ingested at those addresses
* The addressing contract was violated at ingest
* The query was more specific than the data supports

NAM does not mask this with fallback heuristics.

---

## Query vs Retrieval vs Reasoning

NAM:

* Plans queries
* Executes probes
* Returns records

NAM does **not**:

* Summarize
* Reason
* Explain
* Generate answers

Those are downstream concerns.

This separation is what makes NAM composable.

---

## Summary

* Queries are transformed into address probes
* The same encoders are used for ingest and query
* Planning is deterministic and bounded
* Widening is explicit, not heuristic
* Retrieval is exact-key based

A NAM query is not a search.
It is a **geometric exploration of semantic space**.

→ See also: [Ingestion Model](INGESTION_MODEL.md) | [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)
