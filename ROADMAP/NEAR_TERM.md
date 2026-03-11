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

**Goal:** Enable graph-style queries over NAM's semantic address space — walking relationships across dimensions without vector math at query time.

NAM already encodes multi-dimensional semantic relationships (entity, attribute, affordance, context) via LCA byte codes. Today, these dimensions are queried independently via prefix scans on the covering index. Graph traversal extends this by enabling **cross-dimensional navigation**: given an address, find other addresses that share semantic relationships across multiple axis combinations simultaneously.

### How It Works

The core idea: NAM's LCA codebook already contains geometric structure — semantically similar strings map to nearby codebook entries. Graph traversal exploits this structure by materializing relationships at write time, so query-time graph walks are pure key-value lookups with no floating-point computation.

* **Dimensional projections** — instead of per-dimension graphs, NAM builds projections across dimension *combinations* (e.g., entity+affordance, entity+context, attribute+affordance). Each projection captures composite semantic relationships that no single dimension expresses alone.

* **Write-time edge materialization** — LCA codes are codebook indices, not vectors. Proximity only exists in the original embedding space. At write time, the system resolves edges using a precomputed similarity matrix (a sealed artifact trained from codebook vectors). The result: precomputed neighbor lists stored alongside inverted indexes.

* **Query-time graph walks** — all lookups, no computation. Read an address's projection code, read that code's precomputed edge list, look up the inverted index for each neighbor code. The result is a set of semantically related addresses found via integer lookups and set operations.

### Architectural Approach

The graph index is a **separate derived store maintained via change data capture**, following the same DCP-driven pattern used throughout NAM:

* The address store stays simple — standard key-value storage, no custom formats
* The index is derived, rebuildable, and eventually consistent with the address store
* Schema changes or new projections are handled by replaying the stream
* Index writes do not affect address store read/write performance

### Phased Work

1. **Per-dimension storage statistics** — dimension-level metadata (min, max, cardinality) computed during storage flushes. Enables file-level pruning before any data is read.

2. **Inverted indexes** — per-dimension indexes mapping codebook entries to sets of matching addresses. Enables dimensional lookups via simple key-value reads.

3. **Projection codebooks, edges, and graph walks** — trained projection codebooks for configured axis combinations, precomputed similarity matrices as sealed artifacts, and a graph walk API that chains key lookups through edge lists and inverted indexes.

### Properties

* **Deterministic** — graph topology is derived from sealed artifacts, not computed at query time
* **No vector math at query time** — edges are precomputed integers, traversal is key-value lookups
* **Configurable** — graph schemas define which axis combinations to project. Different deployments can materialize different graph topologies.
* **Analytical, not real-time** — subsecond response is the target, not single-digit milliseconds
* **Rebuildable** — the entire graph index can be reconstructed by replaying the address store's change stream

**Outcome:**
NAM's address space becomes navigable as a graph. Queries can discover related addresses across multiple semantic dimensions simultaneously — something no single-axis prefix scan can express.

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
