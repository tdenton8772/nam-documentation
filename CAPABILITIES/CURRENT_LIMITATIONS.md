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

As a result:

* some valid queries fall back to exploratory retrieval
* affordances may not activate until explicitly trained
* affordance confidence thresholds are intentionally strict

This improves trust, but can feel cautious during early pilots.

---

## 3. Entity Resolution Is Still Evolving

Entity detection and normalization currently:

* rely on deterministic normalization rules
* do not perform deep disambiguation
* do not resolve cross-document coreference automatically

This means:

* multiple surface forms may map to separate entities
* entity identity improves with cleaner input data
* advanced entity linking is not yet automatic

This is an active area of refinement, not a solved problem.

---

## 4. Recall Is Geometry-Bound

NAM only retrieves what lies within the probed geometric region.

Today:

* probes are intentionally narrow
* fallback widening is explicit, not automatic
* exhaustive recall is not attempted by default

This can result in:

* fewer results than expected
* “missing” records that lie outside the initial geometry
* the need for explicit exploratory queries

This behavior is intentional but still being tuned for usability.

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

## 7. Operational Maturity Is Early

Current operational constraints include:

* limited deployment automation
* basic observability
* manual artifact promotion

NAM is stable for:

* development environments
* controlled pilots
* internal systems

It is not yet packaged as a fully managed platform.

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

* deterministic ingestion and addressing
* inspectable query execution
* reproducible retrieval behavior
* exploratory semantic navigation
* integration with downstream systems (including LLMs)
* meaningful pilots in well-scoped domains

These are real, working capabilities — not prototypes.

---

## How to Read This Document

If you are:

* **evaluating NAM** → treat this as a candid readiness assessment
* **building with NAM** → use this to scope pilots realistically
* **selling or positioning NAM** → use this to set honest expectations

None of these limitations contradict the core design principles.
They simply reflect where the system is today.
