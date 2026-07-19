# Application caching

Use for cache-aside, stampede prevention, early refresh, TTL jitter, invalidation, eviction-policy choices, and write-through/write-behind.

## Caching

### Cache-aside (lazy)

```
val = GET cache:k
if val: return val
val = db.fetch(...)
SET cache:k val EX <ttl>
return val
```

Cold start: first read of each key misses. Stale window: up to TTL.

### Stampede prevention

Lock-based refresh (expensive queries):

```
val = GET cache:k
if val: return val

lock = SET lock:cache:k 1 NX EX 10
if lock:
  val = db.fetch(...)
  SET cache:k val EX <ttl>
  UNLINK lock:cache:k
  return val

sleep(50 ms); retry   # another worker is refreshing
```

Early refresh:

```
remaining = TTL cache:k
if remaining < <threshold>:
  enqueue async refresh job
return cached value
```

Combine: early refresh avoids expiry bursts; lock handles callers crossing the threshold simultaneously.

### TTL jitter

`EX <base> + rand(0, <jitter>)`. Identical TTLs across a namespace all expire in one active-expire burst and stall the main thread.

### Bulk set with one TTL (9.1+)

```
MSETEX 3 cache:a <json> cache:b <json> cache:c <json> EX 300
MSETEX 2 cache:a <json> cache:b <json> NX EX 300
```

`MSETEX n key value [key value...] [NX|XX] [EX s|PX ms|EXAT unix-s|PXAT unix-ms|KEEPTTL]` sets string keys with a shared expiry. `NX`/`XX` are all-key conditions: return `0` and write nothing if any key fails the condition. No expiry option removes existing TTLs; use `KEEPTTL` to preserve them. Repeating the same key is legal - last value wins.

### Explicit invalidation

`UNLINK cache:k` (async free; intent-visible even if `lazyfree-lazy-user-del` is flipped back to `no`).

Pattern-scoped: `CLIENT TRACKING ON BCAST PREFIX cache:` (see CLIENT TRACKING below).

`notify-keyspace-events Exg` is fire-and-forget pub/sub - not durable; never the sole invalidation mechanism.

### Eviction policy trade-off

`allkeys-lru` / `allkeys-lfu` do not need per-key TTLs - dropping TTLs saves an entry in the `expires` table per key. `volatile-*` needs TTL on every cache key to be eligible for eviction.

LFU tuning: `lfu-log-factor` default 10 (lower = faster counter saturation); `lfu-decay-time` default 1 min (lower = faster demotion of no-longer-hot keys).

### Write-through vs write-behind

Write-through: app writes DB then cache. Always fresh; every write pays cache cost.

Write-behind: app writes cache, async batch to DB. Lowest write latency; **crash loses uncommitted writes** - requires AOF + replication planning.
