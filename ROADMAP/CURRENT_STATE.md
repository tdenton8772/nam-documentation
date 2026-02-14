# Current State

This document describes the **current state of the NAM project today** — what exists, what works, and what is actively being exercised in real systems.

It is intentionally factual and implementation-agnostic.
This is not a roadmap, pitch, or vision statement.

→ See also: [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Near-Term Roadmap](NEAR_TERM.md)

---

## Project Maturity

NAM is in an **active production-pilot phase** with proven horizontal scaling.

This means:

* Core architecture is implemented and exercised end-to-end against real corpora
* The system scales horizontally across multiple ingest, encoding, and storage replicas
* Behavior is deterministic, observable, and reproducible
* APIs are stable for pilots and internal deployments
* Container-orchestrated deployment is operational

The system is past prototype, past "does it work," and into "how well does it scale."

---

## Core System Capabilities (Today)

### Deterministic Ingestion

* Records are ingested through a defined pipeline with deterministic linguistic parsing
* The NLP pipeline is fully rule-based — no model weights, no adaptive behavior
* Encoder heads emit structured semantic signals
* All derived state is reproducible from inputs
* No hidden learning or runtime mutation occurs

→ See: [Ingestion Model](../ARCHITECTURE/INGESTION_MODEL.md)

### Entity-Anchored Address Construction

* Address construction is scoped to the entities each semantic component actually modifies
* Dependency parsing determines which components belong to which entities
* This preserves linguistic structure and prevents spurious address generation
* Address count per record is bounded by the structure of the text, not combinatorial explosion

→ See: [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md)

### Type-Scoped Entity Resolution

* Every entity has a type derived from NER analysis during ingest
* Entity resolution is scoped by type — cross-type collisions are structurally prevented
* Surface forms are matched to canonical entries with approximate matching and structural guards
* The entity store is distributed across the cluster with deterministic resolution order
* All ingest and query replicas share the same entity namespace

### Semantic Addressing

* Records are mapped to multi-axis semantic addresses
* Address construction is deterministic and inspectable
* Address structure reflects semantic geometry, not similarity
* Storage keys are the address space

→ See: [Geometric Retrieval](../ARCHITECTURE/GEOMETRIC_RETREIVAL.md)

### Geometric Retrieval

* Queries generate bounded sets of probes into semantic space
* Retrieval operates over points, lines, planes, and volumes
* Progressive fan-out broadens geometrically — no scanning
* Exploratory and targeted queries are supported
* Results are explainable by address overlap

### Ingest-Time Learning

* New entities are registered and surface forms mapped to canonical entries during ingest
* Unknown vocabulary is auto-classified based on context
* Curated seed vocabulary takes priority over discovered terms
* Query time remains strictly read-only

→ See: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md)

### Offline Training & Artifacts

* Encoder heads are calibrated offline from labeled corpora
* Calibration produces frozen, deterministic artifacts — not trained neural weights
* Artifacts are versioned and promoted explicitly
* Runtime never retrains or mutates models

### Affordance-Aware Querying

* Queries are analyzed for affordances
* Affordances select execution strategies
* Unsupported affordances safely fall back to exploration
* Behavior is consistent across runs

→ See: [Query Model](../ARCHITECTURE/QUERY_MODEL.md)

---

## Implemented Subsystems

Today's NAM implementation includes:

* Pluggable encoder head framework (one head per semantic axis)
* Entity-anchored address construction via dependency-parse-scoped bundling
* Type-scoped entity resolution with distributed shared store
* Rule-based NLP pipeline (deterministic tokenization, POS tagging, NER, dependency parsing)
* Addressing and partitioning logic
* Storage abstraction layer with distributed KV backend
* Query planner and runner with progressive geometric fan-out
* Public query API
* CDC-based ingestion from change streams
* CAS-based lease coordination (no external dependencies beyond the data service)
* Offline calibration pipelines
* Deterministic test harnesses

These systems are not stubs — they are exercised regularly at scale.

---

## Scaling & Deployment

NAM has been validated under horizontal scaling:

* Multiple ingest replicas coordinate via CAS-based lease claims
* Encoding, storage, and NLP run as independent, scalable services
* Sidecar architecture collocates complementary services within pods
* Pipeline stages overlap for throughput
* The system has exactly one external dependency (the data service) for storage, coordination, and streaming

Deployment today includes:

* Container-orchestrated deployments (Kubernetes manifests, Docker Compose)
* Repeatable reset and reload workflows
* Explicit artifact management
* Observability of addresses, probes, and execution paths

→ See: [Design Principles — Deployability](../PHILOSOPHY/DESIGN_PRINCIPLES.md#6-deployability-is-a-core-requirement)

---

## Observability & Debuggability

NAM emphasizes transparency:

* Addresses can be logged and inspected
* Query probes are visible
* Storage lookups are traceable
* Execution paths are explainable
* When retrieval fails, you can identify which axis failed to resolve

This makes NAM suitable for:

* Technical evaluation
* Architectural review
* Regulated or audit-sensitive environments

→ See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Documentation State

Public documentation currently focuses on:

* Explaining the conceptual model
* Defining terminology and invariants
* Setting expectations clearly
* Enabling informed evaluation

Implementation details, internal tuning, and private extensions live elsewhere.

→ See: [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [FAQ](../APPENDIX/FAQ.md)

---

## Summary

Today, NAM is:

* Architecturally coherent
* Horizontally scalable
* Deterministic by design
* Proven under real workloads
* Capable of meaningful pilots and internal deployments
* Intentionally conservative in behavior

It is early — but it is **solid and scaling**.
