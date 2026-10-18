# Graph Index

*Relationships across the address space*

The address index answers "which records live at this region of semantic space?" The **graph index** answers a different question: "starting from this entity or record, what else is connected to it?"

It is the third of NAM's access paths, alongside the deterministic address path and the approximate vector path. Like the others, it is derived entirely from source records and can be rebuilt at any time.

---

## What the Graph Is

In NAM's graph:

* **Records and entities are nodes.**
* **Record-entity edges** say which entities a record mentions, and which records mention an entity.
* **Entity-class edges** say which ontology classes an entity's addresses are stored under.
* **Two entities are connected when they occur in the same record.** A walk follows co-occurrence: from an entity to its records, and from those records to the other entities they mention.

Nothing in the graph is asserted by hand. Every edge follows mechanically from the addresses the pipeline already wrote.

---

## How It Is Stored

The graph lives in its own `graph` bucket, derived from the address store. On the reference deployment it holds the record-entity edges in both directions and the ontology classes each entity is stored under.

There, every graph write is a **blind append**: no write reads first, and nothing is read-modify-write. A redelivered write changes nothing a reader sees, because readers de-duplicate.

In the rotations layout, the graph holds individual key-only records instead: record-entity edges in both directions, an entity record per address, and one record per non-empty axis combination of each address. Enumeration there is a prefix scan.

---

## Who Builds It

On the reference deployment the graph has one writer, the **index builder**, a dedicated change-data-capture consumer of the `nam` (address) bucket:

* It watches address mutations and writes the corresponding graph records.
* It **shards by partition**: with several replicas, replica *N* streams the address partitions whose number modulo the replica count is *N*. The reference deployment runs one shard.
* It checkpoints its progress and **resumes from its checkpoints** after a restart, including a restart of the data service. A transient write failure is retried, never dropped, and the builder reports itself unhealthy when it stalls.
* It runs **off the addressing acknowledgement path**. The addressing pipeline acknowledges a batch once its addresses are durably written; it never waits for the graph.

In the rotations layout the addressing stage writes the graph records itself, through a durable on-pod queue that a separate writer drains, so there too the graph never holds up the address acknowledgement.

The consequence is deliberate: **the graph lags the address index and never blocks it.** A record is queryable through the address path as soon as its addresses land, and through the graph shortly after.

Because it is fully derived, the graph can be flushed and rebuilt from the address bucket without touching source data.

---

## What Uses It

### Graph walk

Starting from a seed entity: take the entity's own addresses, walk co-occurrence to the entities that share records with it, and return their addresses too, so the result carries full coordinates for every entity reached. Hop depth and result counts are bounded.

### Record walk

Starting from an entity: find the records that mention it, then the other entities those records mention, then their records, and so on, hop by hop.

### Entity documents

Given one or more entities, return the records that mention them, with their payloads.

### The graph path in `/v1/query`

When the query's entities resolve (see [Query Model](QUERY_MODEL.md)), the orchestrator runs the graph path seeded from **up to three resolved entities**, walking from each in parallel and merging the results. Records reached from more than one seed come first. The graph path's contributions appear in fused results with `graph` in their `sources`, and its work (hops, edges followed, entities discovered, seeds) appears in the `access_paths` trace.

### Query planning

The address-path planner reads an entity's class records to learn **which ontology classes that entity's addresses are actually stored under**. Those classes are planned first (the STORED tier), before any broader ontology exploration. This lets the planner go straight to the partitions that hold the entity instead of guessing its type from the wording of the question.

---

## Guarantees

The graph index:

* is deterministic — the same addresses produce the same graph records
* is derived — it can be dropped and rebuilt from the address bucket
* never blocks ingest — it trails the address index by design
* uses blind, idempotent writes — a repeated write changes nothing a reader sees, and no write reads first

It does not:

* assert relationships that are not present in the stored addresses
* rank neighbours by importance or similarity
* reflect a record until the index builder has processed its addresses

→ See also: [Addressing Model](ADDRESSING_MODEL.md) | [Query Model](QUERY_MODEL.md) | [Vector Search](VECTOR_SEARCH.md) | [API Reference](API_REFERENCE.md) | [Data Persistence](DATA_PERSISTENCE.md)
