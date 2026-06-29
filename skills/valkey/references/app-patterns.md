# Application pattern routing

Use this as the application-workflow router. Open only the workflow file matching the user task; open adjacent files when the task explicitly crosses workflows.

| User task | Open | Common triggers |
|-----------|------|-----------------|
| Cache-aside, stampede prevention, refresh locks, TTL jitter, explicit invalidation, cache eviction policy | `app-caching.md` | cache-aside, lazy loading, stampede, early refresh, TTL jitter, UNLINK, allkeys-lru, volatile-lru, write-through, write-behind |
| Server-assisted client-side caching | `app-client-side-caching.md` | CLIENT TRACKING, RESP3 push, REDIRECT, `__redis__:invalidate`, BCAST, OPTIN, OPTOUT, tracking table, connection pool |
| Login/session storage | `app-sessions.md` | session hash, sliding TTL, session rotation, HGETEX, per-field TTL, concurrent sessions, session ID entropy |
| Distributed locks and safe release | `app-locks.md` | SET NX PX, DELIFEQ, SET IFEQ, renewal, Redlock, fencing tokens, GC pause, replication unsafe, pre-9.0 Lua fallback |
| Rate limiting | `app-rate-limiting.md` | fixed window, sliding window log, token bucket, HSETEX FNX, HINCRBY, post-increment trap, X-RateLimit-Reset |
| Queues and stream processing | `app-queues.md` | LPUSH/BRPOP, BLMOVE, streams, XREADGROUP, XACK, XPENDING, XAUTOCLAIM, DLQ, priority queue |
| Counters, approximate counting, idempotency, dedup | `app-counters-dedup.md` | INCR, INCRBYFLOAT, sharded counters, idempotency key, HyperLogLog, BITFIELD, SMISMEMBER, Bloom filter |
| Leaderboards | `app-leaderboards.md` | ZADD, ZINCRBY, ZREVRANK, ZRANGE REV WITHSCORES, around-me, ZUNIONSTORE, composite score |
| Pub/Sub and keyspace notifications | `app-pubsub.md` | PUBLISH, SUBSCRIBE, PSUBSCRIBE, SPUBLISH, SSUBSCRIBE, notify-keyspace-events, PUBSUB introspection |
| Search/autocomplete | `app-search.md` | prefix autocomplete, ZRANGE BYLEX, tag filtering, SINTER, SINTERCARD, valkey-search, FT.SEARCH, vector similarity |
