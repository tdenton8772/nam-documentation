# Current Limitations

This document describes the **current, observable limitations** of NAM as it exists today.
These are not philosophical constraints or permanent non-goals — those are covered in [What NAM Cannot Do](WHAT_NAM_CANNOT_DO.md).

Everything here is about **maturity, scope, and operational reality**.

---

## 1. Ontology Classification Often Abstains

Every address carries a BFO/CCO class in its type segment. The ontology head assigns that class from a trained codebook and, when the evidence is weak, backs off to a broader ancestor class — and ultimately to `BFO:0000001` (*entity*), which asserts nothing beyond existence.

What this means today:

* A meaningful share of records and addresses abstain to `BFO:0000001`
* An abstention is treated as a **classification miss**, not as a valid category: the pipeline counts abstaining records and addresses as a quality metric
* The query planner compensates with a safety-net probe of the `BFO:0000001` partition for every query entity, so abstaining records stay reachable — at the cost of extra scans
* The classes downstream heads are tuned against (attribute and affordance relevance, context defaults) are keyed by BFO/CCO class, and their tuning against the full class set is ongoing

Abstention is the honest outcome when evidence is thin, but a high abstention rate weakens the type segment as a partitioning signal.

---

## 2. Attribute, Affordance and Context Axes Are Frequently Empty

Each address has three axes — attribute, affordance and context — and an axis with no value is stored as the non-specific coordinate `lca:0:0`.

Today:

* Many addresses carry `lca:0:0` on one or more axes, most often on the attribute axis
* When axes are empty, an address is mostly identified by its type and entity segments, so the address path narrows less than the model allows
* Axis health (empty rate, distinct values, dominance of a few values, overlap with the type segment) is measured with dedicated anti-collapse metrics, and improving axis coverage is active work

Empty axes are correct behaviour — they mean "not constrained here" — but they limit how much precision the address path can deliver.

---

## 3. Entity Resolution Is Deterministic but Imperfect

Entities are identified by **hash-based, type-independent identifiers**: the same normalized name always yields the same identifier on every pod. Query-time resolution uses the same resolver as the ingest entity head, reading the shared session store.

Current constraints:

* Resolution is name-based. Cross-document coreference ("the president" meaning a specific person) is not automatic
* Near-duplicate entities (spelling variants that normalize differently) are not merged
* The entity *type* recorded in the session store uses the entity head's own vocabulary (for example `person` or `compound_noun`), not BFO/CCO classes, and it is sometimes wrong. The address type segment comes from the ontology head instead, so this does not change where addresses are stored, but it can mislead query-side hints derived from the stored type
* Query-time resolution reads the session store, so its latency depends on data-service responsiveness

---

## 4. The Graph Index Lags the Address Index

The graph index is built by a separate, sharded index builder that follows the address store's change stream. It is never on the ingest acknowledgement path.

As a result:

* After a load, graph traversal can return fewer records than the address index already holds, until the builder catches up
* The builder can be paused for operational reasons. While it is paused, the graph keeps its checkpoints and catches up when it resumes, but it does not grow in the meantime
* Graph results can therefore differ from address results for the same entity for a period after ingest

---

## 5. Vector Search Is Approximate by Design

The vector path returns the top-K records above a cosine floor. It is explicitly approximate:

* It does not guarantee the globally best K records, or every record above the floor
* The cosine floor is a **per-model calibration**. A floor tuned for one embedding model does not carry over to another, and changing the model means re-embedding and reindexing
* Query and ingest embeddings must come from the same model, used the same way; the vector services refuse to start on a mismatched model rather than search a different vector space
* Updating or deleting a source record does not remove its earlier vector entry. Results are de-duplicated by record, but an outdated embedding can still match until the vector index is rebuilt

Vector results are always labelled with their source (`sources` contains `vector`, with a `cosine` score), so callers can tell them apart from address and graph results.

---

## 6. Embedding Throughput Depends on an External GPU Provider

Embedding at ingest runs on a local CPU tier by default. When a backlog builds, the vector workers can claim a GPU from an external provider through a separate GPU registry and stream to it.

Constraints:

* The burst tier depends on provider capacity being available. If no GPU can be placed, embedding continues on the CPU tier, much more slowly
* The CPU and GPU tiers must produce compatible vectors. Both tiers run the same model at the same precision, so that a record embedded on one is visible to a query embedded on the other
* GPU claims are held for a period of idle time before release, so a burst has a cost floor

---

## 7. Pipeline Throughput Is Bound by Compute CPU

Ingest throughput is limited by the CPU of the nodes that run the pipeline workers (NLP, heads, addressing), not by the data service and not by the number of worker instances. Adding workers on the same nodes redistributes CPU rather than adding throughput. Scaling ingest means adding compute, or further reducing per-record CPU in the heads. On the reference deployment the ingest tier sizes itself to its backlog and compute nodes are added as its pods need them, within the deployment's configured maximum.

---

## 8. Query Behaviour Is Still Being Refined

* **Affordance detection is conservative.** Many valid questions fall back to exploratory mode because the affordance vocabulary does not cover them
* **Address results are not quality-ranked.** A full page of address results is not proof of a good answer, which is why vector stays on alongside address for most queries, and the thin-result fallback is a count, not a quality judgement
* **Corpus hygiene matters.** Whatever is in the source bucket is addressed and returned. Applications that store their own working data (for example conversation history) alongside corpus records will see that data in results unless they filter it

---

## 9. Limited Tooling for Corpus Preparation

Preparing data for NAM currently requires:

* manual preprocessing into the expected document schema
* understanding of encoder expectations
* careful corpus selection

There is not yet a turnkey ingestion wizard, automatic schema discovery, or one-click domain onboarding. Early adopters should expect hands-on setup.

---

## 10. Multi-Tenancy Is Logical, Not Enforced

Tenant and partition are segments of every address, and the vector index is kept per tenant, so records written under different tenants live in different key ranges. However:

* Access tokens do not carry tenant scopes, so tenant isolation is not enforced at the API boundary
* Per-tenant encoder configurations and artifacts are not supported
* Tenant-scoped audit logging is not implemented

Deployments that need hard isolation today should run separate instances.

---

## 11. Operational Maturity Is Progressing

Deployment is packaged (a single Helm chart), lifecycle is automated, and the query API is authenticated. Current operational constraints include:

* **Alert notifications are not shipped.** The shipped alert rules are evaluated and firing alerts show in the dashboard, but nothing routes them to email, chat or a pager; a deployment that needs notifications adds its own routing
* **Network policies are opt-in.** The chart contains ingress policies for the data service, supervisor, ingest and query tiers, but they are disabled by default and must be reviewed per deployment before they are switched on
* **Encryption at rest is opt-in** and depends on the storage provisioner. Key rotation is manual
* **Cross-cluster replication** endpoints exist in the API, but replication is not configured in any current deployment
* **Object storage maintenance** (orphan file cleanup, compaction) runs as supervisor tasks and remote workers, one job per data node. Superseded files are deleted only after a settle window, so object storage briefly holds more than the live data after heavy compaction or a flush

NAM is stable for development environments, controlled pilots, and internal and partner deployments. It is not yet a fully managed platform.

> See: [Current State](../ROADMAP/CURRENT_STATE.md) | [Data Persistence](../ARCHITECTURE/DATA_PERSISTENCE.md) | [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)

---

## 12. Evaluation Requires Intentional Test Design

The address path cannot be evaluated meaningfully with similarity benchmarks alone. Useful evaluation requires:

* scenario-driven tests with known-answer queries
* per-path measurement: the `access_paths` trace shows what each path did and contributed, so address, vector and graph can be judged separately
* geometry-aware expectations for the address path, and precision/recall at a stated cosine floor for the vector path

---

## 13. Small Team Reality

Development velocity is constrained, features are prioritized carefully, and correctness is favoured over speed of expansion. This is a reality of stage, not a reflection of ambition.

---

## How to Read This Document

* **Evaluating NAM** → treat this as a candid readiness assessment
* **Building with NAM** → use this to scope pilots realistically
* **Positioning NAM** → use this to set honest expectations

None of these limitations contradict the core design principles. They reflect where the system is today.

→ See also: [What NAM Can Do](WHAT_NAM_CAN_DO.md) | [Current State](../ROADMAP/CURRENT_STATE.md) | [Near-Term Roadmap](../ROADMAP/NEAR_TERM.md)
