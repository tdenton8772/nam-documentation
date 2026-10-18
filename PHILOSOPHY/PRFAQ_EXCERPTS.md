# PRFAQ Excerpts

This document contains selected excerpts from NAM's internal PR/FAQ.
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

Today we announce NAM (Neural Addressable Memory), a converged semantic index for organizing and retrieving meaning in production environments.

At its core, NAM uses deterministic semantic addressing to enable precise, explainable, and repeatable access to information — even when meaning is ambiguous or incomplete. Beside that core, NAM runs a vector path and a graph path over the same records, behind the same query, and tells the caller which path found each result.

NAM is designed to operate alongside existing databases, search engines, and AI systems, providing a structured semantic layer that makes retrieval reliable at scale.

---

## What Problem Is NAM Solving?

Modern AI systems are very good at generating answers — and very bad at explaining why those answers were chosen.

Most retrieval systems:

* rely on probabilistic similarity as their only mechanism
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

NAM does include vector search — because some questions really are about paraphrase and topical neighbourhood. But in NAM it is one labelled path among three, held to a relevance floor, and fused with structured results rather than standing in for them.

---

## What Makes NAM Different?

NAM introduces four key ideas:

1. **Semantic addressing instead of ranking**
   On the core path, records are retrieved based on where they exist in semantic space, not how close they are to a query vector.

2. **Deterministic execution paths**
   Given the same inputs, the address path produces the same retrieval behavior every time.

3. **Explicit ambiguity management**
   NAM represents multiple interpretations in parallel rather than forcing early resolution.

4. **Converged, traceable retrieval**
   Address, vector, and graph paths run behind one query; every result names its sources and every response traces each path.

Together, these enable systems to reason *with* meaning rather than guess *about* it.

---

## Is NAM a Knowledge Graph?

No.

Knowledge graphs encode curated facts and relationships.
NAM encodes **semantic addresses and navigable meaning**.

NAM types records with a standard upper ontology (BFO and the Common Core Ontologies), and it maintains a graph index derived from its own addresses. But it does not require:

* complete schemas
* exhaustive entity resolution
* hand-curated relationships

It can operate with partial, evolving, or even conflicting information — by design.

---

## Is NAM a Vector Database?

No.

NAM has a vector index, but it is one access path of three, it is explicitly approximate, and it exists to complement the deterministic address path rather than replace it. NAM does not aim to be a general-purpose vector store.

---

## Does NAM Learn Over Time?

Yes — but **not at runtime**.

NAM's models are trained offline.
Training produces static artifacts that are:

* versioned
* validated when loaded
* deployed intentionally

At runtime:

* no model is trained
* queries do not change behavior
* navigation rules do not drift

Ingest accumulates vocabulary (new entities and surface forms), which enriches future resolution without changing how any address is computed. This separation is intentional and foundational.

---

## How Does NAM Scale?

NAM scales through:

* partitioning
* deterministic addressing
* key-based retrieval and prefix scans
* bounded query expansion on every path
* horizontally scaled ingest and query services, with ingest sized to its backlog

It avoids:

* scans proportional to the corpus
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

* hide which mechanism produced a result
* guess intent
* silently adapt behavior
* hide uncertainty behind unlabelled scores
* depend on hosted AI services at runtime (optional rented GPU capacity runs NAM's own embedding model; without it, embedding runs locally on CPU)

These are trade-offs, not limitations.

---

## How Should People Think About NAM?

Think of NAM as:

> A semantic control plane for retrieval —
> not a model, not a database, and not a black box.

It shapes how systems ask questions —
and ensures the answers can be traced.

---

## Closing Note

NAM is built for the long term.

Its design choices favor:

* stability over novelty
* clarity over cleverness
* contracts over heuristics

These excerpts exist to make those choices explicit.

→ See also: [What Is NAM](../OVERVIEW/WHAT_IS_NAM.md) | [What NAM Can Do](../CAPABILITIES/WHAT_NAM_CAN_DO.md) | [Design Principles](DESIGN_PRINCIPLES.md)
