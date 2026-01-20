# Current State

This document describes the **current state of the NAM project today** — what exists, what works, and what is actively being exercised in real systems.

It is intentionally factual and implementation-agnostic.
This is not a roadmap, pitch, or vision statement.

---

## Project Maturity

NAM is currently in a **working prototype / early production-pilot phase**.

This means:

* Core architectural ideas are implemented, not theoretical
* Major subsystems exist and interact end-to-end
* Behavior is deterministic and observable
* APIs are stable enough for pilots, not yet frozen

The system is past the “research” phase and not yet a turnkey product.

---

## Core System Capabilities (Today)

As of now, NAM provides:

### Deterministic Ingestion

* Records are ingested through a defined pipeline
* Encoder heads emit structured semantic signals
* All derived state is reproducible from inputs
* No hidden learning or runtime mutation occurs

### Semantic Addressing

* Records are mapped to multi-axis semantic addresses
* Address construction is deterministic
* Address structure is inspectable and debuggable
* Storage keys reflect semantic geometry, not similarity

### Geometric Retrieval

* Queries generate probes into semantic space
* Retrieval operates over points, lines, planes, and regions
* Exploratory and targeted queries are supported
* Results are explainable by address overlap

### Offline Training & Artifacts

* Encoder heads are trained offline
* Training emits fixed, versioned artifacts
* Artifacts are promoted explicitly
* Runtime never retrains or mutates models

### Affordance-Aware Querying

* Queries are analyzed for affordances
* Affordances select execution strategies
* Unsupported affordances safely fall back to exploration
* Behavior is consistent across runs

---

## Implemented Subsystems

Today’s NAM implementation includes:

* Pluggable encoder head framework
* Addressing and partitioning logic
* Storage abstraction layer
* Query planner and runner
* Public query API
* Ingestion adapters
* Offline training pipelines
* Deterministic test harnesses

These systems are not stubs — they are exercised regularly.

---

## Observability & Debuggability

NAM emphasizes transparency:

* Addresses can be logged and inspected
* Query probes are visible
* Storage lookups are traceable
* Execution paths are explainable

This makes NAM suitable for:

* technical evaluation
* architectural review
* regulated or audit-sensitive environments

---

## Deployment Reality

Currently, NAM is suitable for:

* Developer environments
* Controlled pilots
* Internal or partner deployments

Deployment today involves:

* manual setup
* explicit artifact management
* hands-on configuration

This is expected at this stage and is being addressed incrementally.

---

## Documentation State

Public documentation currently focuses on:

* explaining the conceptual model
* defining terminology and invariants
* setting expectations clearly
* enabling informed evaluation

Implementation details, internal tuning, and private extensions live elsewhere.

---

## Summary

Today, NAM is:

* architecturally coherent
* functionally real
* deterministic by design
* capable of meaningful pilots
* intentionally conservative in behavior

It is early — but it is **solid**.
