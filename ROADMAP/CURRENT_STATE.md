# Current State

This document describes the **current state of the NAM project today** — what exists, what works, and what is actively being exercised in real systems.

It is intentionally factual and implementation-agnostic.
This is not a roadmap, pitch, or vision statement.

> See also: [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md) | [Near-Term Roadmap](NEAR_TERM.md)

---

## Project Maturity

NAM is in an **active production-pilot phase** with proven horizontal scaling and operational security.

This means:

* Core architecture is implemented and exercised end-to-end against real corpora
* The system scales horizontally across multiple ingest, encoding, and storage replicas
* Authentication, network isolation, and transport security are operational
* Cluster lifecycle is automated from bootstrap through health monitoring
* Behavior is deterministic, observable, and reproducible
* APIs are stable for pilots and internal deployments
* Deployment is packaged and repeatable

The system is past prototype, past "does it work," and into "how well does it operate in production."

---

## Core System Capabilities (Today)

### Deterministic Ingestion

* Records are ingested through a defined pipeline with deterministic linguistic parsing
* The NLP pipeline is fully rule-based — no model weights, no adaptive behavior
* Encoder heads emit structured semantic signals
* All derived state is reproducible from inputs
* No hidden learning or runtime mutation occurs

> See: [Ingestion Model](../ARCHITECTURE/INGESTION_MODEL.md)

### Entity-Anchored Address Construction

* Address construction is scoped to the entities each semantic component actually modifies
* Dependency parsing determines which components belong to which entities
* This preserves linguistic structure and prevents spurious address generation
* Address count per record is bounded by the structure of the text, not combinatorial explosion

> See: [Addressing Model](../ARCHITECTURE/ADDRESSING_MODEL.md)

### Hash-Based Entity Resolution

* Every entity is identified by a **deterministic hash** of its canonical form — type-independent and stable
* The same entity always receives the same identifier, regardless of ontology classification (person, object, concept, etc.)
* Resolution uses a **fuzzy-first strategy**: candidate key lookups (approximate matching) before exact surface form matching
* The entity store is distributed across the cluster with deterministic resolution order
* All ingest and query replicas share the same entity namespace
* Node-level caching reduces resolution latency to microseconds for warm lookups
* The entity store persists across pipeline resets, acting as a warm cache that accumulates knowledge over time

### Semantic Addressing

* Records are mapped to multi-axis semantic addresses
* Address construction is deterministic and inspectable
* Address structure reflects semantic geometry, not similarity
* Storage keys are the address space

> See: [Geometric Retrieval](../ARCHITECTURE/GEOMETRIC_RETRIEVAL.md)

### Geometric Retrieval with Progressive Fan-Out

* Queries generate bounded sets of probes into semantic space
* Retrieval operates over points, lines, planes, and volumes
* **Progressive fan-out** probes addresses in specificity order within each ontology tier
* **Inter-tier satisfaction** stops probing when enough results are found, preventing unbounded exploration
* **Ontology hint reordering** prioritizes tiers containing types the bundler observed in the query
* Two explicit query modes: **exploratory** (no affordance detected) and **affordance** (specific intent recognized)
* Budget limits cap total probes and document fetches
* Exploratory and targeted queries are supported
* Results are explainable by address overlap, tier probed, and specificity level

### Ingest-Time Learning

* New entities are registered and surface forms mapped to canonical entries during ingest
* Unknown vocabulary is auto-classified based on context
* Curated seed vocabulary takes priority over discovered terms
* Query time remains strictly read-only

> See: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md)

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

> See: [Query Model](../ARCHITECTURE/QUERY_MODEL.md)

---

### Ontology Classification

* 26 canonical ontology types organized into 4 tiers
* No catch-all "OTHER" category — every record maps to a real semantic type
* "concept" serves as the broadest valid category
* Ontology hints from the bundler inform query-time tier prioritization

---

## Implemented Subsystems

Today's NAM implementation includes:

* **Staged ingest pipeline** — CDC streaming, rule-based NLP, ontology classification, parallel encoder heads, entity-anchored addressing
* **Five encoder heads** — entity (hash-based, fuzzy-first resolution), attribute (dependency-linked), affordance (verb extraction with synonym folding), context (sentence-level), ontology (26 canonical types)
* **Entity-anchored address construction** via dependency-parse-scoped bundling (30-67% address reduction vs. naive Cartesian product)
* **Hash-based entity resolution** with fuzzy-first matching, distributed shared store, and node-level caching
* **Rule-based NLP pipeline** — deterministic tokenization, POS tagging, lemmatization, NER, dependency parsing (10,000+ parses/sec)
* **Progressive fan-out query execution** with ontology hint reordering, satisfaction thresholds, and budget-bounded probing
* **Object-storage-backed persistence** — S3-compatible durable storage with on-demand caching for fast cold starts
* **Per-pod message bus architecture** — pod-local NATS for stage-to-stage communication, no cross-pod coordination for processing
* **CAS-based lease coordination** — no external dependencies beyond the data service for distributed partition management
* **DCP health monitoring and recovery** — automatic detection and restart of exhausted CDC adapters
* **Public query API** with authenticated access
* **Offline calibration pipelines** and deterministic test harnesses

These systems are not stubs — they are exercised regularly at scale.

→ See: [Pipeline Architecture](../ARCHITECTURE/PIPELINE_ARCHITECTURE.md) | [Data Persistence](../ARCHITECTURE/DATA_PERSISTENCE.md)

---

## Security & Operations

NAM now includes a full operational security layer:

### Authentication

* Token-based authentication on all external-facing APIs
* Role-based access control (admin and user roles)
* User management (creation, deletion, password reset)
* Rate limiting on authentication attempts and query volume
* Structured audit logging on every authenticated request

### Network Isolation

* Network-level segmentation between data, ingest, and query layers
* Only designated services accept external connections
* Internal services are not exposed to external networks

### Transport Security

* External-facing services support encrypted transport
* Certificate provisioning supports both external authorities and self-managed certificates

### Encryption at Rest

* Persistent storage volumes support block-level encryption (opt-in)
* Object storage (S3) supports server-side encryption
* Encryption is transparent to the application layer

### Proxy Services

* Optional authenticating proxies for both the query API and direct data access
* Health-aware load balancing across query replicas
* Audit logging at the network boundary

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Lifecycle Management

A dedicated lifecycle service automates cluster operations:

* Waits for the data layer before allowing other services to start
* Coordinates node membership and data distribution
* Verifies storage structures during bootstrap
* Creates initial administrative credentials
* Monitors cluster health on a recurring basis
* Re-joins dropped nodes automatically
* Gates service readiness to prevent operating on an unhealthy cluster

---

## Scaling & Deployment

NAM has been validated under horizontal scaling:

* Multiple ingest replicas coordinate via CAS-based lease claims
* Encoding, storage, and NLP run as independent, scalable services
* Per-pod architecture collocates complementary services for throughput
* Pipeline stages overlap for efficiency
* Node-level entity caching reduces remote lookups across all pods on a node
* The system has exactly one external dependency (the data service) for storage, coordination, and streaming

Deployment today includes:

* Packaged deployment via templated manifests (single command to deploy or upgrade)
* Environment-specific configurations (local development, cloud production)
* Repeatable reset and reload workflows
* Automated cluster bootstrap and health gating
* S3-backed persistent storage with on-demand caching (fast cold starts)
* Automatic DCP health recovery (exhaustion detection, supervisor-driven restart)
* Explicit artifact management
* Observability of addresses, probes, and execution paths

> See: [Design Principles — Deployability](../PHILOSOPHY/DESIGN_PRINCIPLES.md#6-deployability-is-a-core-requirement)

---

## Observability & Debuggability

NAM emphasizes transparency:

* Addresses can be logged and inspected
* Query probes are visible
* Storage lookups are traceable
* Execution paths are explainable
* When retrieval fails, you can identify which axis failed to resolve
* Audit logs trace every authenticated access

This makes NAM suitable for:

* Technical evaluation
* Architectural review
* Regulated or audit-sensitive environments

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Documentation State

Public documentation currently focuses on:

* Explaining the conceptual model
* Defining terminology and invariants
* Setting expectations clearly
* Enabling informed evaluation
* Describing the security and trust model

Implementation details, internal tuning, and private extensions live elsewhere.

> See: [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [FAQ](../APPENDIX/FAQ.md)

---

## Summary

Today, NAM is:

* Architecturally coherent — staged pipeline, entity-anchored bundling, progressive fan-out
* Horizontally scalable — per-pod processing, CAS-based coordination, automatic lease management
* Deterministic by design — rule-based NLP, hash-based entity IDs, reproducible addressing
* Authenticated and access-controlled
* Network-isolated and transport-secured
* Durably persisted — S3-backed storage with on-demand caching and full recovery capability
* Operationally automated — lifecycle management, health gating, DCP recovery
* Proven under real workloads
* Capable of meaningful pilots and production deployments
* Intentionally conservative in behavior

It is early — but it is **solid, secure, and scaling**.
