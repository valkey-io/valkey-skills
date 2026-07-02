# Throughput, pipelining, and hot keys

Use for lazyfree defaults, SCAN semantics, pipelining and pooling, I/O threading, command selection, hot-key mitigation, pipeline memory prefetch, and reply copy avoidance.

## Valkey 9.0 pipeline memory prefetch

Parser reads multiple commands from query buffer; keys prefetched in batches. `prefetch-batch-max-size` default **16**, max **128**. Also helps MGET/MSET/DEL when pipelined or under I/O threads.

## Valkey 9.0 Reply Copy Avoidance

Skips a payload copy for large bulk-string replies under I/O threads. Internal configs `min-io-threads-avoid-copy-reply`, `min-string-size-avoid-copy-reply`.

Valkey 9.1 improves copy-avoidance memory tracking and I/O-thread coordination. Re-benchmark `io-threads` settings after upgrading; old 9.0 thread counts may no longer be the throughput knee.

## Throughput

### Lazyfree defaults flipped in Valkey 8.0 (were `no` in Redis)

All default to `yes`:
- `lazyfree-lazy-user-del` - `DEL` runs async on background thread.
- `lazyfree-lazy-user-flush` - `FLUSHDB` / `FLUSHALL` async.
- `lazyfree-lazy-eviction` - eviction frees async.
- `lazyfree-lazy-expire` - expiration frees async.
- `lazyfree-lazy-server-del` - implicit deletes (e.g. `RENAME` overwrite) async.

Prefer explicit `UNLINK` in code - intent-visible and unaffected if someone flips `lazyfree-lazy-user-del no`.

### SCAN semantics

`SCAN cursor [MATCH pattern] [COUNT hint] [TYPE t]`

- `COUNT` is a **hint**, not a hard limit; actual returned count may be more or fewer.
- Same key may appear across multiple iterations -> **dedupe client-side**.
- Empty pages are valid; keep iterating until cursor returns `0`.
- Type variants: `HSCAN`, `SSCAN`, `ZSCAN` for inside a single key; `SCAN TYPE <t>` for top-level filter.

### Pipelining

Reduces syscall overhead (one `read()`/`write()` per batch), not just RTT. ~5-10x on loopback; ~10x non-pipelined baseline before plateau.

Batch sweet spot: **~10,000 commands per flush**, read replies, next batch. Larger batches grow the server reply buffer.

Auto-pipelining:
- **ioredis**: `enableAutoPipelining: true` batches within one event-loop tick.
- **valkey-glide**: default via multiplexed connection design.

`MULTI/EXEC` inside a pipeline = atomicity + one round-trip.

Pipeline for independent commands. Lua / FUNCTION when cmd B depends on cmd A's result.

For many independent string writes with the same TTL, Valkey 9.1+ `MSETEX` can replace a SET+EX pipeline:

```
MSETEX 3 k1 v1 k2 v2 k3 v3 EX 60
```

`NX`/`XX` are all-key conditions. Direct Valkey Cluster execution still requires all keys in one slot; clients/proxies that split by shard lose single-command atomicity across slots.

### Connection pooling

Start at `num_cores * 2`. Idle 30-60 s. Connect timeout 2-5 s.

Rules:
- **Dedicated pool for pub/sub** (subscribed connections can't serve regular commands).
- **Dedicated connection for blocking ops** (`BLPOP`, `BRPOP`, `XREAD BLOCK`) - same reason.
- RESP2 client-side caching: dedicate the invalidation connection.
- All-busy -> raise pool size or fix slow commands.

GLIDE: one multiplexed connection per cluster node with auto-pipelining - no pool needed.

### I/O threading

`io-threads N` - **N includes the main thread**. Max 256. Command execution stays single-threaded regardless of N.

Starting points: 2-4 cores -> 2; 6-8 -> 4; 12-16 -> 6-8; 32+ -> 8 and benchmark.

Hidden/legacy knobs:
- `events-per-io-thread` (default 2) - epoll events per cycle before yielding. Raise to amortize more; lower only for tail-latency investigation.
- **`io-threads-do-reads` is silently ignored.** Reads are always on I/O threads when `io-threads > 1`. Safe to leave in valkey.conf for Redis migration.

Useful 9.1+ INFO fields:
- `INFO server`: `io_threads_active`.
- `INFO stats`: `total_reads_processed`, `total_writes_processed`, `io_threaded_reads_processed`, `io_threaded_writes_processed`, `io_threaded_poll_processed`, `io_threaded_total_prefetch_batches`, `io_threaded_total_prefetch_entries`.

Does NOT help: small payloads + few connections; CPU-bound Lua/EVAL; single pipelined client; Unix sockets.

### Command-selection quick table

| Slow | Use |
|------|-----|
| `KEYS <pat>` | `SCAN` + MATCH |
| `DEL <bigkey>` | `UNLINK` (after optional HSCAN/HDEL drain) |
| `HGETALL` (large hash) | `HMGET` known fields or `HSCAN` |
| `SMEMBERS` (large set) | `SSCAN` |
| `SORT` (large) | Pre-sort via ZSET or sort client-side |
| `LRANGE 0 -1` | Paginate explicit ranges |
| Individual `SET` loop | Pipeline, `MSET`, or `MSETEX` when all keys share TTL |

## Hot-key mitigation

- Shard the key: `counters:{0}` ... `counters:{N-1}`; client picks shard by `hash(id) % N`; aggregate at read time.
- Read replicas for read-heavy hot keys (`WAIT` if read-after-write needed).
- `CLIENT TRACKING` for cache-friendly hot reads (see `app-client-side-caching.md`).
