# Design Principles

NAM is not defined by its implementation details.
It is defined by the **constraints it refuses to violate**.

These principles are not aspirational.
They are *design laws* that shape every architectural decision, even when doing so is inconvenient.

---

## 1. Determinism Is a First Principle

Given the same inputs, NAM's core must produce the same outputs.

Always.

The address path has no hidden state, no probabilistic ranking, and no stochastic fallback. The same text, with the same artifacts, produces the same addresses on every machine, and the same query against the same data scans the same regions and returns the same records.

Where NAM uses an approximate mechanism — the vector path — it is labelled as approximate, it is held to a relevance floor, and its contribution to every result is visible.

### Why this matters

* Determinism makes systems **explainable**
* Determinism makes systems **debuggable**
* Determinism makes systems **deployable**
* Determinism makes failures **actionable**

NAM is designed to be trusted in environments where *"the model said so"* is not an acceptable explanation.

If meaning cannot be reproduced, it cannot be operationalized.

---

## 2. Geometry First, Similarity Labelled

NAM's core does not retrieve "the most similar results."

It **projects queries into structured semantic space** and retrieves records that occupy compatible positions.

This distinction matters.

Similarity-based retrieval, used alone:

* collapses nuance into a score
* hides reasoning inside distance functions
* requires tuning, thresholds, and re-ranking

NAM's address path instead operates on:

* explicit axes
* explicit partitions (ontology class and entity)
* explicit coordinates
* explicit traversal rules

The result is **navigation, not ranking**.

NAM also runs a vector path, because some questions are about paraphrase and topical neighbourhood that no structural address captures. The rule is that similarity is never smuggled in: a vector result carries its cosine, the vector path enforces a relevance floor, and every fused result lists which paths found it.

---

## 3. Ambiguity Is Preserved, Not Resolved

NAM is designed for **sense-making before certainty**.

When meaning is ambiguous:

* NAM keeps the ambiguity
* NAM represents multiple interpretations in parallel
* NAM allows downstream systems to decide what to do next

When the ontology classifier is not confident, it backs off to a broader class, and ultimately to the unclassified root of the ontology, rather than guessing. NAM does *not*:

* force early resolution
* guess intent
* optimize for "likely correctness"

This allows systems built on NAM to **explore** before they **decide**.

---

## 4. Field-Based Training, Not Runtime Learning

NAM's models learn **offline**, not at query time.

The ontology codebook, the learned coordinate codec (LCA), the affordance capability maps, and the vector index's trained partition are all static artifacts:

* versioned
* validated strictly when loaded — a component refuses to start on a mismatched artifact rather than fall back to defaults
* deployed explicitly

At runtime:

* no model is trained
* no query changes system behaviour
* nothing about navigation rules mutates

Ingest does accumulate vocabulary — new entities, surface forms, and category mappings in the session store — and the vector index grows as records are filed into it. Neither changes how a given address is computed or how a given query is planned. Accumulation is not learning.

### Why this matters

* No surprise behavior in production
* No silent drift
* No coupling between user queries and system state
* No hidden costs or latency spikes

Learning is a **field activity**.
Execution is a **production activity**.

They are intentionally separated.

---

## 5. Pluggability Over Monoliths

NAM is built as a **composable system**.

Encoder heads, the ontology, the embedding model, storage backends, and ingestion paths are all:

* explicit
* swappable
* independently evolvable

Encoder heads are selected per slot through a manifest that ingest and query share, and a head that keeps no state can be swapped on a running cluster. Access paths are pluggable too: the query orchestrator routes through a replaceable policy, and each path can be requested on its own.

NAM assumes:

* heterogeneous data
* heterogeneous teams
* heterogeneous deployment environments

There is no "one true model" or "blessed pipeline."

If a component cannot be replaced, it is considered a design failure.

---

## 6. Deployability Is a Core Requirement

If a system cannot be deployed predictably, it cannot be trusted.

NAM is designed to:

* run on commodity CPUs — GPUs are optional burst capacity for embedding, never a requirement
* deploy from a single templated chart, on a laptop cluster or a production cluster
* function without runtime LLM calls

The core address path needs no neural inference beyond its small, local codecs. The vector path's embedding model runs locally on CPU; a GPU tier, when configured, only adds throughput during backlogs.

This makes NAM viable in:

* regulated industries
* restricted or air-gapped environments
* cost-sensitive deployments
* long-lived systems with strict change control

---

## 7. Speed and Security Are Structural, Not Optional

NAM does not treat performance and security as tuning problems.

They are consequences of the design:

* deterministic addressing
* explicit partitioning
* key-based retrieval and prefix scans
* bounded query budgets on every path
* authentication at every external boundary

This enables:

* predictable latency
* bounded resource usage
* clear security boundaries
* auditable access paths

NAM favors **boring infrastructure** that behaves reliably under load.

---

## 8. Every Result Explains Its Provenance

A converged system is only trustworthy if you can take it apart.

* Every fused result lists the `sources` — address, vector, graph — that found it.
* Every response carries an `access_paths` trace: which paths ran, why, which were skipped and why, what each cost, and what each contributed.
* Degraded storage is reported, never hidden: a path that could not read is distinguished from a path that found nothing.

"Found nothing" and "could not look" are different answers, and NAM never collapses them.

---

## 9. Integration Over Replacement

NAM is designed to sit **between** systems, not replace them.

It complements:

* databases and data warehouses (NAM is never the system of record)
* search engines
* knowledge graphs
* LLM-based applications

NAM does not compete for ownership of:

* storage of record
* generation
* long-term authoritative memory

Its role is to **shape the questions** those systems are asked, and to say how each answer was found.

---

## 10. Constraints Create Capability

Many modern systems optimize for flexibility and convenience.

NAM optimizes for **clarity**.

That means:

* fewer degrees of freedom
* stronger invariants
* explicit contracts
* intentional limitations

These constraints are what make NAM:

* reliable
* inspectable
* composable
* scalable over time

---

## Closing Thought

NAM is not built to impress demos.

It is built to survive:

* production incidents
* organizational turnover
* evolving requirements
* long time horizons

These principles exist to ensure that the system remains understandable **long after its creators are no longer in the room**.

→ See also: [High-Level Systems](../ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md) | [Terminology](TERMINOLOGY.md) | [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)
