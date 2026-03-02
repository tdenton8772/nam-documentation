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
  * Dashboards for ingest throughput, query latency, and cluster health
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

**Status:** Significant progress achieved. Progressive fan-out, ontology hint reordering, and inter-tier satisfaction stopping are implemented and validated.

**Completed:**

* **Progressive fan-out** — queries now probe addresses in specificity order with budget-bounded execution
* **Inter-tier satisfaction** — probing stops when enough results are found, preventing unbounded exploration
* **Ontology hint reordering** — tiers containing types observed by the bundler are probed first
* **Hash-based entity resolution** — type-independent entity IDs prevent ingest/query alignment failures
* **Fuzzy-first entity matching** — candidate key lookups as primary resolution path
* **Ontology hardening** — eliminated "OTHER" catch-all; 26 canonical types with "concept" as broadest fallback
* **LCA neighborhood fan-out** — codebook-aware k-nearest-neighbor expansion at query time. Each axis's coarse code is expanded to include k nearest codebook neighbors, probing semantically adjacent address regions. Bridges between exact-match retrieval and the long-term latent walk vision.

**Remaining work includes:**

* **Affordance coverage expansion**

  * Filling gaps identified during corpus-based testing
  * Improving consistency across domains
* **Entity deduplication**

  * Merging near-identical fuzzy matches (entity minion system)
  * Broader surface form coverage and cross-document coreference
* **LCA ingest-side fan-out**

  * Write neighborhood addresses at ingest time (not just query time) for higher recall
  * Extend LCA axes (temporal, geographic) with model retraining

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

## 6. Multi-Tenant Considerations

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

The short-term roadmap is about **earning trust**:

* Trust that NAM can be deployed securely
* Trust that it behaves deterministically
* Trust that access is authenticated and auditable
* Trust that it can be evaluated honestly
* Trust that domain adaptation is deliberate and controlled

Once those foundations are solid, expanding capability becomes much easier — and much safer.

> See also: [Current State](CURRENT_STATE.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Long-Term Direction](LONG_TERM.md)
