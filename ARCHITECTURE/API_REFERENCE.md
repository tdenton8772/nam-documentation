# API Reference

*Public endpoints for querying, authentication, and administration*

This document describes the HTTP API exposed by the NAM **query service**, plus the authenticated KV proxy. Internal service-to-service APIs (ingest health, supervisor internals) are not covered, except where the query service forwards to them.

All endpoints use JSON request and response bodies.

---

## Authentication

NAM uses **JWT** bearer tokens.

* Every endpoint except login, password reset by token, health and readiness checks, and the two metrics endpoints requires a valid token when authentication is enabled. `/metrics` is open; `/v1/metrics/federate` takes basic auth (below).
* Tokens carry identity and a role claim (`user` or `admin`). Admin endpoints require `admin`; a non-admin token receives `403`, a missing or invalid token `401`. A third role, `metrics`, is for scraper accounts: they cannot log in or refresh, and read only `/v1/metrics/federate`.
* Tokens are time-limited and can be refreshed before expiry.
* Login attempts are rate-limited per client address, and queries per user.
* Access is audit-logged.

### Login

```
POST /v1/auth/login
```

```json
{ "username": "string", "password": "string" }
```

Response:

```json
{ "token": "eyJ...", "expires_in": <seconds>, "kv_map": null }
```

`kv_map` carries the translated partition map for the KV proxy when the proxy is enabled. Send the token as `Authorization: Bearer <token>`.

### Refresh

```
POST /v1/auth/refresh
```

Returns a new token (`token`, `expires_in`, `kv_map`). Requires a valid, unexpired token.

### Change password

```
PUT /v1/auth/password
```

```json
{ "current_password": "string", "new_password": "string" }
```

### Reset password with a reset token

```
POST /v1/auth/reset-password
```

```json
{ "token": "string", "new_password": "string" }
```

The reset token is single-use and is issued by an admin (see User Management).

---

## Query

### Execute a query

```
POST /v1/query
```

Runs a query across NAM's access paths — address, vector and graph — and returns fused results. See [Query Model](QUERY_MODEL.md).

**Request:**

```json
{
  "query": "who assassinated abraham lincoln?",
  "limit": 10,
  "namespace": null,
  "hints": null,
  "fan_out": 3,
  "tenant": null,
  "partition_key": null,
  "paths": null
}
```

| Field | Type | Description |
|---|---|---|
| `query` | string | Natural-language query text (required) |
| `limit` | integer | Maximum results (default 10) |
| `namespace` | string | Query namespace (deployment default otherwise) |
| `hints` | object | Optional planner hints |
| `fan_out` | integer | How far the address path may widen: 0 = point only, 3 = up to a partition-wide scan (default 3) |
| `tenant` | string | Address scope the records were written under; also the vector tenant. Deployment default when unset |
| `partition_key` | string | Partition (scope) within the tenant the records were written under. Deployment default when unset |
| `paths` | array of strings | Pin the access paths: any of `"address"`, `"vector"`, `"graph"`. Omit to let the routing policy choose |

Query parameter `debug=true` adds the address path's execution plan, raw matches and stats when the deployment enables the debug endpoint.

**Response:**

```json
{
  "ok": true,
  "mode": "EXPLORATORY",
  "affordance": null,
  "results": [ { "...": "source record payloads" } ],
  "addresses": [
    {
      "data_pointer": "string",
      "sources": ["address", "vector"],
      "cosine": 0.71,
      "address_meta": { "...": "matched address details" },
      "payload": { "...": "source record" }
    }
  ],
  "truncated": false,
  "errors": [],
  "access_paths": [
    { "path": "address", "status": "used", "reason": "string", "ms": 12.3,
      "stats": {}, "results_contributed": 7 }
  ],
  "partial": false,
  "failed_sources": []
}
```

| Field | Description |
|---|---|
| `ok` | Whether the query executed successfully |
| `mode` | Address-path mode: `EXPLORATORY` or `AFFORDANCE` |
| `affordance` | The detected affordance, if any |
| `results` | Source record payloads of the fused rows, in fused order |
| `addresses` | The fused rows. Each has `data_pointer`, `sources` (which paths found it), `address_meta` (matched key, type and its label and BFO branch, entity, per-axis coarse and fine codes, rotation, match score and geometry; for a row only the vector or graph path found, the address the record is stored under when a bounded lookup finds one, marked `"address_source": "stored"`, and otherwise empty), `cosine` (present exactly when the vector path contributed), and `payload` |
| `truncated` | Whether the address path found more than `limit` |
| `errors` | Non-fatal errors |
| `access_paths` | One trace entry per path: `path`, `status` (`used`, `fallback`, `error`, `skipped`, `not_requested`), `reason`, `ms`, `stats`, `results_contributed`, and when set `unavailable` and `read_failures` |
| `partial` | True when some source's storage reads failed; results are what the healthy sources returned |
| `failed_sources` | Names of the failed sources |
| `degraded` | Present when `partial` is true: `[{source, reason, read_failures}]` |

The `x-nam-server-ms` response header carries server-side latency.

If storage reads failed and no results could be returned, or the query exceeded its time budget, the response is `503` with `Retry-After`, `ok: false`, `error: "data_service_unavailable"`, empty `results` and `addresses`, and the same `degraded`, `failed_sources` and `access_paths` fields.

### Vector search

```
POST /v1/vector/search
```

Runs the vector path alone. Explicitly approximate: it returns the top `limit` records at or above a cosine target. See [Vector Search](VECTOR_SEARCH.md).

**Request:**

| Field | Type | Description |
|---|---|---|
| `query` | string | Query text, embedded by the deployment's model |
| `vector` | array of floats | A raw embedding instead of text |
| `tenant` | string | Vector tenant (deployment default when unset) |
| `cosine_target` | float | Similarity floor. **Clamped** to the deployment's floor: a request cannot ask for records below it |
| `limit` | integer | Maximum results (default 10). `k` is accepted as an alias |
| `resolve` | boolean | Also return the source documents |
| `include_scores` | boolean | Include scores (default true) |

**Response:** `ok`, `results` (each with its `data_pointer` and `cosine`, best first), `cosine_target` (the effective, possibly clamped, target), `limit`, and `documents` with `resolve: true`. The remaining fields report how much of the index the search read; they are diagnostics, not a stable contract. Returns `503` if vector search is not enabled in the deployment, or if the index could not be read (never an empty result in that case).

### Graph walk

```
POST /v1/graph/walk
```

Walks the graph outward from one entity: its own addresses, then the entities that co-occur with it in records, with their addresses. See [Graph Index](GRAPH_INDEX.md).

| Field | Type | Description |
|---|---|---|
| `entity` | string | Seed entity: a 12-hex entity identifier, or a name (normalized and hashed the way the entity head does) |
| `hop_depth` | integer | Hops to walk (deployment default; capped at the deployment maximum) |
| `max_results` | integer | Cap on addresses returned |
| `resolve` | boolean | Also return source documents |

**Response:** `ok`, `seed_keys`, `keys` (addresses reached), `nodes` (each key parsed: `address_key`, `ontology` CURIE, `ontology_label`, `ontology_branch`, `entity_id`, `entity_name`, `entity_type`, per-axis coarse and fine codes, `rotation_label`), `hops_used`, `edges_followed`, `entities_discovered`, `result_count`, `record_entity_map` (record → entities), and `documents` with `resolve: true`.

### Record walk

```
POST /v1/graph/record-walk
```

Walks entity ↔ record edges: records containing the seed entity, then entities co-occurring in those records, hop by hop.

| Field | Type | Description |
|---|---|---|
| `entity` | string | Seed entity (identifier or name) |
| `hop_depth` | integer | Hops (deployment default; capped) |
| `max_records` | integer | Cap on records |

**Response:** `ok`, `records`, `entities` (`[{id, name, type}]`), `seed_entity`, `hops_used`, `record_count`, `entity_count`, `record_entity_map`.

### Entity documents

```
POST /v1/entities/documents
```

```json
{ "entities": ["5d33753699af"], "max_records": 50 }
```

Returns the source records connected to the given entity identifiers: `ok`, `documents` (data pointer → payload), `record_entity_map`, `entity_count`, `document_count`.

### Graph statistics

```
GET /v1/graph/stats
```

Index statistics for the graph bucket.

The graph endpoints return `503` if the graph index is not configured.

---

## Health

```
GET /health
GET /health_check
```

No authentication. Liveness of the query service.

```
GET /ready
```

No authentication. Readiness: `200` once the replica has applied (or tried to apply) its head-slot manifest and, when vector search is enabled, loaded its view of the vector index (with a bounded wait); `503` before that.

```
GET /health/deps
```

Dependency status: whether the data service answered its last probe (`status` is `ok` or `degraded`), recent failures, and cache state. Always `200`; it describes dependencies and is not a probe target.

```
GET /metrics
```

Prometheus metrics. No authentication.

```
GET /v1/metrics/federate
```

Every series the bundled metrics store holds, in Prometheus text format: one scrape target for the whole deployment. Basic auth with a `metrics` or `admin` account; an admin token is accepted too. `401` without valid credentials, `403` for any other role, `429` after repeated failures from one address, `503` when the bundled metrics store is not enabled. Repeated `match[]` selectors narrow the result (default: every job).

---

## Administration

All endpoints below require an `admin` token.

### User management

| Method | Path | Body / Notes |
|---|---|---|
| `POST` | `/v1/admin/users` | `{ "username", "password", "role" }` (`user`, `admin` or `metrics`; defaults to `user`) |
| `GET` | `/v1/admin/users` | List users and roles |
| `PUT` | `/v1/admin/users/{username}` | `{ "role" }` |
| `DELETE` | `/v1/admin/users/{username}` | Remove a user |
| `POST` | `/v1/admin/users/{username}/reset-token` | Returns `{ "ok", "token", "username" }`; the token is used with `/v1/auth/reset-password` |
| `GET` | `/v1/admin/profiles/backup` | All user profiles, encoded, with a count |
| `POST` | `/v1/admin/profiles/restore` | `{ "profiles": { ... } }` from a backup |

### System status and pipeline monitoring

| Method | Path | Returns |
|---|---|---|
| `GET` | `/v1/admin/system-status` | Aggregated health: buckets, pipelines, pods, services |
| `GET` | `/v1/admin/data-nodes` | Per-node health of the data service tier |
| `GET` | `/v1/admin/time-series` | Recent stored time-series samples |
| `GET` | `/v1/admin/encoder-heads` | Active encoder heads and their metrics |
| `GET` | `/v1/admin/head-health` | The latest head-health (collapse) report written by the supervisor's `head_health` task; `?history=N` adds earlier runs |
| `GET` | `/v1/admin/pipeline-errors` | Error counts by stage and pod |
| `GET` | `/v1/admin/dcp-metrics` | Ingest pipeline counters (enqueued, dequeued, errors, item counts) per pipeline. When counters cannot be read the last good reading is served flagged `stale`; with no reading yet, `503`. Optional `history` and `history_window_s` |
| `POST` | `/v1/admin/dcp-metrics/reset-timer` | Resets the load timer used for throughput |
| `GET` | `/v1/admin/storage-stats` | Per-bucket storage statistics (last good snapshot, flagged stale, when the live source is unreachable) |

### Pipeline control

```
POST /v1/admin/reset-checkpoint
```

```json
{ "bucket": "main" }
```

Deletes the ingest pipeline's change-stream checkpoints and zeroes its counters so ingest streams the source bucket again from the beginning. Leases, other consumers' checkpoints and user accounts are untouched, and no bucket is flushed. Running change-stream adapters keep their position in memory, so the reset takes effect when they reconnect.

### Tasks

The supervisor's task framework is reachable through the query service, which checks the caller's admin token and forwards it with the request body.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/admin/tasks` | All task statuses |
| `GET` | `/v1/admin/tasks/{type}` | One task's status |
| `POST` | `/v1/admin/tasks/{type}` | Trigger a task; the JSON body carries per-run options |
| `GET` | `/v1/admin/tasks/{type}/history` | Execution history |
| `POST` | `/v1/admin/tasks/{type}/config` | Runtime override of scheduling, timeout and retry fields |

See [Deployment Architecture](DEPLOYMENT_ARCHITECTURE.md) for the task types.

### Head slots

Each encoder head fills a **slot**; the slot manifest records which implementation fills each one, and ingest and query both apply it. Implementations come from NAM itself or from installed packages such as the open-source `nam-heads`.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/admin/head-slots` | The manifest, each slot's selected and installed implementations, facet slots that can be added, and what this query replica has applied |
| `GET` | `/v1/admin/head-slots/history` | The manifest's change history |
| `PUT` | `/v1/admin/head-slots/{slot}` | `{ "impl", "generation" }`: select a different implementation for a slot |
| `POST` | `/v1/admin/head-slots` | `{ "name", "impl", "generation", "after", "ingest", "query" }`: add a **facet** slot, carried with the record but not part of the address |
| `PUT` | `/v1/admin/head-slots/{slot}/flags` | `{ "generation", "ingest", "query" }`: turn a facet on or off at ingest and at query time |
| `DELETE` | `/v1/admin/head-slots/{slot}?generation=N` | Remove a facet slot |

Every change names the manifest `generation` it was made against and is refused with `409` if the manifest has moved on; an invalid change is `400`. Changes are audit-logged. Only facets can be added, switched off or removed: the anchor and axis slots shape the address itself.

### Document browsing

```
GET /v1/admin/docs/{bucket}?skip=0&limit=50&start_key=
GET /v1/admin/docs/{bucket}/{key}
```

Lists keys (with values) in a bucket, paged, and fetches a single document by key.

### Object storage

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/admin/s3/status` | Current object-storage configuration and data-service disk usage |
| `POST` | `/v1/admin/s3/config` | Sets the query service's object-storage settings. The data service takes its object-storage configuration from the deployment, not from this call |
| `POST` | `/v1/admin/s3/savepoint` | Forces in-memory data to be flushed so it reaches object storage |

### Replication

Endpoints exist to manage cross-cluster replication — remote cluster references (`/v1/admin/xdcr/clusters`), replications (`/v1/admin/xdcr/replications`), replication info and settings (`/v1/admin/xdcr/replication-info`, `/v1/admin/xdcr/settings`). Replication is not configured in the reference deployment.

---

## KV Access

### Partition map

```
GET /v1/kv/map?bucket=main
```

The data service's partition (vbucket) map for an allowed bucket, with addresses translated for access through the KV proxy. `404` when the proxy is not enabled, `403` for a bucket outside the allowlist.

```
GET /v1/kv/session-routing
```

The session store's key-prefix routing table.

### KV proxy

The KV proxy is a TCP service, not HTTP, giving authenticated clients direct key-value access to the data service.

1. The client connects to the proxy port.
2. The client sends its JWT as the first line.
3. The proxy validates the token; on failure it closes the connection.
4. The proxy authenticates upstream on the client's behalf and forwards traffic.

After the handshake only data operations pass. Re-authentication, bucket switching and flush are refused, and change-stream (DCP) access is admin-only.

---

## Errors

Errors use standard HTTP status codes with a JSON body (`detail`, or the structured `503` body for queries):

| Code | Meaning |
|---|---|
| 200 | Success (check `partial` on queries) |
| 400 | Invalid request |
| 401 | Missing or invalid token |
| 403 | Insufficient role, or bucket not allowed |
| 404 | Not found, or feature not enabled |
| 409 | Conflict (for example a task run of that type is still executing, or a head-slot change made against an older manifest generation) |
| 429 | Rate limited |
| 502 | An upstream (supervisor, data service) returned an error |
| 503 | Dependency unavailable, feature not configured, or query could not be answered; may carry `Retry-After` |

---

## Summary

* **Authentication** — JWT, role-scoped, time-limited, rate-limited, audited.
* **Querying** — one endpoint across address, vector and graph paths, with per-row `sources`, an `access_paths` trace, and explicit degraded reporting.
* **Direct paths** — vector search, graph walk, record walk and entity documents.
* **Administration** — users, status, pipeline counters, head health, checkpoints, tasks, head slots, document browsing, object storage.
* **Direct data access** — the partition map and the JWT-authenticated KV proxy.

> See also: [Query Model](QUERY_MODEL.md) | [Vector Search](VECTOR_SEARCH.md) | [Graph Index](GRAPH_INDEX.md) | [High-Level Systems](HIGH_LEVEL_SYSTEMS.md) | [Security and Trust](../GOVERNANCE/SECURITY_AND_TRUST.md)
