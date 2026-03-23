# Near-Term Roadmap

**Time horizon:** ~3-6 months
**Scope:** Hardening NAM for real pilots and early production use

---

### Purpose of the Near-Term Roadmap

The near-term roadmap focuses on making NAM **deployable, testable, and trustworthy** in real environments.

This phase is not about adding fundamentally new capabilities.
It is about **turning what already exists into something people can reliably run, evaluate, and reason about**.

---

## 1. Deployability & Operational Readiness

**Status:** Substantially achieved. Packaged deployment, automated lifecycle management, readiness gating, and repeatable upgrade workflows are operational. Security hardening (authentication, network isolation, transport encryption, encryption at rest) is implemented and exercised.

**Remaining work includes:**

* **Production observability**

  * Metrics export to external monitoring systems
  * Alerting on lease failures, partition lag, and entity store drift
  * A web-based dashboard with system health, pipeline metrics, encoder head monitoring, and time-series graphs is operational; external alerting integrations are not yet connected
* **Configuration hardening**

  * Explicit separation of runtime config vs training artifacts
  * Versioned artifacts with compatibility guarantees
* **Deployment documentation**

  * Runbooks for common operational scenarios
  * Capacity planning guidance

**Outcome:**
Teams can deploy, monitor, and operate NAM without deep internal knowledge.

> See: [Current State -- Scaling](CURRENT_STATE.md#scaling--deployment)

---

## 2. Corpus Generation & Domain Seeding

**Goal:** Make it straightforward to adapt NAM to a specific domain without retraining core logic.

Near-term efforts focus on:

* **Corpus tooling**

  * Utilities for preparing domain-specific text corpora
  * Clear expectations for corpus size and composition
* **Offline artifact generation**

  * Repeatable processes for generating:

    * ontology calibrations
    * affordance / capability mappings
    * encoder runtime configurations
* **Domain isolation**

  * Ability to run multiple domain configurations side-by-side
  * Explicit artifact versioning per domain

**Outcome:**
NAM can be evaluated meaningfully on real customer data, not just synthetic or demo content, without changing the core system.

---

## 3. Test Coverage & Deterministic Validation

**Status:** Foundational coverage in place. Deterministic test harnesses cover the rule-based NLP pipeline, encoder head compatibility, end-to-end ingestion, and authentication. Continuing to expand.

**Remaining work includes:**

* **Scenario-driven integration tests**

  * Full ingestion -> addressing -> storage -> query round-trips
  * Explicit active vs inactive record expectations
  * Cross-replica consistency validation
* **Artifact validation tests**

  * Ensuring affordance, role, and ontology mappings are complete and internally consistent
* **Regression protection**

  * Preventing accidental changes to deterministic behavior
  * Locking execution order, probe behavior, and fallback semantics

**Outcome:**
NAM's behavior becomes *provable*, not anecdotal.
Changes either preserve determinism or fail loudly.

---

## 4. Query Behavior Refinement

**Status:** Core query execution is operational — progressive fan-out, inter-tier satisfaction, ontology hint reordering, covering index retrieval, LCA codebook neighborhood fan-out, and entity codebook with rejection classes are all implemented and validated against real corpora.

**Remaining work includes:**

* **Affordance coverage expansion**

  * Filling gaps identified during corpus-based testing
  * Improving consistency across domains
* **Entity deduplication**

  * Merging near-identical fuzzy matches (entity minion system)
  * Broader surface form coverage and cross-document coreference

**Outcome:**
Queries feel more robust while remaining explainable and deterministic.

---

## 5. Documentation & Evaluation Support

**Status:** Progressing. Public documentation now covers the conceptual model, security and trust model, current capabilities, and governance. Continuing to expand.

**Remaining work includes:**

* Evaluation guidance for pilots
* Example workflows for domain adaptation
* Shared vocabulary across engineering, product, and GTM stakeholders
* Operator-facing documentation (runbooks, capacity planning, troubleshooting)

**Outcome:**
NAM can be evaluated on its merits, not misunderstood or mispositioned.

---

## 6. Dimensional Graph Traversal

**Status:** Operational. The addressing pipeline writes graph index entries directly to a dedicated `graph` bucket alongside address writes. Graph walk and record walk APIs traverse the index at configurable hop depth with entity resolution. Validated at 500K records with 19.7M graph index entries and ~200ms query latency.

**Goal:** Enable graph-style queries over NAM's semantic address space — walking relationships across dimensions without vector math at query time.

NAM encodes multi-dimensional semantic relationships (entity, attribute, affordance, context) via LCA byte codes. The semantic query path probes these dimensions via prefix scans on the covering index. Graph traversal extends this by enabling **cross-dimensional navigation**: given an address or record, find other addresses that share semantic relationships across multiple axis combinations simultaneously.

### How It Works

NAM's LCA codebook contains geometric structure — semantically similar strings map to nearby codebook entries. Graph traversal exploits this structure by materializing relationships at write time, so query-time graph walks are pure key-value lookups with no floating-point computation.

* **Write-time index materialization** — the addressing pipeline writes graph index entries (axis subset keys: single-axis, dual-axis, triple-axis, entity-to-record, record-reverse) directly to the `graph` bucket alongside address writes to the `nam` bucket. Two async write paths per addressing worker handle this in parallel.

* **Prefix-scan traversal** — graph walk and record walk APIs traverse the index via prefix scans. Given a starting query or record, the system resolves its address dimensions, scans the graph index for entries sharing those dimensions, and follows links to related records. Each hop expands the frontier by one degree of semantic separation.

* **Entity resolution** — graph walk results include entity resolution, mapping hash-based entity identifiers back to human-readable names. Per-entity diversity limiting prevents any single entity from dominating results.

### Graph Walk API

Two traversal modes are available:

* **Graph walk** (`POST /v1/graph/walk`) — starts from a query, encodes it through the same pipeline as semantic queries, and walks the graph index at configurable hop depth
* **Record walk** (`POST /v1/graph/record-walk`) — starts from a specific record ID and discovers semantically related records through shared address dimensions

### Remaining Work

* **Projection codebooks** — trained projection codebooks for configured axis combinations, enabling precomputed similarity matrices as sealed artifacts for richer cross-dimensional edges
* **Inverted indexes** — per-dimension indexes mapping codebook entries to sets of matching addresses, enabling dimensional lookups via simple key-value reads

### Properties

* **Deterministic** — graph topology is derived from the address store, not computed dynamically
* **No vector math at query time** — traversal is key-value prefix scans and lookups
* **Configurable** — different deployments can materialize different graph topologies
* **Rebuildable** — the graph index is fully derived from the address store and can be reconstructed by re-processing source data through the pipeline

**Outcome:**
NAM's address space is navigable as a graph. Queries discover related addresses across multiple semantic dimensions simultaneously — something no single-axis prefix scan can express.

---

## 7. Multi-Tenant Considerations

**Goal:** Prepare the system for deployments where multiple tenants or domains share infrastructure.

This is early-stage work that includes:

* Tenant-scoped data isolation patterns
* Per-tenant configuration and artifact management
* Access control boundaries between tenants
* Audit logging scoped to tenant context

**Outcome:**
NAM can support multi-tenant pilots without architectural rework.

---

## What This Phase Is *Not*

The near-term roadmap does **not** include:

* Real-time learning or adaptive query behavior
* Ranking, scoring, or probabilistic relevance models
* Autonomous optimization or self-tuning
* Automated key rotation for encrypted storage

Those belong in later phases.

---

## Summary

The short-term roadmap is about **earning trust and extending reach**:

* Trust that NAM can be deployed securely
* Trust that it behaves deterministically
* Trust that access is authenticated and auditable
* Trust that it can be evaluated honestly
* Trust that domain adaptation is deliberate and controlled
* Graph traversal that unlocks cross-dimensional semantic navigation

Once those foundations are solid, expanding capability becomes much easier — and much safer.

> See also: [Current State](CURRENT_STATE.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Long-Term Direction](LONG_TERM.md)
