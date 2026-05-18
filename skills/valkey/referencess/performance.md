# Performance (memory, latency, throughput)

Encoding/eviction, fragmentation/defrag, latency diagnosis, throughput, key sizing.

## Encoding thresholds (compact -> full; conversion is one-way)

| Type | Compact | Threshold | Full | Config keys |
|------|---------|-----------|------|-------------|
| Hash | listpack | entries <= 512 AND each field/value <= 64 B | hashtable | `hash-max-listpack-entries 512`, `hash-max-listpack-value 64` |
| Sorted set | listpack | entries <= 128 AND each member/score <= 64 B | skiplist + hashtable | `zset-max-listpack-entries 128`, `zset-max-listpack-value 64` |
| Set (strings) | listpack | entries <= 128 AND each member <= 64 B | hashtable | `set-max-listpack-entries 128`, `set-max-listpack-value 64` |
| Set (integers) | intset | entries <= 512 | hashtable | `set-max-intset-entries 512` |
| List | quicklist of listpacks (always) | node cap `list-max-listpack-size -2` (8 KB) | - | - |

**Conversion is permanent per key.** Removing elements below threshold does NOT revert encoding. Delete+recreate to restore compact form.

Past threshold, whole-collection commands (`HGETALL`, `SMEMBERS`, unbounded `ZRANGE`, `HKEYS`) become O(N) and scale with collection size.

Inspect: `OBJECT ENCODING key` -> listpack / hashtable / skiplist / intset / quicklist / embstr / int / raw / stream.

## String encoding (not configurable)

| Condition | Encoding | Alloc |
|-----------|----------|-------|
| Value fits a C `long` | `int` | 8 B inline |
| String <= 44 bytes | `embstr` | single alloc (object + data) |
| String > 44 bytes | `raw` | two allocs (object + data) |

44-byte boundary = `OBJ_ENCODING_EMBSTR_SIZE_LIMIT` in `src/object.c`. Stay <= 44 B to avoid the extra pointer indirection.

## Top-level key overhead

~70-80 bytes per top-level key (dictEntry + SDS + object header). Dominates at millions of tiny keys.

### Hash-bucketing (Instagram: 21 GB -> 5 GB, 4x)

Bucket millions of top-level keys into listpack-sized hashes:

```
SET media:1234 <value>                       # original
HSET media:12 34 <value>                     # key = id/100, field = id%100
```

Each bucket stays under ~100 fields -> listpack -> ~5-10x smaller than individual strings.

Same reason: consolidate same-entity fields into one hash (`HSET user:1000 name ... email ...`) rather than separate keys.

## Size limits (rules of thumb)

| Type | Recommended max | Why |
|------|-----------------|-----|
| Hash | < 10K fields | HGETALL latency + encoding flip |
| Set / Sorted set | < 100K members | SMEMBERS / range cost |
| List | < 100K elements | LRANGE cost |
| String value | < 1 MB | Network + memory pressure |

Split larger collections by time bucket or id range (e.g. `user:1000:events:2026-03`).

## TTL rules

- Set TTL at write time: `SET key val EX 3600` (atomic). `SET` + separate `EXPIRE` leaks keys if `EXPIRE` fails.
- `TTL key` / `PTTL key` returns `-1` (no TTL), `-2` (key missing), otherwise remaining seconds/ms.
- `EXPIRETIME`/`PEXPIRETIME` return absolute Unix expiry.
- Jitter identical TTLs (`EX 3600 + rand(0, 300)`) to avoid expiration storms on the main thread.

## OBJECT introspection

| Command | Returns | Requires |
|---------|---------|----------|
| `OBJECT ENCODING key` | listpack / hashtable / skiplist / intset / quicklist / embstr / int / raw | - |
| `OBJECT FREQ key` | LFU access frequency counter | `maxmemory-policy = *-lfu` |
| `OBJECT IDLETIME key` | seconds since last access | `maxmemory-policy = *-lru` (or `noeviction`) |
| `OBJECT REFCOUNT key` | refcount | - |
| `OBJECT HELP` | subcommands | - |

`valkey-cli --hotkeys` also requires an LFU policy (uses `OBJECT FREQ`). `--bigkeys` and `--memkeys` have no policy requirement.

## Drain-before-UNLINK for big keys

`UNLINK` on a multi-million-entry hash queues a large background free that still competes with other ops. Drain first:

```
HSCAN bigkey 0 COUNT 100; HDEL bigkey <fields>  # repeat until cursor=0
UNLINK bigkey
```

Same for `SSCAN`+`SREM`, `ZSCAN`+`ZREM`, or `LPOP`/`RPOP` in batches for lists.

## Eviction policies (triggered when `used_memory > maxmemory`)

| Policy | Evicts | Use |
|--------|--------|-----|
| `allkeys-lru` | LRU across all keys | General cache default |
| `allkeys-lfu` | LFU across all keys | Power-law access |
| `volatile-lru` | LRU among keys with TTL | Mixed cache + persistent |
| `volatile-lfu` | LFU among keys with TTL | Mixed + power-law |
| `volatile-ttl` | Shortest remaining TTL first | Priority-by-TTL |
| `volatile-random` / `allkeys-random` | Random | Last resort |
| `noeviction` | Nothing; rejects writes with OOM error | Must-not-lose data |

### Footguns

- `volatile-*` with **no TTL-bearing keys** -> no eviction candidates -> behaves like `noeviction` (writes rejected).
- `maxmemory` unset -> Valkey grows until OS OOM-kills the process. Set to ~75% of available RAM.
- Monitor `evicted_keys` in `INFO stats`: sudden spikes mean working set > memory.
- `OBJECT FREQ` and `--hotkeys` require an LFU policy; `OBJECT IDLETIME` requires LRU (or noeviction).

## Bitmaps for boolean populations

100M users as bits = 12 MB per key (100e6 / 8 bytes).

```
SETBIT active:2026-03-29 <user_id> 1
GETBIT active:2026-03-29 <user_id>
BITCOUNT active:2026-03-29
BITOP AND active:both active:2026-03-28 active:2026-03-29
```

BITCOUNT SIMD (8.1+): ~6x @ 1 MB / ~10x @ 10 MB on AVX2/NEON.

## Large values

- < 100 KB: fine.
- 100 KB - 1 MB: compress client-side (clients don't auto-compress).
- > 1 MB: externalize to object store, keep only the reference in Valkey.

## Memory fragmentation

### `INFO memory` fields

- `used_memory` - bytes allocated for data.
- `used_memory_rss` - OS-level RSS.
- `mem_fragmentation_ratio` = RSS / used_memory. `mem_fragmentation_bytes` = absolute overhead.
- `allocator_frag_ratio` / `allocator_frag_bytes` - jemalloc internal (allocated vs active). **Defrag fixes this.**
- `allocator_rss_ratio` / `allocator_rss_bytes` - RSS vs jemalloc resident. Kernel has not reclaimed pages yet. **Defrag cannot fix; resolves on its own or on restart.**

### Ratio interpretation

| Ratio | Meaning | Action |
|-------|---------|--------|
| < 1.0 | Swapping. Severe. | Increase RAM or reduce `maxmemory` now. |
| 1.0 - 1.1 | Healthy | - |
| 1.1 - 1.5 | Normal | Monitor |
| 1.5 - 2.0 | Significant | Consider active defrag |
| > 2.0 | Severe | Enable defrag or restart |

Driver: delete/churn workloads leave live allocations scattered, so pages can't return to OS.

### Active defrag config

Default off. All runtime-tunable via `CONFIG SET`.

| Key | Default |
|-----|---------|
| `activedefrag` | `no` |
| `active-defrag-threshold-lower` | 10 (%) - start |
| `active-defrag-threshold-upper` | 100 (%) - full effort |
| `active-defrag-cycle-min` | 1 (% CPU) |
| `active-defrag-cycle-max` | 25 (% CPU) |
| `active-defrag-ignore-bytes` | 104857600 (100 MB) - don't start below this overhead |

CPU scales linearly between lower and upper thresholds. Below either -> no run. Pauses during RDB save / AOF rewrite (avoids inflating COW). Runs on main thread in slices; requires jemalloc.

### `INFO stats` defrag fields

- `active_defrag_running` - current % CPU used (0 when idle).
- `active_defrag_hits` - allocations relocated.
- `active_defrag_misses` - allocations scanned, already optimal.
- `active_defrag_key_hits` / `active_defrag_key_misses` - same per-key.

### Per-key diagnosis

- `MEMORY USAGE key [SAMPLES N]` - bytes incl overhead. Default `SAMPLES 5` (approximate); `SAMPLES 0` = every element (exact, slowest).
- `valkey-cli --bigkeys` - largest key per data type.
- `MEMORY DOCTOR` - auto-checks fragmentation, peak vs current, defrag effectiveness.
- `MEMORY MALLOC-STATS` - jemalloc dump; in bins, high `nslabs` relative to `curregs` = fragmented size class.

### Skip defrag when

- Instance < 1 GB used_memory - overhead negligible.
- Low-churn / stable key population.
- CPU-bound deployments - defrag competes on the main thread.
- Short-lived instances (daily restart resets fragmentation).
- Fragmentation is in `allocator_rss_ratio` (not `allocator_frag_ratio`) - defrag can't help.

### Version notes

- **8.1** new hashtable (64 B buckets, 7 entries/bucket, chain, SIMD presence scan) -> fewer small allocations -> less fragmentation under churn. Many workloads no longer need defrag after 8.1; re-measure first.
- **9.0** Reply Copy Avoidance reduces alloc churn on read-heavy workloads marginally.

## Latency diagnosis

### Baseline tools

- `valkey-cli --intrinsic-latency N` (run on server host) - OS scheduling floor over N seconds. >1 ms = noisy env (VM contention, throttling, NUMA). Bare metal < 100 us.
- `valkey-cli --latency -h host -p 6379` - PING RTT, continuous min/max/avg.
- `valkey-cli --latency-history ...` - 15 s windows.
- `valkey-cli --latency-dist ...` - distribution spectrum.

Network RTT >10x intrinsic -> network, not server.

### LATENCY monitor

Disabled by default (`latency-monitor-threshold` = 0). Enable: `CONFIG SET latency-monitor-threshold 5` (ms). Overhead negligible.

Subcommands: `LATENCY LATEST | HISTORY <event> | GRAPH <event> | DOCTOR | RESET [event...] | HISTOGRAM [cmd...]`.

`LATENCY HISTORY` keeps up to 160 samples/event. `LATENCY HISTOGRAM <cmds...>` = per-command us distribution.

Event types:
- `command` - slow command; cross-reference COMMANDLOG.
- `fast-command` - O(1) cmd exceeded threshold; system-level (not query) issue.
- `fork` - RDB/AOFrewrite fork; ~1-2 ms per GB dataset.
- `expire-cycle` - active expiration burst.
- `active-defrag-cycle` - defrag CPU.
- `aof-fsync-always` - fsync-always blocking main thread.
- `aof-write-pending-fsync` - write delayed by in-flight fsync -> disk contention.

Run `LATENCY DOCTOR` first - auto-checks THP, fork speed, disk contention.

### COMMANDLOG

See valkey-features.md for full command surface (8.1+, replaces SLOWLOG). Use during latency diagnosis:

```
COMMANDLOG GET N slow           # execution time outliers
COMMANDLOG GET N large-request  # payload bytes
COMMANDLOG GET N large-reply    # response bytes
```

Set `CLIENT SETNAME` per service to trace entries back.

### Fork pause sources

`INFO persistence`:
- `latest_fork_usec` - last fork duration.
- `rdb_last_cow_size` - approaches `used_memory` when THP inflates COW.

THP: COW operates on 2 MB pages instead of 4 KB - one-byte write copies 2 MB. Disable: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`.

Fork blocks clients ~1-2 ms/GB; 64 GB -> 64-130 ms pause.

### Expiration storms

Active expiration loops a DB while sampled-stale > `ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE` (10%, tuned by `active-expire-effort`). Identical TTLs -> bursts. Fix: jitter `EX 3600 + rand(0, 300)`.

### AOF / swap

`appendfsync everysec`: background fsync. If fsync >1 s (disk contention), main thread stalls up to 1 additional second; signal = `aof-write-pending-fsync` events.

Swap: 10-100 ms per swapped page. Keep `maxmemory` <= ~75% RAM.

### I/O threads (8.0+)

Default ON. High p99 + normal p50 + clean COMMANDLOG -> queue latency; raise `io-threads`.

### `INFO latencystats`

`eventloop_duration_sum`, `eventloop_duration_cmd_sum`. Cmd dominates -> COMMANDLOG; otherwise loop time on I/O/persistence/periodic tasks.

### `CLIENT LIST`

Flag `b` = blocked; high `omem` = output buffer backing up.

### Slow-command replacements

| Slow | Use |
|------|-----|
| KEYS [pattern] | SCAN cursor MATCH |
| HGETALL (large) | HSCAN / HMGET specific fields |
| SMEMBERS (large) | SSCAN |
| SORT (large) | pre-sort in app or use ZSET |
| DEL (large key) | UNLINK |
| LRANGE 0 -1 | paginate explicit ranges |

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
| Individual `SET` loop | Pipeline or `MSET` |

## Hot-key mitigation

- Shard the key: `counters:{0}` ... `counters:{N-1}`; client picks shard by `hash(id) % N`; aggregate at read time.
- Read replicas for read-heavy hot keys (`WAIT` if read-after-write needed).
- `CLIENT TRACKING` for cache-friendly hot reads (see app-patterns.md).

## valkey-benchmark

```
valkey-benchmark -t SET,GET        # cmd list
                 -c 50              # concurrent connections (default 50)
                 -n 100000          # total requests (default 100000)
                 -P 16              # pipeline depth (default 1)
                 --threads 4        # client-side threads
                 -d 64              # payload size in bytes (default 3)
                 --tls              # enable TLS
                 -q                 # quiet: ops/sec only
```

Pitfalls:
- Default `--threads 1` + server `io-threads 8` -> client is the bottleneck. Match `--threads` to server `io-threads`.
- One connection + deep pipeline saturates the socket buffer. Prefer `-c 100 -P 16` over `-c 1 -P 1000`.
- Warm-up pass first, then measure.
- Bench your real R/W mix, not GET-only.
- Pipeline plateau: linear P1->P16, flattens by P64-128; deeper only adds per-batch latency.

For realistic mixed workloads: `memtier_benchmark --ratio 4:1` (80/20 R/W) validates io-threads / MPTCP / TLS-offload changes better than single-command benches.
