# The NAM Query Model

*How questions become geometric scans*

This document explains how a natural-language question is transformed into a deterministic set of storage operations in NAM.

NAM does not “search” in the traditional sense.
It **plans** a query and **executes prefix scans** against known regions of semantic space.

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
4. **Plan prefix scans on the covering index**
5. **Execute scans**
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

* Ontology tiers are scanned in a predefined order, with tiers containing the bundler's ontology **hints** scanned first
* Each tier groups related ontologies (e.g., Tier 1: person, object, location, concept; Tier 2: time, event, action; etc.)
* The system scans progressively from most-specific addresses to least-specific within each tier

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

Ontologies only define *where scans may be constructed*.

---

## Step 5: Address Planning

This is the core of the query model.

The planner:

* Takes ontology + entity + axis signals from encoding
* Captures **ontology hints** from the bundler (what types the bundler actually observed)
* Reorders ontology tiers so that hinted tiers are probed first
* Constructs a **finite, budget-bounded set of prefix scan queries**
* Orders them from most specific to most general within each tier

Each planned query selects the optimal rotation of the covering index where the specified axes form a contiguous key prefix, then constructs a prefix scan:

* **Point scan** (all axes specified) — prefix covers exact coordinates
* **Line scan** (two axes specified) — prefix covers two coordinates, scans across the third
* **Plane scan** (one axis specified) — prefix covers one coordinate, scans across two others
* **Volume scan** (no axes specified, partition only) — scans entire partition

No “contains”, no “similar to”, no post-filtering.

Planning is **budget-aware**: the total number of planned scans is capped to prevent unbounded exploration.

---

## Step 6: Scan Execution (Progressive Fan-Out)

Each planned prefix scan query is executed against the storage layer using a **progressive fan-out** strategy:

1. Within each ontology tier, scans are sorted by **specificity** (how many axes are specified in the prefix)
2. The most specific scans are executed first (specificity 3: all axes in prefix)
3. Then less specific (specificity 2, 1, 0)
4. After completing a tier, if enough matches have been found (above a **satisfaction threshold**), remaining tiers are skipped

For each scan:

* The optimal rotation is selected (where specified axes form a key prefix)
* A prefix scan retrieves all matching records in a single storage operation
* Results are collected and deduplicated across scans

### Early termination

Progressive fan-out enables **early termination**: if a tier yields sufficient results, the system stops scanning rather than exhaustively exploring all planned regions. This bounds query latency while preserving determinism — the same query with the same data always terminates at the same point.

### Budget limits

Execution is bounded by:

* A maximum number of prefix scans
* A maximum number of document fetches
* A satisfaction threshold per tier

These limits ensure that even highly ambiguous queries complete in bounded time.

---

## Codebook Neighborhood Fan-Out

Because coordinates are LCA byte codes (not raw strings), the query engine can exploit the **geometric structure of the codebook** to widen scans systematically.

At startup, the query service precomputes a neighbor table: for each codebook entry, the k nearest entries by cosine distance. At query time, each axis's code is expanded to include its k nearest neighbors (default k=3), producing 4 candidate codes per axis (1 exact + 3 neighbors).

The Cartesian product of these candidates creates a deterministic set of address coordinates. Each unique combination of specified axes then maps to a single prefix scan on the appropriate rotation of the covering index.

This mechanism creates **deterministic ordering** of scan priorities: exact-match codes are scanned first, then progressively more distant neighbors. The ordering is stable across identical queries. Combined with the covering index, neighborhood expansion adds recall without multiplying the number of storage operations — each unique prefix is scanned once regardless of how many neighbor combinations map to it.

---

## Step 7: Controlled Widening (Not Guessing)

NAM does not “broaden” queries by:

* Removing constraints arbitrarily
* Dropping entities
* Falling back to similarity

Widening occurs through structured mechanisms:

* **Specificity descent** — within a tier, scans progress from fully-specified (all axes in prefix) to partially-specified (fewer axes, broader scan region)
* **Ontology tier expansion** — when a tier is exhausted without satisfaction, the next tier is scanned
* **Hint-prioritized ordering** — tiers containing ontologies observed by the bundler are scanned before others
* **Rotation selection** — the covering index guarantees that any combination of specified axes can be efficiently scanned via the appropriate rotation

Every widening step is explainable: the system can report which tier it scanned, at what specificity, and whether satisfaction was reached.

---

## Results Are Not Ranked by Similarity

NAM returns:

* Records that exist at scanned address regions

Ordering (if any) reflects:

* Address specificity
* Scan order
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
* Executes scans
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

* Queries are transformed into prefix scans on a covering index
* The same encoders are used for ingest and query
* Planning is deterministic and bounded
* Widening is explicit, not heuristic
* Retrieval uses structured prefix scans, not similarity

A NAM query is not a search.
It is a **geometric exploration of semantic space**.

→ See also: [Ingestion Model](INGESTION_MODEL.md) | [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Pipeline Architecture](PIPELINE_ARCHITECTURE.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)
