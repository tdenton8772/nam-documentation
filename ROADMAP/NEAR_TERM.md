# Near-Term Roadmap

**Scope:** Hardening NAM for real pilots and early production use

---

## Purpose of the Near-Term Roadmap

The near-term roadmap is about making what exists **more accurate, more operable and easier to evaluate**. It is mostly not about new capabilities.

Nothing here is a dated commitment. Items are listed because they are open work today.

> See: [Current State](CURRENT_STATE.md) for what is already shipped.

---

## 1. Address Quality

The address path is only as precise as its type segment and axes.

* **Lower ontology abstention.** Reduce how often records fall back to `BFO:0000001`, through codebook training and better sentence evidence, and keep the abstention rate as a tracked quality signal
* **Tune downstream heads to BFO/CCO classes.** Attribute and affordance relevance, exemptions and context defaults are keyed by class; their tuning against the full class set is open work
* **Fuller axes.** Raise attribute, affordance and context coverage, and keep any single value from dominating an axis, guided by the anti-collapse axis metrics
* **Entity types that match the ontology.** Align the entity store's type vocabulary with the BFO/CCO classes the address key uses, so query-side hints derived from stored types are reliable

---

## 2. Query Behaviour

* **Smarter exploratory widening** — informed by which dimensions actually hold data, not only by dropping axes mechanically
* **Affordance coverage** — close gaps where valid question shapes fall back to exploratory mode
* **Entity coverage** — broader surface-form matching and merging of near-duplicate entities
* **Routing from evidence** — the routing policy records a reason and a cost for every decision, path by path. Those traces are the input to a cost-based routing policy that would replace the current heuristic one

---

## 3. Index Completeness and Freshness

* **Graph freshness** — keep the sharded graph builder close behind the address store, and make its lag visible as a first-class metric
* **Vector index maintenance** — replace or remove a record's vector entry when its source record is updated or deleted, and keep the CPU and GPU embedding tiers verifiably interchangeable

---

## 4. Throughput

Ingest is bound by compute-node CPU, and the ingest tier already sizes itself to its backlog. Near-term work is reducing per-record CPU in the heads and NLP.

---

## 5. Storage Operations

* **Accurate per-node storage statistics** in the admin API and dashboard

---

## 6. Observability

* Alert notification routing (email, chat, paging) for the shipped rule set
* Structured logging with trace identifiers across service boundaries
* Refined dashboard views for metric history, alerts and log search

---

## 7. Determinism and Test Coverage

* Property-based tests that the same input produces the same addresses across runs and replicas
* Failure-path tests: storage write failures, partial batches, change-stream interruptions
* Scenario tests that run ingest through query on all three paths and assert per-path expectations from the `access_paths` trace

---

## 8. Tenancy

Tenant and partition are already segments of every address, and the vector index is kept per tenant. Open work:

* tenant scopes in access tokens, enforced at the query API and KV proxy
* tenant and session metadata on every source document
* per-tenant configuration and tenant-scoped audit logging

---

## 9. Corpus Tooling and Domain Seeding

* Utilities to normalize, chunk and validate domain text into the document schema
* Repeatable offline generation of domain artifacts: ontology codebook, head vocabularies, capability maps
* Running more than one domain configuration side by side

---

## 10. Documentation and Evaluation Support

* Evaluation guidance for pilots, with per-path measurement
* Operator documentation: runbooks, capacity planning, troubleshooting
* Worked examples of domain adaptation

---

## What This Phase Is *Not*

The near-term roadmap does **not** include:

* runtime learning or self-modifying query behaviour
* opaque, unattributed ranking
* automated key rotation for encrypted storage

> See also: [Current State](CURRENT_STATE.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Long-Term Direction](LONG_TERM.md)
