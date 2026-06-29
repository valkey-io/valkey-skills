---
name: valkey
description: "Use when building apps against Valkey - caching, sessions, queues, locks, rate-limiting, leaderboards, counters, pub/sub, streams, scripting, search, cluster, replication, HA, persistence, security. Not for server internals (valkey-dev) or ops (valkey-ops)."
---

Reference files live under `references/`. Use this file as the router: open the smallest file that matches the user's task, then open adjacent files only when the task crosses a boundary.

## Feature / Version Routing

| User task | Open |
|-----------|------|
| Version availability, "what changed in Valkey 8/9", Redis compatibility, module gap | `referencess/valkey-features.md` |
| CAS, `SET IFEQ`, `DELIFEQ`, conditional delete/update, Lua replacement for compare-and-set/delete | `referencess/conditional-writes.md` |
| Hash-field TTL, `HSETEX`, `HGETEX`, `HEXPIRE`, `HTTL`, per-field expiry return codes | `referencess/hash-field-ttl.md` |
| COMMANDLOG, SLOWLOG compatibility, slow command / large request / large reply lookup | `referencess/commandlog.md` |
| Polygon geo queries, `GEOSEARCH BYPOLYGON`, antimeridian caveats | `referencess/geosearch-bypolygon.md` |
| Redis OSS/CE migration, `extended-redis-compatibility`, RDB magic, client identity, module availability | `referencess/redis-compatibility.md` |

## Application Workflow Routing

| User task | Open |
|-----------|------|
| Cache-aside, stampede prevention, early refresh, TTL jitter, explicit invalidation, cache eviction choices | `referencess/app-caching.md` |
| Server-assisted client-side caching, CLIENT TRACKING, RESP3 push, RESP2 REDIRECT, BCAST/OPTIN/OPTOUT | `referencess/app-client-side-caching.md` |
| Sessions, sliding TTL, session rotation, per-field session TTL, concurrent session tracking | `referencess/app-sessions.md` |
| Distributed locks, safe lock release, renewal, Redlock, fencing tokens, pre-version Lua fallback | `referencess/app-locks.md` plus `referencess/conditional-writes.md` for command semantics |
| Rate limiting, fixed/sliding windows, token bucket, per-field TTL rate counters | `referencess/app-rate-limiting.md` |
| Queues, streams, consumer groups, pending/reclaim, trimming, dead-letter queues, priority queues | `referencess/app-queues.md` |
| Counters, idempotency, approximate counting, dedup, HyperLogLog, BITFIELD, Bloom filters | `referencess/app-counters-dedup.md` |
| Leaderboards and rankings | `referencess/app-leaderboards.md` |
| Pub/Sub, sharded pub/sub, keyspace notifications, PUBSUB introspection | `referencess/app-pubsub.md` |
| Search, autocomplete, tag filtering, valkey-search | `referencess/app-search.md` |

## Cluster / HA Routing

| User task | Open |
|-----------|------|
| Hash slots, hash tags, CROSSSLOT, MOVED/ASK, cluster clients, read replicas, per-primary SCAN, sharded pub/sub, numbered DBs | `referencess/cluster-key-client-behavior.md` |
| Slot migration, resharding, `CLUSTER MIGRATESLOTS`, `GETSLOTMIGRATIONS`, no-ASK-window behavior | `referencess/cluster-slot-migration.md` |
| Replication, full/partial resync, backlog sizing, diskless replication, dual-channel, MPTCP, Sentinel failover, retry/idempotency rules | `referencess/replication-sentinel-retries.md` |
| Durability, `WAIT`, `WAITAOF`, AOF/RDB persistence, fork pauses, copy-on-write, cache-vs-source-of-truth | `referencess/durability-persistence.md` |

## Performance Routing

| Diagnosis area | Open |
|----------------|------|
| Memory encoding, key sizing, compact encodings, top-level overhead, OBJECT introspection, bitmaps, large values | `referencess/performance-memory-encoding.md` |
| TTL, eviction policy, maxmemory, LFU/LRU choices, big-key drain before UNLINK | `referencess/performance-eviction-ttl.md` |
| Fragmentation, active defrag, MEMORY DOCTOR, allocator ratios | `referencess/performance-fragmentation.md` |
| Latency, intrinsic latency, LATENCY monitor, COMMANDLOG, fork/expiration/AOF/swap signals | `referencess/performance-latency.md` |
| Throughput, pipelining, connection pooling, I/O threads, hot-key mitigation | `referencess/performance-throughput.md` |
| Benchmarking, `valkey-benchmark`, `memtier_benchmark`, pipeline/client-thread pitfalls | `referencess/performance-benchmarking.md` |

## Other Focused References

| User task | Open |
|-----------|------|
| Lua, EVAL/EVALSHA, FUNCTIONs, script determinism, server.call/pcall, KILL behavior | `referencess/scripting.md` |
| AUTH, ACLs, ACL categories, key/channel permissions, TLS | `referencess/security.md` |
| Production anti-patterns, quick corrections, detection commands, fix matrix | `referencess/anti-patterns.md` |
