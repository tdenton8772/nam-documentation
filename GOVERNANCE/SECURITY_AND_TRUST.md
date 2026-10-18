# Security and Trust

NAM is designed to be **trustworthy by construction**, not by policy alone.

This document explains how NAM approaches security, auditability and operational trust, and why those properties are tied to its architecture.

---

## Trust as a System Property

In NAM, trust is not layered on afterward. It comes from:

- Deterministic behaviour on the structural access paths
- Attribution of every result to the path that found it
- Explicit state transitions and durable ingest
- Rebuildable derived data
- Constrained learning surfaces
- Authenticated access at the external boundaries

Operators can reason about *what happened*, *why it happened*, and *what will happen again* under the same conditions.

---

## Determinism and Attribution

Given the same input data, encoder artifacts, ontology and addressing rules, NAM produces the same:

- Addresses and partitions
- Address-path query plans and probes
- Graph index records and walks

The vector path is explicitly approximate, and says so: every vector result carries its cosine, a request cannot lower the deployment's similarity floor, and a read the path could not complete is reported as an error rather than an empty answer.

Across all paths, every fused result lists its `sources`, and every response carries an `access_paths` trace of what each path did, why it ran, and what it contributed. A result is never presented without its provenance.

Nondeterminism on the structural paths is treated as a fault, not an acceptable variance.

---

## Authentication and Access Control

### Identity and Tokens

- Users authenticate with credentials and receive a signed, short-lived token (HS256 JWT)
- Tokens carry identity and role claims
- Every query API request except login, password reset by token, health and readiness checks, and the metrics endpoint requires a valid token
- Expired tokens are rejected; clients refresh explicitly before expiry
- The deployment generates a signing secret by default, so authentication is on in every packaged deployment

### Role-Based Access

- **Admin**: full access, including user management, administrative status and maintenance tasks
- **User**: query and read access

Roles are validated on every request.

### User Lifecycle

- An initial admin user is created during cluster bootstrap
- Users are created, modified and removed through admin endpoints
- Password reset uses single-use, time-limited tokens issued by an admin
- User records live in NAM's own data service; no external identity provider is required

### Supervisor API

The supervisor's administrative API accepts the same token and requires the admin role on every administrative route. Only health and readiness probes are open; the read-only scaling decision that the autoscaler polls requires its own separate token. A supervisor without a signing secret refuses every protected request rather than running unauthenticated: it fails closed, and there is no switch that turns authentication off.

---

## Direct Data Access (KV Proxy)

For clients that need direct key-value access, NAM provides an optional proxy in front of the data service:

- The client presents a JWT as the first thing on the connection; the proxy validates it before any traffic is forwarded
- After the handshake, the proxy inspects each request and allows only key-value data operations, from an explicit allow-list. Re-authentication, bucket switching and flush are refused
- Change-stream (DCP) operations are allowed for admin tokens only, because a change stream can read a whole bucket
- Writes to the source bucket are validated against the document schema
- Connection lifecycle events (authentication, failure, bytes forwarded, duration) are audit-logged

An allow-list is used deliberately: a deny-list fails open on any operation it forgets.

---

## Network Isolation

The chart includes ingress NetworkPolicies for the data service, supervisor, ingest and query tiers. Each policy enumerates the clients allowed to reach that tier.

- Data service: reachable only from enumerated NAM components (and from operator pods explicitly labelled as data clients)
- Supervisor: reachable only from enumerated components
- Ingest and query: reachable only on their service ports from their enumerated callers

These policies are **opt-in**. They should be reviewed against each deployment's rendered output before they are enabled, because enabling them blocks every unlisted client.

Only designated services are exposed externally: the query API (or its optional proxy), the frontend, and optionally the KV proxy.

---

## Proxy Architecture

An optional authenticating reverse proxy can front the query API. It validates tokens at the edge, sets trusted identity headers for the backend, balances across query replicas with health-aware routing, can terminate TLS, and audit-logs every proxied request. It is disabled by default.

---

## Transport Security

- The query service can be published through a TLS ingress with certificates from a managed certificate issuer, and can also serve TLS directly
- Transport encryption and application authentication are independent and are used together
- Traffic between components inside the cluster uses the cluster network; internal transport encryption is a deployment choice

---

## Encryption at Rest

Persistent data volumes can be encrypted at the block level through the storage provisioner, with encryption parameters and passphrase held in a dedicated cluster secret.

Encryption at rest is **opt-in** and off by default. Key rotation is manual.

---

## Audit Logging

The query service, the query proxy and the KV proxy emit structured audit entries. Each includes a timestamp, the authenticated identity, the action, the target path or resource, the client address, the outcome and the duration. Entries are machine-parseable and written to standard output for collection by external log aggregation.

---

## Rate Limiting

- Login attempts are rate-limited per client address
- Queries are rate-limited per authenticated user

Limits are configurable per deployment.

---

## Secrets Management

- Data-service credentials, the token signing secret and administrative passwords are Kubernetes secrets, never embedded in images or committed configuration
- Deployments can reference externally managed secrets instead of generating them
- Maintenance jobs read credentials by secret reference
- Task state, status and history never contain credentials: secret-named fields are stored and served redacted

---

## Safe Maintenance

The supervisor's task framework runs maintenance under explicit rules:

- **Destructive tasks** — bucket flush, cluster reset, object-storage clear and address healing — run only when triggered by hand. They never run on a schedule or by automatic retry, and the configuration API cannot change that
- Runtime overrides are limited to scheduling, timeout and retry fields; anything else is refused
- A task type cannot be started again while a run of it is still executing

---

## Durable Ingest

On the reference deployment, every record handed from the change stream into the pipeline is held in durable, file-backed streams on a per-pod volume until its writes have landed; at minimum, records are held durably from the point their head results are joined. The log rejects new work when full rather than evicting unacknowledged records. A failed batch is redelivered; a batch that exhausts its retries is counted as an error rather than dropped silently. Reads that fail at query time are reported as degraded results, not as empty ones.

---

## Derived State and Rebuildability

NAM never treats its indexes as authoritative. Addresses, vectors and graph records are all:

- Derived from source records
- Reconstructible from source data plus versioned artifacts
- Disposable if corruption or drift is suspected

Rebuildability is a core safety mechanism.

---

## Constrained Learning Surfaces

- Encoder heads, the address codec and the embedding model do not change at query time
- Training is explicit, offline and auditable; artifacts are versioned and validated strictly when loaded
- Which implementation fills each encoder-head slot changes only when an admin selects one, through an audited API that refuses a change made against an outdated manifest
- Runtime behaviour is fixed for a given set of artifacts and head selections

NAM learns *new knowledge* (addresses, entities, surface forms), not *new rules*, during operation.

---

## Data Exposure and Isolation

Each deployment holds its own indexes, including its own vector index; nothing is shared across deployments. Tenant and partition are segments of every address, and the vector index is kept per tenant, which separates data logically within a deployment. Access tokens do not yet carry tenant scopes, so deployments that need hard isolation should run separate instances. Deployments can be single-tenant, domain-specific or air-gapped.

---

## Failure Modes Are Visible

When NAM cannot answer well, it returns fewer results, widens explicitly, and says so in the trace. When storage reads fail, the response is flagged as partial with the failing sources named, or returns a service-unavailable status if nothing could be read. When cluster health degrades, readiness gates stop new work and the supervisor reports the unhealthy components.

---

## Security Is Architectural, Not Procedural

NAM does not rely on prompt hardening, heuristic filters or security through obscurity. It relies on:

- Deterministic, attributed retrieval
- Authenticated access at external boundaries, fail-closed administration, and an allow-listed data proxy
- Optional network isolation and encryption at rest
- Structured audit logging
- Durable, acknowledged ingest
- Constrained learning surfaces and rebuildable state

---

## Summary

NAM is designed to be trusted because it is predictable, attributed, authenticated, auditable, inspectable, rebuildable and constrained. Trust comes from understanding how it works — and knowing it will work the same way tomorrow.

---

> See also: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [Contributions](CONTRIBUTIONS.md) | [High-Level Systems](../ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md) | [Deployment Architecture](../ARCHITECTURE/DEPLOYMENT_ARCHITECTURE.md)
