# What Is NAM?

Neural Addressable Memory (NAM) is a system for **finding information when you don't yet know exactly what you're looking for**.

Most data systems assume you already have a clear question.
NAM assumes you don't.

Instead of trying to guess the "best answer," NAM helps you **navigate meaning**, explore relationships, and narrow your understanding until the right question becomes obvious.

---

## A Simple Way to Think About It

Most modern AI retrieval works like this:

> "Here's your question.
> I'll search for things that look similar.
> I'll rank them.
> I'll give you the top result."

NAM's core works differently:

> "Here's your input.
> I'll figure out what *kind* of thing you're talking about and which things you named.
> I'll place it into a structured semantic space.
> I'll let you explore that space deliberately."

The difference is subtle — and fundamental.

NAM does not pretend similarity is useless. It also keeps a vector index, for the cases where wording differs but meaning matches, and a graph index, for walking from what you named to what is connected to it. But those sit beside the structured core, and every result tells you which of them found it.

---

## Why Similarity Alone Isn't Enough

Similarity-based systems (vectors, embeddings, ranking models), used on their own:

* Always return *something*, even when they shouldn't
* Can't explain *why* something matched beyond a score
* Collapse multiple meanings into a single number
* Behave differently over time as models change
* Are difficult to reason about or debug

They are optimized for **answers**, not **understanding**.

NAM is optimized for **sense-making** — and uses similarity as one clearly labelled signal among several, not as the foundation.

---

## What NAM Actually Stores

NAM does **not** store "knowledge" in the traditional sense, and it does not store your documents as the system of record.

It stores **ways to find them**.

Each record is translated into structured addresses based on:

* what it refers to (entities)
* what kind of thing it is (an ontology class from a standard upper ontology)
* what qualities it asserts (attributes)
* what actions it describes (affordances)
* under what circumstances it applies (context)

These addresses form a **map of meaning**, not a list of facts.

Alongside the addresses, NAM keeps an embedding of each record (for the vector path) and a graph of which records mention which entities (for the graph path). All three are derived from the same source records and can be rebuilt from them.

---

## Geometry, Not Ranking

Instead of asking:

> "Which result is most similar?"

NAM's address path asks:

> "Where does this live?"

Information can exist as:

* a **point** (very specific)
* a **line** (varies along one dimension)
* a **plane** (varies along two)
* a **volume** (broad or exploratory)

Queries do not "search" — they **project** into this space.

---

## One Query, Three Paths

A single query can run up to three ways at once:

* **Address** — structured, deterministic navigation of the address space
* **Vector** — nearest records by embedding, above a similarity floor
* **Graph** — outward from the entities the query names

The results are merged. Each result lists which paths found it, and each response includes a trace of what every path did and why it ran (or why it didn't). You can also ask for one path explicitly.

---

## Determinism Is the Point

NAM's address path is deterministic by design.

That means:

* the same input always produces the same addresses
* the same query probes the same regions
* retrieval behavior is stable and explainable
* results can be audited, replayed, and reasoned about

The vector path is approximate by nature, and NAM says so: its contribution to every result is visible, with its similarity score.

NAM favors **predictability over cleverness**.

---

## Ambiguity Is Not a Bug

Most systems try to eliminate ambiguity as quickly as possible.

NAM treats ambiguity as **useful information**.

If something could mean multiple things:

* NAM keeps those interpretations alive
* exploration happens before narrowing
* nothing is collapsed prematurely

This makes NAM especially useful for:

* early-stage investigation
* research
* discovery
* exploratory analytics
* AI systems that need grounding before generation

---

## What NAM Enables

NAM makes it possible to:

* ask incomplete or vague questions
* explore a domain before deciding what matters
* see which mechanism found each result
* build AI systems that ground themselves before generating
* combine symbolic structure with neural signals in one query
* support explainable, inspectable retrieval

---

## What NAM Is *For*

NAM is best suited for:

* exploratory AI systems
* analytical platforms
* investigation workflows
* decision support
* systems where correctness and explainability matter
* environments where data meaning evolves over time

It is **not** designed to replace:

* search engines
* chatbots
* transactional databases
* dashboards
* recommendation systems

NAM complements those systems — it doesn't compete with them directly.

---

## In One Sentence

**NAM is a converged semantic index — deterministic, geometry-based addressing at its core, with labelled vector and graph paths beside it — designed to help humans and machines understand information before trying to answer questions about it.**

→ See also: [Where NAM Fits](WHERE_NAM_FITS.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [Geometric Retrieval](../ARCHITECTURE/GEOMETRIC_RETRIEVAL.md) | [Query Model](../ARCHITECTURE/QUERY_MODEL.md)
