# PRFAQ Excerpts

This document contains selected excerpts from NAM’s internal PR/FAQ.
They reflect the *intentional framing* of the system — what it is, what it is not, and why it exists.

These excerpts are written to support:

* partners
* advisors
* early customers
* GTM collaborators

They avoid implementation detail while preserving architectural truth.

---

## Press Release (Excerpt)

**NAM Introduces Deterministic Semantic Addressing for Operational AI Systems**

Today we announce NAM (Neural Addressable Memory), a new system for organizing and retrieving meaning in production environments.

Unlike similarity-based retrieval systems, NAM uses deterministic semantic addressing to enable precise, explainable, and repeatable access to information — even when meaning is ambiguous or incomplete.

NAM is designed to operate alongside existing databases, search engines, and AI systems, providing a structured semantic layer that makes retrieval reliable at scale.

---

## What Problem Is NAM Solving?

Modern AI systems are very good at generating answers — and very bad at explaining why those answers were chosen.

Most retrieval systems:

* rely on probabilistic similarity
* collapse meaning into opaque scores
* behave differently across runs
* require constant tuning

This makes them difficult to:

* trust
* debug
* secure
* deploy in regulated or mission-critical environments

NAM exists to solve **semantic reliability**, not just recall.

---

## Why Not Just Use Vector Search or RAG?

Vector search and RAG are powerful tools — but they are optimized for *approximation*.

They work best when:

* answers are fuzzy
* recall matters more than precision
* occasional mistakes are acceptable

NAM is optimized for a different class of problems:

* systems that must explain *why* a record was retrieved
* workflows that depend on consistent behavior
* environments where ambiguity must be preserved, not hidden

NAM does not replace vector search or RAG.
It structures and constrains them.

---

## What Makes NAM Different?

NAM introduces three key ideas:

1. **Semantic addressing instead of ranking**
   Records are retrieved based on where they exist in semantic space, not how close they are to a query vector.

2. **Deterministic execution paths**
   Given the same inputs, NAM produces the same retrieval behavior every time.

3. **Explicit ambiguity management**
   NAM represents multiple interpretations in parallel rather than forcing early resolution.

Together, these enable systems to reason *with* meaning rather than guess *about* it.

---

## Is NAM a Knowledge Graph?

No.

Knowledge graphs encode facts and relationships.
NAM encodes **semantic affordances and navigable meaning**.

NAM does not require:

* complete schemas
* exhaustive entity resolution
* upfront ontology alignment

It can operate with partial, evolving, or even conflicting information — by design.

---

## Does NAM Learn Over Time?

Yes — but **not at runtime**.

NAM is trained offline.
Training produces static artifacts that are:

* versioned
* reviewed
* deployed intentionally

At runtime:

* nothing learns
* nothing adapts
* nothing drifts

This separation is intentional and foundational.

---

## How Does NAM Scale?

NAM scales through:

* partitioning
* deterministic addressing
* key-based retrieval
* bounded query expansion

It avoids:

* global scans
* re-ranking pipelines
* unbounded fan-out

This makes performance predictable and cost-controllable.

---

## Who Is NAM For?

NAM is built for teams that:

* operate production AI systems
* care about correctness and explainability
* need predictable behavior under load
* work in regulated or security-sensitive environments

It is not designed for:

* quick demos
* exploratory chatbots
* consumer-facing novelty applications

---

## What Does NAM Refuse to Do?

NAM intentionally does **not**:

* rank results by “best match”
* guess intent
* silently adapt behavior
* hide uncertainty behind scores
* depend on external AI services at runtime

These are trade-offs, not limitations.

---

## How Should People Think About NAM?

Think of NAM as:

> A semantic control plane for retrieval —
> not a model, not a database, and not a black box.

It shapes how systems ask questions —
and ensures the answers can be trusted.

---

## Closing Note

NAM is built for the long term.

Its design choices favor:

* stability over novelty
* clarity over cleverness
* contracts over heuristics

These excerpts exist to make those choices explicit.
