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

Based on encoder output, the planner selects a **query mode**, such as:

* Exploratory
* Descriptive
* Lookup-like
* Pattern-seeking

The mode affects:

* How wide the query is allowed to go
* Which ontologies are eligible
* How aggressively wildcarding is applied

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

* Takes ontology + entity + axis signals
* Constructs a **finite set of concrete addresses**
* Orders them from most specific → most general

Each planned address is a valid KV lookup.

Examples:

* Point probes (x,y,z fully specified)
* Line probes (one axis wildcarded)
* Plane probes (two axes wildcarded)

No “contains”, no “similar to”, no post-filtering.

---

## Step 6: Probe Execution

Each planned address becomes a **direct storage probe**.

For each probe:

* A partition key is computed
* A bucket is selected
* The storage backend is queried directly

If a probe returns records:

* They are collected
* Deduplicated
* Associated with the probe that retrieved them

If a probe returns nothing:

* The planner proceeds to the next address

This is intentional widening, not retry logic.

---

## Step 7: Controlled Widening (Not Guessing)

NAM does not “broaden” queries by:

* Removing constraints arbitrarily
* Dropping entities
* Falling back to similarity

Widening only occurs via:

* Explicit wildcard axes (`__null__`)
* Ontology tier expansion
* Planner-defined rules

Every widening step is explainable.

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

→ See also: [Ingestion Model](INGESTION_MODEL.md) | [Geometric Retrieval](GEOMETRIC_RETREIVAL.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md)
