# NEAR_TERM.md

**Time horizon:** ~3–6 months
**Scope:** Hardening NAM for real pilots and early production use

---

### Purpose of the Near-Term Roadmap

The near-term roadmap focuses on making NAM **deployable, testable, and trustworthy** in real environments.

This phase is not about adding fundamentally new capabilities.
It is about **turning what already exists into something people can reliably run, evaluate, and reason about**.

---

## 1. Deployability & Operational Readiness

**Status:** Partially achieved. Container orchestration (Kubernetes, Docker Compose), repeatable reset workflows, and sidecar architecture are operational. Continuing to harden.

**Remaining work includes:**

* **Production observability**

  * Structured logging, metrics export, health endpoints
  * Alerting on lease failures, partition lag, and entity store drift
* **Configuration hardening**

  * Explicit separation of runtime config vs training artifacts
  * Versioned artifacts with compatibility guarantees
* **Deployment documentation**

  * Runbooks for common operational scenarios
  * Capacity planning guidance

**Outcome:**
Teams can deploy, monitor, and operate NAM without deep internal knowledge.

→ See: [Current State — Scaling](CURRENT_STATE.md#scaling--deployment)

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

**Status:** Foundational coverage in place. Deterministic test harnesses cover the rule-based NLP pipeline, encoder head compatibility, and end-to-end ingestion. Continuing to expand.

**Remaining work includes:**

* **Scenario-driven integration tests**

  * Full ingestion → addressing → storage → query round-trips
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

**Goal:** Improve reliability and usefulness without changing the core query model.

Near-term work includes:

* **Better exploratory probing**

  * Smarter widening when exact geometric matches are absent
  * Clear rules for fallback behavior
* **Affordance coverage expansion**

  * Filling gaps identified during corpus-based testing
  * Improving consistency across domains
* **Entity handling improvements**

  * Broader surface form coverage and cross-document coreference
  * Reduced noise from weak or spurious entity matches
  * Type-scoped resolution is operational; depth and coverage continue to improve

**Outcome:**
Queries feel more robust while remaining explainable and deterministic.

---

## 5. Documentation & Evaluation Support

**Goal:** Enable informed evaluation without access to private implementation details.

This includes:

* Public-facing documentation updates
* Clear evaluation guidance for pilots
* Honest articulation of tradeoffs and limitations
* Shared vocabulary across engineering, product, and GTM stakeholders

**Outcome:**
NAM can be evaluated on its merits, not misunderstood or mispositioned.

---

## What This Phase Is *Not*

The near-term roadmap does **not** include:

* Real-time learning or adaptive query behavior
* Ranking, scoring, or probabilistic relevance models
* Autonomous optimization or self-tuning
* Broad, multi-tenant production hardening

Those belong in later phases.

---

## Summary

The short-term roadmap is about **earning trust**:

* Trust that NAM can be deployed
* Trust that it behaves deterministically
* Trust that it can be evaluated honestly
* Trust that domain adaptation is deliberate and controlled

Once those foundations are solid, expanding capability becomes much easier — and much safer.

→ See also: [Current State](CURRENT_STATE.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Long-Term Direction](LONG_TERM.md)

