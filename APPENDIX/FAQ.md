# Frequently Asked Questions (FAQ)

This FAQ answers the questions we hear most often from engineers, architects, and operators encountering NAM for the first time.

If you’re looking for marketing claims, this is not the right place.
If you’re looking to understand what NAM *actually does*, this is.

---

## What problem is NAM solving?

NAM addresses a specific gap:

> How do you retrieve meaningful, explainable knowledge from large corpora **without collapsing meaning into probabilistic similarity**?

NAM is built for environments where:
- Determinism matters
- Recall boundaries must be understood
- Answers must be inspectable
- Semantic drift is unacceptable

It is not designed to replace LLMs, search engines, or databases — it complements them where those tools break down.

---

## Is NAM a vector database?

No.

NAM does not rely on nearest-neighbor similarity, distance metrics, or global embedding spaces.

Retrieval in NAM is geometric and exact:
- Queries generate probes
- Probes intersect address space
- Records match by structural alignment, not similarity score

Vectors may exist *inside* encoder heads, but they are not the retrieval primitive.

---

## Is NAM a knowledge graph?

Not in the traditional sense.

NAM does not:
- Require predefined edges
- Store explicit triples
- Traverse relationship graphs at query time

Relationships in NAM emerge from **shared address geometry**, not graph traversal.

You can derive graphs from NAM outputs — but NAM itself is not a graph engine.

---

## Is NAM a RAG system?

No — but it can support RAG systems.

NAM does not:
- Rank chunks by similarity
- Optimize for prompt stuffing
- Assume an LLM is the consumer

Instead, NAM focuses on **semantic retrieval correctness**.  
If you feed those results into an LLM, you get a more reliable RAG pipeline — but NAM stands independently.

---

## Does NAM “understand” language?

NAM does not claim human-like understanding.

It performs:
- Structured semantic interpretation
- Ontology-aware encoding
- Deterministic address assignment

Understanding, in NAM, means *consistent semantic placement*, not generative reasoning.

---

## Does NAM learn in real time?

Yes — but in a constrained and intentional way.

NAM learns by:
- Accumulating new addressed records
- Expanding the family of known addresses
- Increasing coverage within existing semantic geometry

NAM does **not**:
- Change its encoder logic at runtime
- Invent new strategies
- Mutate behavior unpredictably

It gains **knowledge**, not **new rules**.

---

## What happens when NAM cannot answer a question?

NAM fails *structurally*, not creatively.

That means:
- Fewer results
- Wider probes
- Explicit uncertainty

NAM does not hallucinate completeness or correctness.

If geometry does not intersect, the system makes that visible.

---

## How is this different from “just better embeddings”?

Embeddings collapse meaning into distance.
NAM preserves meaning as structure.

Distance-based systems must guess relevance.
NAM tests alignment.

This difference matters when:
- Exact recall matters
- False positives are dangerous
- Explanations are required
- Systems must behave identically across environments

---

## Can NAM be trained on my domain?

Yes.

Encoder heads, ontologies, and artifacts are designed to be:
- Domain-specific
- Separately trained
- Versioned
- Auditable

This is a core design goal, not an afterthought.

---

## Does NAM require an LLM?

No.

LLMs can be used:
- During training
- For labeling
- As downstream consumers

But NAM does not depend on them at runtime.

---

## Is NAM deterministic?

Yes — by design.

Given the same inputs and configuration, NAM will:
- Produce the same addresses
- Probe the same geometry
- Return the same results

This is non-negotiable.

---

## Can NAM scale?

NAM scales horizontally along familiar axes:
- **Partition-based distribution** — data is divided across partitions, each owned by one ingest pod
- **CAS-based lease coordination** — pods claim and shed partitions without external coordinators
- **Per-pod pipeline isolation** — each pod runs a complete processing pipeline with a local message bus
- **Parallel entity extraction** — the most I/O-intensive encoder head runs multiple workers per pod
- **Asynchronous storage writes** — addressing never blocks the pipeline; writes are batched in the background

Adding pods triggers automatic rebalancing. Removing pods triggers automatic failover. Throughput scales linearly with pod count.

→ See: [Current State — Scaling](../ROADMAP/CURRENT_STATE.md#scaling--deployment) | [Pipeline Architecture](../ARCHITECTURE/PIPELINE_ARCHITECTURE.md)

---

## What kinds of problems is NAM *not* good at?

NAM is not a good fit for:
- Open-ended text generation
- Creative writing
- Approximate semantic similarity search
- Replacing human judgment

These are intentional non-goals.

---

## Is NAM production-ready today?

NAM is an active system in production-pilot phase with proven horizontal scaling and operational security.

It is suitable for:
- Controlled pilots and internal deployments
- Domain-specific evaluations
- Partner integrations with hands-on support

Packaged deployment, automated lifecycle management, authenticated access, network isolation, and transport security are operational. The core pipeline (rule-based NLP, entity-anchored bundling, hash-based entity resolution, progressive fan-out queries) is exercised against real corpora.

Recent hardening includes:
- S3-backed persistent storage with on-demand caching for fast cold starts
- Hash-based, type-independent entity identifiers for stable ingest/query alignment
- Ontology system with 26 canonical types (no catch-all fallback)
- Automatic DCP health recovery and supervisor-driven pod restart

It is not yet a fully managed platform.

> See: [Current State](../ROADMAP/CURRENT_STATE.md) | [Current Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md)

---

## How does NAM persist data?

NAM uses a **distributed key-value store backed by S3-compatible object storage**.

Data is written locally, then asynchronously replicated to object storage. On restart:
- Only metadata manifests are downloaded (fast cold start)
- Individual data files are fetched on demand when first accessed
- This enables startup times measured in seconds, not minutes

All address state is **derived** — it can be rebuilt by re-ingesting source data. The address index is never a system of record.

> See: [Data Persistence](../ARCHITECTURE/DATA_PERSISTENCE.md)

---

## How does NAM handle failures?

NAM is designed for **automatic recovery** at multiple levels:

- **Pod failure**: Leases expire, surviving pods reclaim partitions, CDC resumes from checkpoints
- **Storage failure**: S3 serves as durable backup; local state is rebuilt on demand
- **CDC adapter exhaustion**: Consecutive failure detection triggers health reporting, supervisor kills and restarts the pod
- **Complete cluster loss**: All state is recoverable from S3; coordination state starts fresh

Recovery is automatic — no manual intervention required for common failure modes.

---

## How does NAM handle security?

Security in NAM is structural, not bolted on.

At the **design level**:
- Determinism eliminates an entire class of nondeterministic exploits
- Constrained learning prevents prompt-based or query-time manipulation
- Derived state is rebuildable, eliminating long-lived hidden corruption

At the **operational level**:
- All external access is authenticated with signed, time-limited tokens
- Role-based access control separates administrative and user operations
- Network isolation segments internal components from external traffic
- Transport encryption protects data in transit
- Encryption at rest is available for persistent storage
- Structured audit logs trace every authenticated access
- Automated lifecycle management prevents operating on an unhealthy or misconfigured cluster

NAM does not rely on prompt hardening, heuristic filters, or security through obscurity.

> See: [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## Why does NAM look "different" from other AI systems?

Because it was designed backward from failure cases.

NAM exists because:
- Similarity is not meaning
- Recall without explanation is dangerous
- Nondeterminism breaks trust
- Black boxes do not belong everywhere

The system reflects those beliefs.

---

## Where should I start next?

If you want:
- Conceptual clarity → [Architecture](../ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md)
- Practical constraints → [Capabilities](../CAPABILITIES/WHAT_NAM_CAN_DO.md) | [Limitations](../CAPABILITIES/CURRENT_LIMITATIONS.md)
- Current status → [Current State](../ROADMAP/CURRENT_STATE.md)
- Future direction → [Near-Term](../ROADMAP/NEAR_TERM.md) | [Long-Term](../ROADMAP/LONG_TERM.md)
- Definitions → [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [Glossary](GLOSSARY.md)

This documentation is designed to be read non-linearly.

→ See also: [What Is NAM](../OVERVIEW/WHAT_IS_NAM.md) | [Terminology](../PHILOSOPHY/TERMINOLOGY.md) | [Current State](../ROADMAP/CURRENT_STATE.md)
