# Glossary

This glossary is a **navigation aid**, not an authoritative definitions document.

Canonical definitions for NAM concepts live in
**PHILOSOPHY/TERMINOLOGY.md**.
This file exists to help readers quickly orient themselves and find the
correct place to learn more.

---

## Address

A structured semantic location used by NAM to store and retrieve knowledge.
Addresses are deterministic, composable, and geometric.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Address**

---

## Address Family

A related set of addresses derived from a single record or concept.
Address families allow NAM to answer more questions over time without
re-encoding prior data.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Address Families**

---

## Affordance

An inferred capability of a query that determines *how* NAM should interpret
and explore knowledge (e.g., identity lookup, explanation, comparison).

Affordances guide retrieval strategy but do not score or rank results.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Affordance**

---

## Axis (plural: Axes)

A semantic dimension of an address (e.g., X, Y, Z) used to position records
within NAM’s geometric space.

Axes are populated deterministically by encoder heads.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Axes**

---

## Determinism

The principle that identical inputs must always produce identical outputs
(addresses, probes, retrieval paths).

Determinism is a first-order requirement for trust, auditability, and deployment.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Determinism**

---

## Encoder Head

A pluggable component responsible for extracting a specific semantic signal
(entity, ontology, affordance, context) from text.

Encoder heads may be trained offline but behave deterministically at runtime.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Encoder Heads**

---

## Geometry

The conceptual model NAM uses for retrieval: points, lines, planes, and volumes
within a semantic space.

Retrieval is geometric navigation, not similarity scoring.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Geometry**

---

## Ontology

A controlled semantic category used to group and reason about concepts
(e.g., person, location, process).

Ontologies define structure, not meaning.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Ontology**

---

## Probe

The geometric representation of a query generated at query time.
A probe defines *where* and *how* NAM should explore its stored space.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Probe**

---

## Record

An ingested unit of information (text plus metadata) that is encoded into one
or more addresses.

Records are immutable once ingested.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Record**

---

## Wildcard

A deliberate absence in an address axis used to widen retrieval.
Wildcards are first-class geometric constructs, not errors or fallbacks.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Wildcards**

---

## Entity Bundle

A group of semantic components (attributes, affordances, contexts) linked to a
specific entity via dependency parsing. Bundles prevent combinatorial address
explosion by only generating addresses for linguistically grounded relationships.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Entity Bundle**

---

## Exploratory Mode

A retrieval mode used when no specific affordance is detected.
Exploratory retrieval favors coverage and semantic breadth over precision.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Exploratory Retrieval**

---

## Affordance Mode

A retrieval mode activated when an affordance verb is recognized in the query.
Narrows probing to specific ontology tiers based on the detected affordance.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Affordance Mode**

---

## Progressive Fan-Out

The query execution strategy that probes addresses in order of decreasing specificity,
with early termination when a satisfaction threshold is reached. Deterministic and
budget-bounded.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Progressive Fan-Out**

---

## Lease

A time-bounded ownership claim on a data partition, stored as a key-value document
and managed via compare-and-swap operations. Enables distributed partition
coordination without external dependencies.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Lease**

---

## LCA (Learned Codec for Addressing)

A character-level neural encoder that maps string coordinate values to compact
byte-code pairs (`lca:coarse:fine`). Provides geometric structure — semantically
similar strings map to nearby codes — enabling neighborhood-based query fan-out.

> See: **ARCHITECTURE/ADDRESSING_MODEL.md — Coordinate Encoding (LCA)**

---

## Knowledge Accumulation

The process by which NAM becomes more capable over time by accumulating
address families—not by changing encoder behavior.

→ See: **PHILOSOPHY/TERMINOLOGY.md — Learning vs Accumulation**

---

### Notes

* This glossary intentionally avoids implementation detail.
* If a term appears ambiguous or overloaded, defer to **TERMINOLOGY.md**.
* New terms should be added here *only after* being defined canonically.

→ See also: [Terminology](../PHILOSOPHY/TERMINOLOGY.md) (canonical definitions)
