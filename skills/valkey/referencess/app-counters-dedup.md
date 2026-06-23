# Counters, approximate counting, and dedup

Use for INCR correctness, float-counter caveats, sharded counters, idempotency claims, HyperLogLog, BITFIELD packed counters, SET/SADD dedup, and Bloom filters.

## Atomic and sharded counters

### INCR family correctness

`INCR / INCRBY / DECR / DECRBY` operate on **int64 signed** (`LLONG_MIN .. LLONG_MAX`). Crossing the boundary returns `"increment or decrement would overflow"` - **no silent wrap**. For long-running counters near int64, rotate windowed keys (`events:<yyyy-mm-dd-HH>` + TTL).

### INCRBYFLOAT footguns

- `long double` accumulation -> drift (`0.1 + 0.2 ≠ 0.3`). **Do not use for money.** Store smallest currency unit (cents, satoshis) as integer and `INCRBY`.
- **Replicates as `SET <key> <final> KEEPTTL`** in replication stream and AOF, not as `INCRBYFLOAT`. AOF grep for `INCRBYFLOAT` misses it.
- Errors only on NaN/Infinity, not on magnitude.

### Windowed counters

`INCR` auto-creates key at 1. `EXPIRE` is separate; a crash between them leaves a permanent key. Use a pipeline:

```
[pipeline]
INCR events:2026-03-29T15
EXPIRE events:2026-03-29T15 7200
```

`EXPIRE` on an existing key resets TTL - acceptable for rolling windows.

### Sharded counters (hot-key bottleneck)

Single key at >10K writes/sec serializes on the same object + replication stream, even on one node.

Two concerns:
- **Avoid hot object on one node**: shard to N keys -> writes parallelize within the node.
- **Distribute across cluster nodes**: drop the shared hash tag -> cross-slot reads need client fan-out.

Default to the first. Shared tag keeps shards on one slot:

```
INCR counter:{pageviews}:7                             # random shard 0..N-1
MGET counter:{pageviews}:0 ... counter:{pageviews}:15  # sum client-side
```

Without the shared tag, `MGET` across shards yields `CROSSSLOT` - group shard keys by slot, issue one pipelined MGET per slot, sum. GLIDE cluster clients do this automatically.

Shard counts: 8-16 for 10-100K writes/sec; 32-64 above that + client batching.

### Idempotency claim

`SET idempotent:<op-id> "processing" NX EX 3600` - returns OK on first claim, nil on duplicate.

On success, replace with result: `SET idempotent:<op-id> <json> XX EX 86400`.

On failure:
- Valkey 9.0+: `DELIFEQ idempotent:<op-id> "processing"` - only deletes if still the placeholder (avoids racing with a successor).
- Pre-9.0: `DEL` (best-effort) or wrap in Lua with GET-then-DEL-if-equal.

**TTL is essential**: a crashed process without TTL leaves a permanent claim blocking all retries.

## Approximate counting and dedup

### HyperLogLog

- 0.81% standard error. Up to ~12 KB per key at full cardinality.
- Sparse encoding at low cardinality (dozens of bytes for ~100 uniques). Auto-promotes to dense when sparse runs out of space.
- Memory: 1M uniques in a SET ~50 MB vs HLL 12 KB.

Commands:
- `PFADD key element [element...]` - returns 1 if HLL estimate changed.
- `PFCOUNT key [key...]` - single or on-the-fly merged count across multiple HLLs (no dest key).
- `PFMERGE dest src1 [src2...]` - materialize merge into `dest`.

**`PFCOUNT` is `READONLY` at command level but `RW` at the key-spec level** - it mutates the HLL header to cache computed cardinality and replicates. Cluster read-routing and ACLs that forbid writes will reject it. Treat as write path.

### BITFIELD packed counters

Fixed-width signed/unsigned integers packed into one string key.

- Type range: `i1..i64`, `u1..u63`.
- **`u64` is not supported** - `BITFIELD key INCRBY u64 #0 1` errors: *"Invalid bitfield type. Note that u64 is not supported but i64 is."* RESP cannot reliably encode unsigned > INT64_MAX. For 64-bit counters use `i64`.

Positional syntax: `#N` = the Nth element of the specified width. `u8 #14` = byte offset `14*8`.

```
BITFIELD stats:page INCRBY u8 #14 1        # hour 14 of 24 hourly counters
BITFIELD stats:page GET u8 #0 GET u8 #1 ...
```

Overflow modes:
- `OVERFLOW WRAP` - default; modular wrap.
- `OVERFLOW SAT` - saturate at type min/max.
- `OVERFLOW FAIL` - return nil on overflow (INCRBY) instead of mutating.

### Dedup patterns

- `SET dedup:<id> 1 NX EX 86400` - exact, per-event key. OK = new, nil = duplicate.
- `SADD processed:batch:<n> id1 id2 ...` then `SMISMEMBER processed:batch:<n> id1 idX id2 ...` - batch membership check; returns array of 0/1.

### Bloom filter (valkey-bloom module)

`BF.RESERVE key <error_rate> <capacity> [EXPANSION n] [NONSCALING]`

- **Default is scaling**: past capacity, new sub-filters are added. The stated error rate applies **only to the first sub-filter**; effective FPR compounds across sub-filters, and memory is unbounded.
- `NONSCALING`: filter rejects new adds past capacity; stated error rate holds for the filter's life.
- Choose `NONSCALING` when the error-rate guarantee matters more than capacity elasticity; default scaling when capacity is uncertain.

Semantics:
- `BF.ADD key item` - 1 if newly added, 0 if probably already present.
- `BF.EXISTS key item` - 0 means **definitely not** in set (no false negatives); 1 means probably in (false-positive rate per the RESERVE).
- Bloom filters do not support TTL on individual items; recreate to reset.
