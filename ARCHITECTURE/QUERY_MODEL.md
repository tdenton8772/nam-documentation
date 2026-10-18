# The NAM Query Model

*How a question becomes a set of access-path executions*

This document explains what happens between a query arriving at `POST /v1/query` and results coming back.

NAM is a converged index. A query is planned across up to three access paths built from the same source records, the paths run in parallel, and their results are fused into one list. Every result row says which path found it, and every response carries a trace of what each path did.

---

## The Three Access Paths

| Path | What it searches | Nature |
|---|---|---|
| **address** | The semantic address index (`nam` bucket) by key lookups and prefix matches | Deterministic, structural |
| **vector** | The vector index (`vector` bucket): top-K above a cosine floor | Explicitly approximate |
| **graph** | The graph index (`graph` bucket), walked outward from resolved entities | Deterministic, relational |

The address path is the core and is always run when routing is automatic. The other two are described in [Vector Search](VECTOR_SEARCH.md) and [Graph Index](GRAPH_INDEX.md).

---

## Orchestration

### Routing

A query is routed in one of two ways:

* **Pinned** — the caller passes `paths` (for example `["vector"]` or `["address", "graph"]`). Exactly those paths run.
* **Automatic** — the routing policy decides from cheap signals computed before anything executes: the query's intent and the entities resolved from it.

The routing policy is:

| Signals | Paths run |
|---|---|
| No entity resolved | address + vector |
| At least one entity resolved | address + vector + graph |

The graph path needs a resolved entity to start from, so it runs only when one exists. The vector path stays on even when an entity resolves, because the address path alone cannot reach every relevant record.

Routing is a pluggable policy. Every decision records its reason in the trace.

### Parallel execution

The chosen paths run concurrently. A path that fails becomes an `error` entry in the trace; the other paths still answer.

### Thin-result fallback

In automatic mode, if the vector path did not run and the address path returned fewer results than a configured fraction of the requested limit (never fewer than a small minimum, never more than the limit), the vector path runs as a fallback. Its trace entry is marked `fallback` and its reason states the counts. The current routing policy always runs the vector path in automatic mode, so the fallback applies to a policy that leaves it out.

### Fusion

Results are fused by data pointer (the source record's identity):

1. Address results first, in the order the address path produced them.
2. Vector hits next. A hit for a record the address path already returned adds `vector` to that row's `sources` and attaches its `cosine`; otherwise it becomes a new row.
3. Graph results last, the same way.
4. Rows whose record key names one of the resolved entities move to the front, whichever path found them. A query about a person puts that person's own record first.

Each fused row carries:

* `data_pointer` — where the source record lives
* `sources` — the paths that found it, for example `["address", "vector"]`
* `cosine` — present exactly when the vector path contributed to the row
* `address_meta` — the matched address details when the address path found it. For a row only the vector or graph path found, it is the address the record is stored under when a bounded lookup finds one, marked as stored rather than matched; otherwise it is empty
* `payload` — the source record, attached after fusion

### The `access_paths` trace

Every response lists one entry per path:

| Field | Meaning |
|---|---|
| `path` | `address`, `vector` or `graph` |
| `status` | `used`, `fallback`, `error`, `skipped` (automatic routing chose not to run it) or `not_requested` (the caller pinned other paths) |
| `reason` | Why the path ran or did not |
| `ms` | Time spent |
| `stats` | Path-specific counters: geometry level and probes for address; the work done and the top cosine for vector; hops, edges and seed entities for graph |
| `results_contributed` | Rows this path added that no earlier path had |

Entries may also carry `unavailable` (the path hit a data-service error) and `read_failures` (storage reads that failed underneath and would otherwise look like "no data").

### Degraded storage is reported, never hidden

* If some reads failed but results exist, the response is `200` with `partial: true`, `failed_sources`, and a `degraded` list saying which source failed and why.
* If reads failed and nothing came back, or the query exceeded its time budget, the response is `503` with a `Retry-After` header.

A caller can always tell "found nothing" from "could not look."

---

## The Address Path in Detail

### 1. Canonicalize

The query text is normalized by a rule-based canonicalizer:

* Articles are stripped.
* **Intent** is detected from the interrogative (`who` → identity, `where` → location, `when` → temporal, `what is` → definition, `how` → process, `why` → causal, `which` → selection).
* Interrogatives and leading copulas/auxiliaries are removed; intent already carries their signal.

### 2. Resolve entities

Entities are resolved against the **session store** — the same store ingest writes surface forms into — using the **same resolver as the ingest entity head**:

* Candidates come from the entity head's own query-mode extraction, plus contiguous multi-word spans of the original query looked up as exact stored names.
* A mention covered by a longer resolved mention is dropped (the longest specific name wins).
* Bare numerals, roman numerals, very short non-acronym words, and spans that begin or end with a function or question word are dropped.
* Once any specific name resolves, generic common nouns are dropped.
* Results are ordered longest first, then by confidence.

Resolved entities feed routing (above), the graph path's seeds, and fusion's entity-first ordering.

### 3. Encode and bundle

The same encoder heads used at ingest run in query mode on the canonical text. The entity-anchored bundler then links attributes, affordances and contexts to the query's entities, exactly as at ingest.

### 4. Determine the mode

* **Affordance mode** — the question's shape matches a supported affordance. The affordance selects the ontology tiers to probe.
* **Exploratory mode** — otherwise. All exploratory tiers are probed, with tiers containing hinted classes first.

### 5. Plan

The planner produces an ordered, budget-capped list of scans, grouped into tiers:

1. **STORED** — for each query entity, the graph index records which BFO/CCO classes its addresses are actually stored under. Those partitions are probed first, round-robin across (entity, class) pairs so that every partition's first scan comes before any partition's second. Every entity is also probed under **BFO:0000001** (unclassified), whether or not the index lists it: list pages, ambiguous records and anything indexed after the last graph build land there. For entities with no stored classes, these safety-net probes are capped to a fraction of the budget.
2. **Ontology tiers** — remaining entities are explored across tiers, one per BFO branch (independent continuants, occurrents, specifically dependent continuants, generically dependent continuants). Within a tier, classes are ordered: hinted classes first, then classes related to a hint by subsumption (ancestor or descendant), then the rest, shallower before deeper. The budget is spread across classes and rotated across entities so a small per-class budget still reaches every entity.
3. **FALLBACK** — if exploration planned nothing, the unexpanded scans run as-is.

### 6. Fan out across the codebook

Each specified axis's coarse code is expanded to its nearest codebook neighbours. The leading axis's codes become scan prefixes; the other specified axes are post-filtered against their acceptable codes. See [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md).

### 7. Execute with progressive widening

The runner executes tiers in order:

* Probes run until enough **distinct** records are found or the fetch budget is spent. On the reference deployment each tier is read in a fixed number of batched round trips; in the rotations layout each prefix scan pages with a cursor until enough distinct records are found or the prefix is exhausted.
* In the STORED tier, each partition's first fetch takes a fair share of the limit, so one large partition cannot crowd out the others.
* After a tier has produced a satisfying number of matches, later tiers are skipped.
* The whole plan repeats at broader geometry levels (point → line → plane → cube) until the limit is reached or the request's `fan_out` cap is hit. Records are deduplicated across levels.

### 8. Return

Matches are deduplicated by record, payloads are attached, and each row carries `address_meta` (the matched key, its type and entity, the per-axis coarse and fine codes, match score and matched geometry).

---

## The Graph Path in Detail

The graph path seeds from the resolved entities — up to three, strongest first. One walk runs per seed, in parallel, with the per-walk limit split across seeds so total work stays that of one walk. Results are merged so that records reached from more seeds come first: for a query naming three people, records connected to all three lead, then the union. A seed whose walk fails is skipped. See [Graph Index](GRAPH_INDEX.md).

---

## The Vector Path in Brief

The query is embedded with the same model used at ingest, the part of the vector index nearest it is read, and the candidates are ranked by cosine similarity. Only records at or above the deployment's cosine floor are returned, top `limit` by cosine. See [Vector Search](VECTOR_SEARCH.md).

---

## Ordering and Scores

* The address path orders by scan order and match score; it does not compute embedding similarity.
* The vector path's only score is `cosine`.
* Fusion keeps address order first, then vector-only rows, then graph-only rows, with entity-named records lifted to the front.

NAM returns candidates with their provenance. Answer generation, summarization and reasoning are downstream concerns.

---

## Why a Query Can Return Nothing From a Path

* **address** — no records were written at the planned regions, the query was more specific than the data supports, or the scope (tenant, partition) differs from what ingest wrote.
* **vector** — nothing in the part of the index the query read cleared the cosine floor. (A failed read is an `error`, not an empty result.)
* **graph** — no entity resolved, or the resolved entity has no edges yet (the graph index lags the address index).

Each case is visible in `access_paths`.

---

## Summary

* `POST /v1/query` routes across address, vector and graph paths, by policy or as pinned by the caller.
* Paths run in parallel; results are fused by record with address priority and entity-first ordering.
* Every row lists its `sources`; every response carries an `access_paths` trace; degraded storage is reported.
* The address path canonicalizes, resolves entities from the session store, plans STORED-then-ontology tiers, fans out over the codebook, and widens geometrically until enough distinct records are found.

→ See also: [Geometric Retrieval](GEOMETRIC_RETRIEVAL.md) | [Addressing Model](ADDRESSING_MODEL.md) | [Vector Search](VECTOR_SEARCH.md) | [Graph Index](GRAPH_INDEX.md) | [API Reference](API_REFERENCE.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md)
