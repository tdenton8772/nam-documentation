# Security and Trust

NAM is designed to be **trustworthy by construction**, not by policy alone.

This document explains how NAM approaches security, auditability, and operational trust — and why those properties are inseparable from its architectural choices.

---

## Trust as a System Property

In NAM, trust is not layered on afterward.

It emerges from:
- Deterministic behavior
- Explicit state transitions
- Rebuildable derived data
- Constrained learning surfaces
- Authenticated access at every boundary

The system is designed so that operators can reason about *what happened*, *why it happened*, and *what will happen again* under the same conditions.

---

## Determinism as a Security Primitive

Determinism is a first-order design principle in NAM.

Given the same:
- Input data
- Encoder configurations
- Ontology definitions
- Addressing rules

NAM will produce the same:
- Addresses
- Partitions
- Query probes
- Retrieval behavior

This enables:
- Reproducibility
- Forensic analysis
- Controlled deployments
- Predictable upgrades

Nondeterministic behavior is treated as a system fault, not an acceptable variance.

---

## Authentication and Access Control

NAM enforces authenticated access to all external-facing services.

### Identity and Tokens

- Users authenticate with credentials and receive a signed, time-limited token
- Tokens carry identity and role claims
- Every API request (except health checks and login) requires a valid token
- Token expiry is configurable and enforced strictly — expired tokens are rejected, not refreshed silently

### Role-Based Access

NAM supports role-based access:

- **Admin**: full access including user management
- **User**: query and read access

Roles are encoded in the token and validated on every request.

### User Lifecycle

- An initial admin user is created during cluster bootstrap
- Additional users are created, modified, and removed through management endpoints
- Password reset is supported via single-use, time-limited tokens
- User records are stored in the same data service used by the rest of the system — no external identity provider is required

### Access Control Is Not Optional

Authentication is a structural requirement, not a configuration choice.

Every path from outside the cluster to internal services passes through an authenticated boundary. This is true for both the query API and direct data access.

---

## Network Isolation

NAM supports network-level isolation between its internal components.

### Segmentation Model

The system separates concerns into distinct network zones:

- **Data layer**: Storage nodes accept connections only from authorized internal services (ingest, query, lifecycle management, entity caching)
- **Ingest layer**: Pipeline workers communicate only with each other (via a local message bus) and with the data layer
- **Query layer**: Query services communicate only with the data layer

### External Boundaries

Only designated services accept external connections:

- The query API (or its proxy) serves user-facing traffic
- Direct access to storage is optionally mediated by an authenticating proxy
- No internal services are exposed to external networks

When network isolation is enabled, unauthorized lateral movement between components is structurally prevented — not just discouraged by convention.

---

## Proxy Architecture

NAM provides optional proxy services for mediating external access.

### Query Proxy

An authenticating reverse proxy can front the query API:

- Validates tokens at the network edge, before requests reach the application
- Sets trusted identity headers for the backend
- Distributes load across multiple query replicas with health-aware routing
- Supports transport encryption termination
- Produces structured audit logs for every proxied request

### Data Access Proxy

For deployments that expose the data service's key-value protocol to external clients:

- A lightweight proxy validates tokens on connection
- After authentication, the proxy performs transparent protocol-level forwarding
- No per-operation overhead after the initial handshake
- Connection lifecycle (connect, disconnect, bytes transferred, duration) is audit-logged

Both proxies are optional and disabled by default. They exist for deployments that require defense-in-depth at the network boundary.

---

## Transport Security

NAM supports encrypted transport between clients and services.

- Query endpoints can be served over TLS
- Certificate provisioning supports both external certificate authorities and self-managed certificates
- Transport encryption is independent of application-level authentication — both can be enabled together for defense in depth

Internally, services within a cluster communicate over the cluster network. Transport encryption between internal components is a deployment choice, not a system requirement.

---

## Encryption at Rest

NAM supports encryption of persistent storage volumes.

When enabled:

- Storage volumes are encrypted transparently at the block level
- Encryption is configured via the storage provisioner — no application-level changes required
- Encryption parameters (cipher, key derivation, key size) are configurable
- Encryption secrets are managed as cluster secrets, separate from application credentials

Encryption at rest is opt-in and disabled by default for development environments. Production deployments can enable it without changes to the application layer.

Key rotation is not yet automated. The system is designed to support rotation workflows in the future, but this requires coordination at the storage layer that is currently manual.

---

## Audit Logging

NAM produces structured audit logs across all authenticated boundaries.

Every audit entry includes:

- Timestamp
- Authenticated identity (or "anonymous" for public paths)
- Action performed
- Target path or resource
- Client address
- Response status
- Duration

Audit logs are:

- Machine-parseable (structured format)
- Emitted to standard output for collection by external log aggregation
- Produced by both application services and proxy services

This enables:

- Forensic investigation of access patterns
- Detection of anomalous behavior
- Compliance reporting
- Operational monitoring

Audit logging is not optional — it is a structural property of every authenticated boundary.

---

## Rate Limiting

NAM enforces rate limits on sensitive operations.

- Authentication attempts are rate-limited per source address
- Query operations are rate-limited per authenticated user

Rate limits prevent:

- Credential brute-force attacks
- Resource exhaustion from excessive queries
- Abuse of public-facing endpoints

Limits are configurable per deployment but always enforced.

---

## Lifecycle Management

NAM includes automated cluster lifecycle management.

A dedicated lifecycle service:

- Waits for the data layer to become available before allowing other components to start
- Coordinates node membership and data distribution
- Verifies expected storage structures exist before signaling readiness
- Creates initial administrative credentials during bootstrap
- Monitors cluster health on a recurring basis
- Re-joins dropped nodes automatically

### Readiness Gating

Ingest and query services do not start processing until the lifecycle service confirms the cluster is healthy and properly configured.

This prevents:

- Ingestion against misconfigured storage
- Queries against an uninitialized cluster
- Race conditions during startup

---

## Secrets Management

NAM separates credentials from configuration.

- Data service credentials, authentication signing keys, and administrative passwords are managed as cluster secrets
- Secrets are never embedded in configuration files or container images
- Deployments can reference externally managed secrets instead of creating new ones
- Encryption passphrases (when encryption at rest is enabled) are stored separately from application secrets

This supports integration with external secret management systems without requiring changes to the application.

---

## Derived State and Rebuildability

NAM does not treat indexed or addressed data as authoritative.

All stored state is:
- Derived from original inputs
- Reconstructible from raw data + configuration
- Disposable if corruption or drift is suspected

This dramatically reduces:
- Long-lived hidden state
- Silent corruption
- Irrecoverable indexing errors

Rebuildability is a core safety mechanism.

---

## Constrained Learning Surfaces

NAM does not allow unconstrained learning inside the runtime system.

Specifically:
- Encoder heads do not self-modify at query time
- Training is explicit, offline, and auditable
- Learned artifacts are versioned and inspectable
- Runtime behavior is fixed for a given configuration

This prevents:
- Prompt-based exploitation
- Gradual semantic drift
- Undocumented behavior changes
- Emergent decision logic

NAM learns *new knowledge*, not *new rules*, during operation.

---

## Explicit Boundaries Between Components

NAM enforces strict boundaries between:
- Ingestion
- Encoding
- Addressing
- Storage
- Query planning
- Retrieval
- Authentication
- Lifecycle management

Each boundary:
- Has a clear contract
- Can be reasoned about independently
- Can be swapped or replaced intentionally

This separation limits blast radius and makes failures diagnosable.

---

## Auditability and Explainability

NAM is designed to support audit trails without requiring introspection into opaque models.

For any result, the system can surface:
- Which addresses were probed
- Which partitions were accessed
- Which records matched geometrically
- Which dimensions were unconstrained
- Who requested the query (authenticated identity)

This makes it possible to explain *why something was returned* without invoking probabilistic reasoning — and *who asked for it* without relying on external access logs.

---

## Data Exposure and Isolation

NAM does not require:
- Centralized embeddings
- Cross-tenant similarity spaces
- Shared semantic indices

Deployments can be:
- Fully isolated
- Single-tenant
- Air-gapped
- Domain-specific

This supports high-sensitivity environments where data separation is non-negotiable.

---

## Failure Modes Are Visible

NAM is designed so that failure is observable.

When NAM cannot answer a question:
- It returns fewer results
- It widens probes explicitly
- It surfaces uncertainty structurally

It does not hallucinate completeness or correctness.

When cluster health degrades:
- The lifecycle service logs unhealthy components
- Readiness gates prevent serving on an unhealthy cluster
- Audit logs capture failed or degraded requests

This makes trust a matter of *understanding limits*, not hiding them.

---

## Security Is Architectural, Not Procedural

NAM does not rely on:
- Model prompt hardening
- Heuristic filters
- Post-hoc validation layers
- Security through obscurity

Instead, it relies on:
- Deterministic execution
- Authenticated access at every boundary
- Network-level isolation between components
- Encrypted transport and storage
- Structured audit logging
- Constrained learning surfaces
- Rebuildable state
- Automated lifecycle management

Security emerges from structure, not rules.

---

## Summary

NAM is designed to be trusted because it is:
- Predictable
- Authenticated
- Isolated
- Encrypted
- Auditable
- Inspectable
- Rebuildable
- Constrained

These properties are intentional, not incidental.

Trust in NAM comes from understanding how it works — and knowing that it will work the same way tomorrow.

---

> See also: [Design Principles](../PHILOSOPHY/DESIGN_PRINCIPLES.md) | [Contributions](CONTRIBUTIONS.md) | [High-Level Systems](../ARCHITECTURE/HIGH_LEVEL_SYSTEMS.md)
