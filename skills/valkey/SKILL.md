---
name: valkey
description: "Use when building apps against Valkey - caching, sessions, queues, locks, rate-limiting, leaderboards, counters, pub/sub, streams, scripting, search, cluster, replication, HA, persistence, security. Not for server internals (valkey-dev) or ops (valkey-ops)."
---

Seven reference files under `referencess/`. Scan the trigger lists, open the matching file.

## referencess/valkey-features.md

Version-gated commands, per-release changelog, Redis compatibility, RDB magic, module gap.

Triggers: `SET IFEQ` (8.1+), `DELIFEQ` (9.0+), CAS, compare-and-swap, safe lock release, replication rewrite to SET/DEL, GET flag ambiguity, WRONGTYPE, byte-exact comparison, IFEQ never creates missing key.

Triggers: hash field TTL, `HSETEX`, `HGETEX`, `HEXPIRE`, `HPEXPIRE`, `HEXPIREAT`, `HPEXPIREAT`, `HTTL`, `HPTTL`, `HEXPIRETIME`, `HPEXPIRETIME`, `HPERSIST`, FNX, FXX, NX/XX/GT/LT on fields, KEEPTTL, setter return codes (1/2/0/-2), HTTL codes (-1/-2), HGETALL filters expired, HLEN does not filter, HINCRBY rewrites to HSETEX.

Triggers: `COMMANDLOG GET/LEN/RESET/HELP` (8.1+), `commandlog-execution-slower-than`, `commandlog-request-larger-than`, `commandlog-reply-larger-than`, slow / large-request / large-reply categories, SLOWLOG legacy alias, REQUEST_POLICY ALL_NODES, RESPONSE_POLICY AGG_SUM.

Triggers: `GEOSEARCH BYPOLYGON` (9.0+), num-vertices, auto-closed polygon, WITHDIST centroid, antimeridian, `GEOSEARCHSTORE`, Web Mercator lat limits.

Triggers: numbered DBs in cluster (9.0+), `cluster-databases`, `MOVE` reply 0 ambiguity, per-node SCAN/FLUSHDB/DBSIZE.

Triggers: atomic slot migration (9.0+), `CLUSTER MIGRATESLOTS`, `CLUSTER GETSLOTMIGRATIONS`, `CLUSTER CANCELSLOTMIGRATIONS`, no ASK window, `valkey-cli --cluster reshard` still legacy.

Triggers: dual-channel replication (8.0+), `dual-channel-replication-enabled`, MPTCP (9.0+), `mptcp`, `repl-mptcp`, kernel 5.6+.

Triggers: pipeline memory prefetch (9.0+), `prefetch-batch-max-size`. Reply Copy Avoidance (9.0+).

Triggers: release history, what's new in 8.0 / 8.1 / 9.0, BITCOUNT SIMD, PFMERGE SIMD, ZRANK speedup, iterator prefetch, cache-line hashtable, TLS handshake offload, lazyfree default flips, repl-diskless-sync default flip, RDB magic VALKEY at version 80+, un-deprecated commands.

Triggers: Redis compatibility, Redis OSS 2.x-7.2.x baseline, Redis CE 7.4+ incompatible, RIOT, RedisShake, `extended-redis-compatibility`, identity surfaces (valkey-server, valkey.conf, INFO server valkey_version).

Triggers: module gap, valkey-search, valkey-bloom, valkey-json, time series (none).

## referencess/app-patterns.md

Application idioms: caching, sessions, locks, rate limiting, queues, counters, leaderboards, pub/sub, search.

Triggers (caching): cache-aside, lazy loading, stampede prevention, refresh lock, early refresh, TTL jitter, explicit invalidation with UNLINK, eviction policy trade-off, `allkeys-lru` vs `volatile-lru` expires-table cost, `lfu-log-factor`, `lfu-decay-time`, write-through, write-behind.

Triggers (CLIENT TRACKING / server-assisted client-side caching): RESP3 push, RESP2 REDIRECT, `__redis__:invalidate`, NOLOOP, BCAST PREFIX, OPTIN, OPTOUT, `tracking-table-max-keys`, spurious invalidation, `tracking_total_keys`/`_items`/`_prefixes`, connection-pool pitfall, client library support matrix.

Triggers (sessions): classic hash session, sliding TTL, session rotation atomicity, privilege escalation, `HGETEX` read-and-refresh atomic, per-field TTL, concurrent-session tracking (SADD/SCARD/SPOP), session ID entropy, UUIDv4 122 bits.

Triggers (locks): `SET NX PX`, per-acquisition random value, `DELIFEQ` release (9.0+), `SET ... IFEQ ... PX` renewal (8.1+), IFEQ nil means no longer own, replication unsafe for locks, Redlock, 5-step algorithm, N=5 independent primaries, sequential vs parallel acquire, Redlock unlock 0 is expected, fencing tokens, monotonic token, GC pause, crash-recovery footgun, AOF fsync-always.

Triggers (rate limiting): fixed window, 2x-at-boundary, sliding window counter, sliding window log, ZADD+ZREMRANGEBYSCORE+ZCARD, token bucket, per-field rate limiting (9.0+), `HSETEX FNX` + `HINCRBY`, post-increment trap, access-refreshed fixed-window trap, `HGETEX EX` is not sliding.

Triggers (queues): LPUSH+BRPOP at-most-once, LPUSH+BLMOVE+LREM at-least-once, streams, `XADD`, `XREADGROUP`, `XACK`, `XPENDING`, `XCLAIM`, `XAUTOCLAIM`, three-value return, `next_cursor 0-0`, `deleted_ids`, `JUSTID`, `XTRIM ~`, `MAXLEN`, `MINID`, dead-letter queue, `BUSYGROUP` error, PEL pending list, multiple consumer groups, `ZPOPMIN`/`BZPOPMIN` priority queue.

Triggers (counters): `INCR`/`INCRBY` int64 overflow error, INCRBYFLOAT drift, money, INCRBYFLOAT replicates as SET+KEEPTTL, windowed counters, pipeline INCR+EXPIRE, hot-key bottleneck, sharded counters with hash tag, MGET across shards, idempotency key, `SET NX EX "processing"`, DELIFEQ release claim.

Triggers (approximate counting): HyperLogLog, `PFADD`/`PFCOUNT`/`PFMERGE`, PFCOUNT is RW, sparse vs dense encoding, BITFIELD, `#N` positional, `u63` max (u64 not supported), OVERFLOW WRAP/SAT/FAIL, dedup with SET NX EX, SMISMEMBER batch check, Bloom filter, `BF.RESERVE`, scaling vs NONSCALING, `BF.ADD`, `BF.EXISTS`, no false negatives.

Triggers (leaderboards): ZSET, `ZADD`, `ZINCRBY`, `ZREVRANK`, `ZRANGE ... REV WITHSCORES` (6.2+), around-me window, `ZUNIONSTORE AGGREGATE SUM`, time-bucketed, composite score tiebreak, IEEE 754 packing.

Triggers (pub/sub): fire-and-forget, at-most-once, subscriber connection monopolization, 32 MB output buffer, `client-output-buffer-limit pubsub`, PSUBSCRIBE O(N), sharded pub/sub, `SPUBLISH`/`SSUBSCRIBE`/`SUNSUBSCRIBE`, `cluster-allow-pubsubshard-when-down`, keyspace notifications, `notify-keyspace-events`, flags `KEA`, `__keyspace@<db>__`, `__keyevent@<db>__`, `PUBSUB CHANNELS/NUMSUB/NUMPAT/SHARDCHANNELS/SHARDNUMSUB`.

Triggers (search / autocomplete): prefix autocomplete, score-0 lex order, `ZRANGE ... BYLEX`, `SINTER`/`SUNION`/`SINTERCARD` (7.0+) tag filtering, valkey-search module, `FT.SEARCH`, vector similarity.

## referencess/performance.md

Memory, latency, throughput - encoding, eviction, fragmentation/defrag, latency diagnosis, pipelining/pooling/io-threads, keys, bitmaps, benchmarks.

Triggers (encoding): listpack, hashtable, skiplist, intset, quicklist, `hash-max-listpack-entries 512`, `hash-max-listpack-value 64`, `zset-max-listpack-entries 128`, `set-max-listpack-entries 128`, `set-max-intset-entries 512`, `list-max-listpack-size -2`, one-way conversion, delete+recreate to restore. String encoding: int, embstr (<=44 B), raw, `OBJ_ENCODING_EMBSTR_SIZE_LIMIT`.

Triggers (top-level key overhead): ~70-80 B per key, hash-bucketing, Instagram 21->5 GB 4x.

Triggers (sizing): hashes <10K fields, sets/zsets/lists <100K, strings <1 MB, split by time bucket or id range.

Triggers (TTL): `SET ... EX` atomic, TTL/PTTL/EXPIRETIME/PEXPIRETIME return codes, -1 / -2, PERSIST ambiguous 0.

Triggers (OBJECT): `OBJECT ENCODING`, `OBJECT FREQ` (requires *-lfu), `OBJECT IDLETIME` (requires *-lru or noeviction), `OBJECT REFCOUNT`, `OBJECT HELP`, `--bigkeys`, `--memkeys`, `--hotkeys` (LFU).

Triggers (big-key drain): HSCAN+HDEL, SSCAN+SREM, ZSCAN+ZREM, LPOP batches before UNLINK.

Triggers (eviction): `maxmemory`, `maxmemory-policy`, `allkeys-lru`, `allkeys-lfu`, `volatile-lru`, `volatile-lfu`, `volatile-ttl`, `volatile-random`, `allkeys-random`, `noeviction`, volatile-* with no TTLs behaves like noeviction, `evicted_keys` in INFO stats.

Triggers (bitmaps): SETBIT/GETBIT/BITCOUNT/BITOP, 100M users ~12 MB, BITCOUNT SIMD (8.1+).

Triggers (fragmentation): `used_memory`, `used_memory_rss`, `mem_fragmentation_ratio`, `allocator_frag_ratio` (defrag fixes), `allocator_rss_ratio` (defrag cannot fix), ratio <1.0 means swap. Active defrag: `activedefrag`, `active-defrag-threshold-lower/upper`, `active-defrag-cycle-min/max`, `active-defrag-ignore-bytes`, `active_defrag_running`/`hits`/`misses`/`key_hits`/`key_misses`. `MEMORY USAGE SAMPLES`, `MEMORY DOCTOR`, `MEMORY MALLOC-STATS` bins. 8.1 hashtable impact.

Triggers (latency): `valkey-cli --intrinsic-latency`, `--latency`, `--latency-history`, `--latency-dist`, `latency-monitor-threshold`, `LATENCY LATEST`, `LATENCY HISTORY`, `LATENCY GRAPH`, `LATENCY DOCTOR`, `LATENCY RESET`, `LATENCY HISTOGRAM`. Event types: `command`, `fast-command`, `fork`, `expire-cycle`, `active-defrag-cycle`, `aof-fsync-always`, `aof-write-pending-fsync`. Fork pause ~1-2 ms/GB, `latest_fork_usec`, `rdb_last_cow_size`, THP disable. Expiration storm, `ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE`, `active-expire-effort`, TTL jitter. `INFO latencystats` `eventloop_duration_sum/_cmd_sum`. `CLIENT LIST` flag `b`, `omem`.

Triggers (throughput): lazyfree 8.0 default flips (`lazyfree-lazy-user-del/flush/eviction/expire/server-del`), UNLINK intent-visible. SCAN COUNT hint, duplicates, dedupe client-side, iterate until 0, TYPE filter. Pipelining syscall reduction, ~10,000 batch sweet spot, `enableAutoPipelining` (ioredis), GLIDE multiplexed. MULTI/EXEC inside pipeline. Connection pooling, dedicated pool for pub/sub and blocking ops. I/O threading, `io-threads` includes main, `io-threads-do-reads` silently ignored, `events-per-io-thread`. Hot-key mitigation via sharding, read-from-replica, CLIENT TRACKING.

Triggers (benchmark): `valkey-benchmark` `-t`/`-c`/`-n`/`-P`/`--threads`/`-d`/`--tls`/`-q`. `memtier_benchmark --ratio`. Pitfalls: client threads must match server io-threads, pipeline plateau P64-128.

## referencess/cluster-and-ha.md

Cluster topology, replication, Sentinel, persistence - deployment and failure survival.

Triggers (slot model): 16384 slots, CRC16, hash tag first `{...}` only, empty tag means no tag, `CLUSTER KEYSLOT`, hot slot risk, co-location patterns.

Triggers (CROSSSLOT): error message, same-slot-required commands list (MGET, MSET, MSETNX, SINTER/SUNION/SDIFF + STORE, ZINTER/ZUNION/ZDIFF + STORE, LMOVE, SMOVE, RENAME, RENAMENX, EVAL/FCALL multi-KEYS, COPY). `clusterSlotByCommand` in `src/cluster.c`. Client-side fan-out: DEL/UNLINK multi-key, SCAN per primary.

Triggers (redirects): MOVED permanent, ASK once-with-ASKING, 9.0+ atomic migration no-ASK-window.

Triggers (read-from-replica): `READONLY`/`READWRITE`, valkey-glide `ReadFrom.Primary|PreferReplica|AZAffinity|AZAffinityReplicasAndPrimary`, ioredis `scaleReads`, valkey-py `read_from_replicas`, staleness, `WAIT` for read-after-write.

Triggers (cluster ops): pipelining per-node batch size, SCAN loop over every primary, topology-change-mid-scan, regular PUBLISH fan-outs across cluster bus, sharded pub/sub.

Triggers (replication internals): PSYNC2, partial vs full resync, full-resync triggers, dual replication IDs, `repl-backlog-size` default 10 MB (too small), `repl-backlog-ttl`, sizing formula (`write_rate * max_disconnect * 2`), `master_repl_offset`. `client-output-buffer-limit replica 256mb 64mb 60`, resync loop, `sync_full`/`sync_partial_ok`/`sync_partial_err`. Diskless: `repl-diskless-sync yes` (Valkey default), `repl-diskless-sync-delay`, `repl-diskless-load swapdb`. Dual-channel (8.0+). `replica-priority` (0 never promote). Sentinel selection order: priority, offset, run-id. Replica chains downsides. `min-replicas-to-write`, `min-replicas-max-lag`, `NOREPLICAS` error.

Triggers (HA / Sentinel): port 26379, group name not hostname, multiple Sentinels, `down-after-milliseconds`, `failover-timeout`, 5-30 s outage. Retry signals: `READONLY`, `LOADING`, `ECONNREFUSED`, `ECONNRESET`, `CLUSTERDOWN`, `MASTERDOWN`. Idempotency rule (INCR/LPUSH/RPUSH/XADD-no-ID/ZINCRBY unsafe; SADD safe).

Triggers (WAIT / WAITAOF): `WAIT <N> <ms>` in-memory, blocks caller only, timeout 0 blocks forever, applies to all preceding writes in connection. `WAITAOF <local> <replicas> <ms>` (7.2+) fsync durability, returns `[local_fsyncs, replica_fsyncs]`.

Triggers (Sentinel vs Cluster): fits-one-node vs sharded choice.

Triggers (persistence): `appendfsync everysec` 2-s trap, `aof-write-pending-fsync`, `always` ~1000/s rotational. Hybrid: `appendonly yes`, `aof-use-rdb-preamble yes` (default, fast restart), `save 3600 1 300 100 60 10000`, AOF loaded if both present. Fork pause ~1-2 ms/GB, `latest_fork_usec`, THP disable. COW during snapshot 0-2x memory. Cache vs source-of-truth.

## referencess/scripting.md

Lua scripting and FUNCTIONs.

Triggers: `EVAL`, `EVALSHA`, `SCRIPT LOAD`, `NOSCRIPT` after restart/failover, `FUNCTION LOAD` persists in RDB/AOF, `FCALL`, `FCALL_RO`, `EVAL_RO` for replicas, ACL `@read`-only, `flags={'no-writes'}`, `busy-reply-threshold` (5000 ms default), `lua-time-limit` legacy alias, `-BUSY`, `SCRIPT KILL`, `FUNCTION KILL`, `SHUTDOWN NOSAVE`, kill works only before first write. Lua memory in `maxmemory`. Native replacements: SET IFEQ, DELIFEQ. Determinism: `server.call('TIME')`/`RANDOMKEY`/`math.random()` avoid in writes. Shebang `#!lua name=<lib>`. `server.register_function` positional vs table form. `FUNCTION LIST/DELETE/DUMP/RESTORE`, `FUNCTION LOAD REPLACE`. `server.call` raises, `server.pcall` single-value with `{err=...}` table on error (not tuple), `server.error_reply`, `server.status_reply`, `redis.*` alias. KEYS[] same-slot rule.

## referencess/security.md

Auth, ACL, TLS.

Triggers (AUTH): `AUTH <pw>`, `AUTH <user> <pw>`, `requirepass`, `ACL SETUSER`.

Triggers (ACL atoms): on/off, `>password`, `<password`, `nopass`, `resetpass`, `+cmd`/`-cmd`, `+@category`/`-@category`, `+cmd|subcommand`, `~pattern`, `%R~pattern` read-only, `%W~pattern` write-only, `allkeys`, `resetkeys`, `&channel-pattern`, `allchannels`, `resetchannels`.

Triggers (categories): `@read`, `@write`, `@string`, `@hash`, `@list`, `@set`, `@sortedset`, `@stream`, `@pubsub`, `@connection`, `@transaction`, `@scripting`, `@admin`, `@dangerous`, `@slow`, `@fast`, `@geo`, `@keyspace`, `@bitmap`, `@hyperloglog`.

Triggers (introspection): `ACL CAT`, `ACL WHOAMI`, `ACL LIST`, `ACL GETUSER`.

Triggers (footgun): ACLs do NOT restrict by DB number.

Triggers (templates): read-write app, reader, cache-only, queue worker, dual read/write surfaces. `-@admin` + `-@dangerous` exclude FLUSHALL/CONFIG/DEBUG/KEYS.

Triggers (connection knobs): `CLIENT SETNAME`, `CLIENT NO-EVICT ON`.

Triggers (TLS): `tls-port` (commonly 6380), CA cert, mTLS cert+key, 6379 stays plaintext unless replaced, **8.1+ TLS handshake offload** to I/O threads.

## referencess/anti-patterns.md

Corrections, detection, fix matrix - "am I doing something stupid" lookup.

Triggers (non-obvious corrections): DEL is already async by default in 8.0+ (`lazyfree-lazy-user-del yes`), UNLINK is intent-visible. SCAN/HSCAN/SSCAN/ZSCAN return duplicates + empty pages. `--hotkeys`/`OBJECT FREQ` require *-lfu. SLOWLOG legacy alias; canonical COMMANDLOG.

Triggers (listpack thresholds): `hash-max-listpack-entries 512`/value 64, set 128/64, zset 128/64, `list-max-listpack-size -2`. Whole-collection O(N) past threshold. Sizing rules.

Triggers (detection): `--bigkeys`, `--memkeys`, `--hotkeys`, `CONFIG GET maxmemory`, `INFO memory`, `ACL LIST`, `COMMANDLOG GET 10 slow`.

Triggers (fix matrix): KEYS in prod, single hot key, WATCH+MULTI replaced by IFEQ/DELIFEQ/FUNCTION, pub/sub for durable, blocking ops on pooled connection, FLUSHALL callable, missing TTL + no eviction, SORT on large, unbounded lists/streams, values >1 MB.
