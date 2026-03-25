---
title: Redis in System Design - Architecture, Trade-offs, and Failure Modes
date: 2026-03-25 00:00:00 +/-0000
categories: [System Design, Distributed Systems]
tags: [redis, caching, architecture]
---

# Redis in System Design: Architecture, Trade-offs, and Failure Modes

> *A deep dive into Redis internals, caching patterns, cluster mechanics, and production failure modes.*

---

## What is Caching, Precisely?

Caching is the **strategic materialization** of computed or fetched data — reducing latency, offloading downstream systems, and improving throughput under read-heavy workloads.

A cache earns its place only when three conditions hold:

- The cost of recomputation or retrieval is significantly higher than cache access
- Data access exhibits temporal or spatial locality
- Staleness is acceptable within defined bounds

---

## Redis Overview

Redis is an **in-memory, single-threaded** data structure server optimized for low-latency access patterns. Its event-driven architecture (via `epoll`/`kqueue` multiplexing) achieves high throughput without multi-threaded contention on shared data structures.

```
Clients ──► Event Loop ──► Command Processor ──► In-Memory Store
            (epoll/kqueue)   (single thread)       (~100k+ ops/s)
```

### Key Characteristics

- **In-memory storage** — optional persistence via RDB/AOF
- **Single-threaded command execution** — avoids locking, ensures deterministic behavior
- **Data structure abstraction** — not just a key-value store
- **Sub-millisecond latency** — often microseconds
- **~100k+ ops/sec per instance** — workload dependent

### Persistence Trade-offs

Redis provides limited durability guarantees:

| Mode | Trade-off |
|------|-----------|
| **RDB (snapshotting)** | Better performance, potential data loss window |
| **AOF (append-only file)** | Better durability, higher write amplification |

```
Redis (in-memory)
    ├── RDB snapshot  →  performance+ / data loss window
    └── AOF log       →  durable / write amplification

For strict durability + Redis API:
    └── Amazon MemoryDB  →  multi-AZ transactional log + durable Redis API
```

Redis is typically used as:
- A **cache (non-durable)**
- A **performance layer** in front of durable storage

For strict durability with a Redis-compatible API, **Amazon MemoryDB** provides a multi-AZ transactional log.

---

## Advanced Use Cases

Redis is far more than a cache — its rich data structures enable a wide range of coordination and messaging patterns:

```
                    ┌─────────────────────┐
   Caching ─────────►                     ├──────── Leaderboards
   Rate limiting ───►     Redis Core      ├──────── Dist. locking
   Geospatial ──────►                     ├──────── Pub/Sub
   Streams ─────────►                     ├──────── Time Series
                    └─────────────────────┘
```

1. **Caching layer** — read-through / write-through / write-back patterns
2. **Distributed locking** — `SET NX PX` + Redlock (with caveats around safety guarantees)
3. **Leaderboards / ranking systems** — Sorted Sets (`ZSET`) with score-based ordering
4. **Rate limiting** — token bucket / sliding window using atomic ops
5. **Geospatial queries** — radius search via geohash indexing
6. **Event streaming** — Redis Streams (consumer groups, at-least-once delivery)
7. **Pub/Sub** — fire-and-forget messaging (no persistence, no replay)

---

## Core Data Structures

Each Redis data structure uses an internal encoding optimized for memory layout and access speed:

| Data Structure | Internal Encoding | Typical Use Case |
|----------------|-------------------|------------------|
| `String` | SDS (Simple Dynamic String) | Caching, counters, session tokens |
| `Hash` | Ziplist / Hashtable | Object storage, partial updates |
| `List` | Quicklist (linked list + ziplist) | Queues, activity feeds |
| `Set` | Hashtable / Intset | Uniqueness, tag sets |
| `Sorted Set` | Skiplist + Hashtable | Rankings, priority queues |
| `Bloom Filter` | Probabilistic bit arrays | Cache penetration protection |
| `Geo Index` | Sorted Set (geohash encoding) | Proximity search |
| `Time Series` | Module-based | Metrics, observability |

---

## Key Design & Data Distribution

Redis is fundamentally a **key-partitioned system**. Key design directly impacts performance, hotspot avoidance, and cluster correctness.

### Key Design Principles

- **Avoid hot keys** — skewed access patterns create single-node bottlenecks
- **Use namespacing** — `service:entity:id` (e.g. `user:42:profile`)
- **Keep values reasonably sized** — avoid large payloads that inflate memory and network usage

---

## Redis Cluster Internals

Redis partitions its keyspace into **16,384 hash slots**. Each key is assigned to exactly one slot, and each node owns a range of slots.

### Slot Calculation

```
slot = CRC16(key) % 16384
```

### Routing Flow

```
"user:42"
    │
    ▼
CRC16("user:42") % 16384  →  slot 7382
    │
    ▼
Slot map:
    0 – 5460    →  Node 1
    5461 – 10922 →  Node 2  ◄── slot 7382 routes here
    10923 – 16383 →  Node 3
```

### Implications

- All operations for a key are routed to a single node
- Multi-key operations require keys in the same slot (use **hash tags** `{}`)

**Hash tag example** — both keys hash on `123`, guaranteed same slot:

```
user:{123}:profile
user:{123}:orders
```

---

## TTL & Eviction Semantics

TTL is central to cache correctness and memory control. Redis uses two complementary expiration strategies:

### Expiration Behavior

```
Passive expiry (on access)          Active expiry (background)
─────────────────────────           ──────────────────────────
Client GET key                      Background scan
    │                                   │
    ▼                                   ▼
TTL check on access             Random key sampling
    │                                   │
    ├── valid  →  return value          ▼
    │                           Eviction policy
    └── expired →  delete + miss        │
                                        ▼
                                Freed memory
```

### Eviction Policies

| Policy | Behavior |
|--------|----------|
| `volatile-lru` | Evict LRU key among keys with TTL set |
| `allkeys-lru` | Evict LRU key from all keys |
| `volatile-ttl` | Evict key with shortest TTL |
| `allkeys-random` | Evict random key from all keys |

### TTL Design Insight

TTL should reflect:
- **Data freshness requirements** — how stale is acceptable?
- **Recompute cost** — expensive queries warrant longer TTL
- **Traffic patterns** — high QPS keys need careful expiry management

---

## Caching Write Patterns

How you write data to cache is as important as how you read it. The three main strategies differ in consistency, latency, and risk:

### Write-through

```
App  ──►  Cache  ──►  DB
```

- Write goes to cache and DB synchronously
- **Consistent.** Higher write latency.

### Write-back (async)

```
App  ──►  Cache  ──╌╌╌►  DB  (async flush)
```

- Write goes to cache immediately; DB is updated asynchronously
- **Fast.** Risk of data loss on crash.

### Read-through / Cache-aside

```
App  ──►  Cache
            │
         HIT│MISS
            │  └──► DB  ──► populate cache  ──► return
            │
            └──► return value
```

- On cache miss, fetch from DB and populate cache
- Common pattern for read-heavy workloads

---

## Failure Modes at Scale

Production Redis deployments face three classic failure patterns. Understanding them is critical for any staff-level system design discussion.

---

### 1. Cache Penetration

**Problem** — Repeated requests for *non-existent* keys bypass the cache entirely and hammer the database. Common with scrapers or malformed IDs.

```
Problem path:
    Client  ──►  Redis (MISS: key DNE)  ──►  Database (empty query × N)

Mitigations:
    ┌─────────────────┐  ┌──────────────────────┐  ┌────────────────────────┐
    │  Bloom filter   │  │    Cache null         │  │   Input validation     │
    │ pre-check exist.│  │  store null + short   │  │  reject malformed IDs  │
    │  before Redis   │  │  TTL for missing keys │  │     at gateway         │
    └─────────────────┘  └──────────────────────┘  └────────────────────────┘
```

**Mitigations:**
- **Bloom filters** — pre-check key existence before hitting Redis
- **Cache null responses** — store `null` with a short TTL for missing keys
- **Strict input validation** — reject malformed IDs at the API gateway

---

### 2. Cache Breakdown (Hot Key Expiry)

**Problem** — A single high-traffic key expires. All concurrent requests find a cache miss and simultaneously query the database — a thundering herd.

```
Timeline ──────────────────────────────────────────────────►
                     T=0: key expires
                          │
              ┌───────────┴───────────────┐
         Req 1 │  Req 2  │  Req 3  │ ...× N
              └──────────────────────────►  DB overload!
```

**Mitigations:**
- **Request coalescing (mutex / single-flight)** — only one request rebuilds the cache; others wait
- **Logical expiration** — serve stale data while refreshing asynchronously in the background
- **Pre-warming hot keys** — reload before expiry rather than after

---

### 3. Cache Avalanche

**Problem** — Many keys expire at the same time (e.g., after a bulk load with uniform TTL), causing a sudden spike in backend traffic across the board.

```
No jitter (mass expiry):                With TTL jitter:
─────────────────────────               ────────────────────
key A │ key B │ key C │ key D           key A expires at T=0
all expire at T=0                       key B expires at T=+rand
         │                              key C expires at T=+rand
         ▼                                       │
    DB spike 💥                                  ▼
                                        smooth, distributed load ✓
```

**Mitigations:**
- **TTL jitter** — add randomness to expiry: `TTL = base_ttl + rand(0, jitter)`
- **Staggered warming strategies** — don't reload everything at once
- **Multi-layer caching (L1 + L2)** — an in-process L1 absorbs traffic even when Redis keys expire

---

## Operational Considerations

Before deploying Redis in production, ensure you've addressed:

- **Memory fragmentation** — tune `jemalloc` to reduce wasted allocations
- **Persistence overhead vs latency SLAs** — AOF `fsync` every write will cost you
- **Replication lag** — async replicas can fall behind under write pressure
- **Failover behavior** — understand Sentinel election timeouts and Cluster failover mechanics
- **Hot key detection and mitigation** — use `redis-cli --hotkeys` and local shadowing
- **Network bottlenecks** — Redis is often network-bound before it is CPU-bound

---

## When NOT to Use Redis

| Requirement | Better alternative |
|-------------|-------------------|
| Strong consistency | PostgreSQL (serializable), CockroachDB |
| Dataset exceeding memory | DynamoDB, Cassandra, Bigtable |
| Heavy cross-shard transactions | Single-node RDBMS or NewSQL |

---

## Summary

Redis is best understood not as a database, but as a **low-latency, in-memory computation and coordination layer**.

Its effectiveness depends on:

1. **Correct data modeling** — choose the right data structure for each access pattern
2. **Thoughtful key design** — namespacing, hotspot avoidance, hash tags for co-location
3. **Awareness of failure modes** — penetration, breakdown, and avalanche have distinct mitigations
4. **Explicit trade-offs** — between consistency, durability, and performance

> Redis gives you speed by default. Correctness is your responsibility.

---

<p align="center">
  <i>Thanks for reading! If you found this useful or have corrections/additions, feel free to reach out or drop a comment.</i>
</p>
