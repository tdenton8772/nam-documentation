# Current Limitations

This document describes the **current, observable limitations** of NAM as it exists today.
These are not philosophical constraints or permanent non-goals — those are covered elsewhere.

Everything here is about **maturity, scope, and operational reality**.

---

## 1. Limited Domain Coverage by Default

Out of the box, NAM does not understand *every* domain.

* Encoder heads ship with general-purpose ontologies
* Affordances are trained from curated corpora
* Domain specificity must be introduced intentionally

This means:

* NAM performs best in domains it has been trained on
* New domains require encoder calibration and artifact generation
* Zero-shot domain performance is intentionally conservative

This is expected behavior, not a failure mode.

---

## 2. Affordance Detection Is Conservative

Affordance detection today is designed to:

* avoid false positives
* prefer exploratory mode over incorrect precision
* err on “unknown” rather than “wrong”

Recent improvements:

* copular constructions (“X is Y”) now produce “classify” or “describe” affordances instead of being filtered by the light verb firewall
* query canonicalizer detects interrogative intent (who/what/where/when/how) and injects affordance hints when heads produce NULL

As a result:

* some valid queries still fall back to exploratory retrieval
* affordances may not activate until explicitly trained
* affordance confidence thresholds are intentionally strict

This improves trust, but can feel cautious during early pilots.

---

## 3. Entity Resolution Is Stable but Still Improving

Entity resolution has been significantly hardened:

* entities are identified by **hash-based, type-independent identifiers** — the same entity always gets the same ID regardless of ontology classification
* resolution uses a **fuzzy-first strategy** — candidate key lookups before exact surface form matching
* the entity store is shared across all replicas with deterministic resolution order
* node-level caching reduces resolution latency to microseconds for warm lookups

Current constraints:

* deep cross-document coreference is not yet automatic
* entity identity improves with cleaner input data and broader surface form coverage
* entity deduplication (merging near-identical fuzzy matches) is planned but not yet implemented

→ See: [Current State — Entity Resolution](../ROADMAP/CURRENT_STATE.md#hash-based-entity-resolution)

---

## 4. Recall Is Geometry-Bound

NAM only retrieves what lies within the probed geometric region.

Today:

* **progressive fan-out** automatically widens from specific to general probes within each ontology tier
* **inter-tier satisfaction** stops probing when enough results are found, which may leave some tiers unexplored
* ontology tier reordering based on bundler hints improves recall for the most likely types

This can result in:

* fewer results than expected for queries spanning many ontology types
* results concentrated in the first few tiers when satisfaction is reached early
* the need for explicit exploratory queries to access less-common ontology tiers

This behavior is intentional — bounded recall is a feature, not a bug. The satisfaction threshold and probe budgets are tunable.

---

## 5. Limited Tooling for Corpus Preparation

Preparing data for NAM currently requires:

* some manual preprocessing
* understanding of encoder expectations
* careful corpus selection

There is not yet:

* a turnkey ingestion wizard
* automatic schema discovery
* one-click domain onboarding

Early adopters should expect hands-on setup.

---

## 6. Evaluation Requires Intentional Test Design

NAM cannot be evaluated with:

* traditional IR benchmarks
* recall/precision curves
* similarity-based scoring tests

Meaningful evaluation today requires:

* scenario-driven tests
* known-answer queries
* geometry-aware expectations

This makes evaluation more rigorous — but also more demanding.

---

## 7. Operational Maturity Is Progressing

Deployment is packaged and repeatable, with automated lifecycle management, authenticated access, network isolation, transport security, and S3-backed persistent storage. A web-based dashboard provides system health monitoring, encoder head status, pipeline metrics with time-series graphs, and query interfaces.

Current operational constraints include:

* limited external alerting integrations (metrics are visible but not yet exported to external monitoring systems)
* manual artifact promotion
* no turnkey multi-tenant isolation
* encryption key rotation is manual

NAM is stable for:

* development environments
* controlled pilots
* internal and partner deployments

The storage layer now supports **object-storage-backed persistence** with on-demand caching, enabling fast cold starts and data recovery without local disk dependencies. This significantly improves operational resilience.

It is approaching production-grade operations but is not yet a fully managed platform.

> See: [Current State -- Scaling](../ROADMAP/CURRENT_STATE.md#scaling--deployment) | [Data Persistence](../ARCHITECTURE/DATA_PERSISTENCE.md)

---

## 8. Small Team / Single-Threaded Development Reality

Practically speaking:

* development velocity is constrained
* features are prioritized carefully
* correctness is favored over speed of expansion

This is a reality of stage, not a reflection of ambition.

---

## What *Does* Work Well Today

Despite these limitations, today’s NAM implementation already supports:

* deterministic ingestion and addressing via a staged, rule-based pipeline
* hash-based entity resolution with fuzzy-first matching
* progressive fan-out query execution with early termination
* covering index (N cyclic rotations per base address) enabling single-prefix-scan retrieval for any axis combination
* LCA byte-code coordinate encoding with codebook neighborhood fan-out
* entity codebook with rejection classes (~42 addresses/record average)
* ontology-aware retrieval with 26 canonical types (no catch-all fallback)
* S3-backed persistent storage with on-demand caching
* CDC-based continuous ingestion with automatic lease coordination
* graph index with 19.7M+ entries written directly by the addressing pipeline
* query canonicalization with intent detection and entity expansion
* copular affordance handling and NOUN compound/appositive attribute extraction
* context keyword fallback for domain and temporal signals
* ETMPFAIL retry with exponential backoff for write queue saturation resilience
* inspectable query execution and reproducible retrieval behavior
* exploratory semantic navigation
* validated at 500K records with 13.3M+ semantic addresses and 0 write failures
* integration with downstream systems (including LLMs)
* meaningful pilots in well-scoped domains

These are real, working capabilities — not prototypes.

→ See: [Current State](../ROADMAP/CURRENT_STATE.md) for detailed subsystem status

---

## How to Read This Document

If you are:

* **evaluating NAM** → treat this as a candid readiness assessment
* **building with NAM** → use this to scope pilots realistically
* **selling or positioning NAM** → use this to set honest expectations

None of these limitations contradict the core design principles.
They simply reflect where the system is today.
