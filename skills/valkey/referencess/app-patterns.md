# Application patterns

Caching, sessions, locks, rate limiting, queues, counters, leaderboards, pub/sub, search.

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

## CLIENT TRACKING (server-assisted client-side caching)

### Protocol

RESP3: push frames on the same connection. `HELLO 3` + `CLIENT TRACKING ON`. Invalidations arrive as push frames with a key array (nil = full flush).

RESP2: no push. Two-connection REDIRECT:
- Inval conn: `CLIENT ID` -> id; `SUBSCRIBE __redis__:invalidate` (pub/sub mode - dedicate, never reuse for data).
- Data conn: `CLIENT TRACKING ON REDIRECT <id> NOLOOP`.
- Messages on `__redis__:invalidate`: array of keys to evict, or nil -> full local flush.

`__redis__:invalidate` channel name is fixed in Valkey (not renamed).

`NOLOOP`: suppress self-invalidation when the tracking client writes a key it read. Default choice for write-through caches.

### Modes

| Mode | Enable | Server cost | Precision |
|------|--------|-------------|-----------|
| Default | `CLIENT TRACKING ON` | per (key, client) | exact |
| BCAST | `CLIENT TRACKING ON BCAST PREFIX user: PREFIX session:` | per prefix subscription | prefix-wide |
| OPTIN | `CLIENT TRACKING ON OPTIN` + `CLIENT CACHING YES` before each tracked read | per opted key | exact |
| OPTOUT | `CLIENT TRACKING ON OPTOUT` + `CLIENT CACHING NO` before excluded reads | per tracked key | exact |

BCAST is the only mode that scales for high-cardinality keyspaces and is compatible with interchangeable connection pools.

### Tracking table limits

`tracking-table-max-keys` default 1000000; default mode only.

At limit: server evicts a random key and emits a **spurious invalidation** (not an error) - extra misses, not failures.

`INFO stats`:
- `tracking_total_keys` - distinct keys; bounds against `tracking-table-max-keys`.
- `tracking_total_items` - per-(key, client) entries; `items >= keys`.
- `tracking_total_prefixes` - active BCAST prefix subscriptions.

Sizing: `clients * avg_tracked_keys_per_client`. 1000 clients x 5000 keys = 5M -> raise limit or BCAST.

Per-entry ~64-128 B; 1M entries -> ~64-128 MB. BCAST stores only per-client prefix subscriptions.

### Consistency caveats

- Brief stale window between write ack and invalidation arrival.
- No ordering across multi-key writes; invalidations are independent.
- Reconnect drops pending invalidations -> **flush local cache on reconnect**, then re-run setup.
- Primary failover / restart wipes tracking table -> flush on reconnect.
- RESP2 inval connection must stay alive; if it drops, cache silently goes stale. Use `CLIENT NO-EVICT ON` on it.

### Connection-pool pitfall

Default mode is per-connection - interchangeable pools break it. Dedicate one long-lived connection per app instance, or use BCAST.

### Do-not-track

Write-heavy keys; TTL < 1 s; high-cardinality volatile keys; locks / coordination keys.

### Client library support

| Client | Server-assisted tracking |
|--------|--------------------------|
| valkey-glide (7 langs) | yes (CacheConfig) |
| valkey-go | yes |
| redisson (Java) | yes |
| lettuce (Java) | partial |
| redis-py / valkey-py | no built-in; manual RESP2 wiring |
| ioredis / iovalkey | no built-in; manual RESP2 wiring |
| node-redis | no built-in |

## Sessions

### Classic hash sessions

```
HSET session:abc123 user_id 1000 role admin ip "10.0.0.1"
EXPIRE session:abc123 1800

# Sliding TTL on each authenticated request
EXPIRE session:abc123 1800
```

Rotation on privilege escalation (session fixation prevention): HGETALL old -> HSET new + EXPIRE -> UNLINK old. Use MULTI/EXEC or Lua for steps 2-3 atomicity; pipelining alone allows observing intermediate state. In cluster mode, co-locate old+new keys via hash tag: `session:{user:1000}:abc123`, `session:{user:1000}:xyz789`.

### Per-field TTL (9.0+)

See valkey-features.md for full HSETEX/HGETEX/HEXPIRE surface and gotchas. Session use case: different lifetimes per field (csrf_token 5 min, auth_token 30 min, profile_data stable).

### HGETEX: read-and-refresh atomic

Replaces HMGET + HEXPIRE pipeline. Each listed field's TTL resets; unlisted fields untouched. True per-field sliding window.

```
HGETEX session:abc EX 3600 FIELDS 2 user_id email
```

### Concurrent-session tracking

Side index of session IDs per user:

```
SADD user:1000:sessions abc123
EXPIRE user:1000:sessions 86400   # sweep orphans
SCARD user:1000:sessions          # count
SMEMBERS user:1000:sessions       # list
SREM user:1000:sessions abc123    # on destroy
SPOP user:1000:sessions           # evict random for max-N enforcement
```

### Identity hygiene

Session IDs: 128+ bits of crypto randomness. UUIDv4 is 122 bits - acceptable but tight. Never expose internal hash structure in API responses.

## Distributed locks

### Single-instance lock

Acquire: `SET lock:resource <random_value> NX PX <ttl_ms>` (atomic). Random value must be per-acquisition (UUID / crypto random) to prove ownership on release/renew.

Release:
- 9.0+: `DELIFEQ lock:resource <random_value>` - returns 1 if you owned it, 0 otherwise.
- Pre-9.0: Lua `if server.call('GET',K)==V then return server.call('DEL',K) else return 0 end`.

### Renewal / extension

`SET lock:resource <val> IFEQ <val> PX <ttl_ms>` (8.1+) - atomic renew-if-owner.

**IFEQ returns `nil`** in two cases, both meaning **you no longer own this lock**:
1. Value mismatch (another client owns it).
2. **Key missing** - your lock already expired (IFEQ never creates a missing key).

Stop the protected work on `nil`; a worker that ignores `nil` keeps operating on a resource another client now holds.

Pre-8.1: Lua `if server.call('GET',K)==V then return server.call('PEXPIRE',K,T) else return 0 end`.

Auto-renewal: renew at ~2/3 TTL; stop the timer on first `nil`.

### Replication is NOT safe for locks

Async-replication hole:
1. A acquires on primary.
2. Primary crashes before the write replicates.
3. Replica promoted.
4. B acquires the same lock on the new primary - both hold it.

Sentinel failover has the same window. If mutual exclusion must survive a primary crash, use Redlock or a CP coordinator (ZooKeeper, etcd).

### Redlock (N independent primaries, no replication between them)

Typical N=5. Algorithm:

1. T1 = now (ms).
2. `SET key val NX PX ttl` on each instance **sequentially** with a small per-instance timeout (5-50 ms for a 10 s lock) - slow instance trips the timeout, not the whole acquire.
3. Acquired iff successes >= N/2 + 1 AND elapsed (now - T1) < ttl.
4. Effective validity = ttl - elapsed (minus a small clock-drift allowance).
5. On failure: `DELIFEQ` on **all** contacted instances, including ambiguous ones.

Parallel fan-out is an implementation option; sequential is canonical (simpler partial-failure reasoning).

Unlock: `DELIFEQ key <val>` on every contacted instance. **`0` is expected, not an error** - lock expired there or a different owner holds it. Don't retry the `0` instance.

Extend: `SET key val IFEQ val PX new_ttl` (8.1+) on all instances. Extended iff majority succeeds within remaining validity. Bound retries to preserve liveness.

#### Crash-recovery footgun

No persistence: restarted instance has no memory of granted locks -> may grant the same lock to a different client. Either AOF fsync-always (perf hit) or **keep the crashed instance down ≥ max-TTL before rejoining**. AOF everysec loses up to ~2 s on power loss - still unsafe if another client is acquiring in that window.

### When Redlock is overkill

- Idempotent protected op (duplicate is harmless) -> one primary enough.
- Best-effort exclusion OK (rate limits, dedup of mostly-unique work).
- Latency sensitive - Redlock costs N round-trips per acquire.

### Fencing tokens

Defeats GC/network-pause failure: client holds lock, pauses past TTL, another client acquires, original resumes thinking it still holds.

Each acquire returns a **monotonic token**; the protected resource rejects operations with `token < last_seen_token`.

Acquire + INCR must be **atomic in one script** - otherwise acquire-without-bump (failed INCR) or bump-without-acquire (out-of-order tokens):

```
-- KEYS[1]=lock, KEYS[2]=token counter; ARGV[1]=val, ARGV[2]=ttl_ms
if server.call('SET', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
  return server.call('INCR', KEYS[2])
else
  return nil
end
```

Fencing requires the protected resource to validate tokens; APIs without that support cannot use this pattern.

## Rate limiting

### Algorithms summary

| Algorithm | Memory | Accuracy | Burst | Atomicity |
|-----------|--------|----------|-------|-----------|
| Fixed window | 1 key | approx | up to **2x at boundary** | INCR+EXPIRE |
| Sliding window counter | 2 keys | good | smoothed | INCR+EXPIRE |
| Sliding window log | O(N) ZSET entries | exact | none | ZADD+ZREMRANGEBYSCORE+ZCARD, needs MULTI or Lua |
| Token bucket | 1 hash (2 fields) | exact | configured capacity | requires Lua / FUNCTION |
| Per-field TTL (9.0+) | 1 hash (1 field per scope) | approx | 2x boundary | HSETEX FNX + HINCRBY |

### Fixed window

```
k = ratelimit:user:42:<yyyy-mm-ddTHH:MM>
count = INCR k
if count == 1: EXPIRE k <window_s>
if count > limit: reject
```

INCR+EXPIRE race on first request -> on crash key leaks forever. Safer: `SET k 0 NX EX <window>; INCR k` or Lua.

### Sliding window log

```
now_ms = <client>
cutoff = now_ms - window_ms
ZREMRANGEBYSCORE k -inf cutoff
ZCARD k
ZADD k now_ms "<now>-<rand>"      # unique member per request
PEXPIRE k window_ms
```

Wrap in Lua or MULTI/EXEC for atomicity. Timestamp MUST come from the client (not `server.call('TIME')`) for determinism.

### Token bucket (requires Lua)

Hash state: `{tokens, last_refill_ms}`. Refill = `(now - last_refill) * refill_rate`, capped at `capacity`. Lua must be atomic: read state, refill, decide, write back. Under effects replication the replica sees only `HMSET` - deterministic.

### Per-field rate limiting (9.0+)

One hash per user, one field per scope (`/api/orders`, `/api/users`, ...). Field TTL auto-expires the counter.

```
# Create-or-noop with 60s TTL:
HSETEX rate:user:42 FNX EX 60 FIELDS 1 /api/orders 1
# 1 if newly created, 0 if already present.

# If returned 0:
count = HINCRBY rate:user:42 /api/orders 1
if count > limit: reject

# For X-RateLimit-Reset:
HTTL rate:user:42 FIELDS 1 /api/orders
```

Footguns:
- **`HINCRBY` preserves field TTL** - counter ticks down toward original expiry.
- **`HSET` strips field TTL.** Use `HSETEX ... KEEPTTL` for in-place value updates that must keep TTL.
- **Post-increment rejection**: counter already bumped by the time you reject. Fine for access control; **unsafe for billing/SLA metrics** - use token bucket (check-before-consume).
- **Access-refreshed fixed-window trap**: `HGETEX EX 60 ...; HINCRBY ...` (restart TTL on every hit) is **not** sliding - it's a fixed window whose clock resets on use. A steady 1 req/s stream holds the counter open forever. Use sliding-window-log for true sliding.

### Cluster

Pin rate-limit keys with a hash tag matching the limited identity: `{user:42}:ratelimit:...` puts all scopes for one user on one shard.

### Hygiene

- Always set TTL on the rate-limit key (no orphaned counters).
- Limit by user/API key, not IP (shared NAT, spoofing).
- Multi-scope checks (per-user AND per-key) -> pipeline.

## Queues

### List-based

- `LPUSH key job` + `BRPOP key <timeout_s>` - **at-most-once**. Consumer crash loses in-flight job.
- `LPUSH key job` + `BLMOVE src dst LEFT RIGHT <timeout_s>` + `LREM dst 1 job` on success - **at-least-once**; requires a recovery job scanning the processing list for stuck items.

Lists lack: retry counts, dead-letter, priority, scheduling. Use Streams for anything non-trivial.

### Stream-based (XADD / XREADGROUP)

Setup:

```
XGROUP CREATE queue:tasks workers $ MKSTREAM
```

- `$` = start from new messages. Use `0` to replay history.
- `MKSTREAM` creates the stream if missing.
- Duplicate create errors with `BUSYGROUP` - catch and ignore.

Produce:

```
XADD queue:tasks * type email to "u@x" subject "Hi"
# * = auto-generate ID (ms-seq, monotonic)
```

Consume:

```
XREADGROUP GROUP workers <consumer> COUNT 10 BLOCK 5000 STREAMS queue:tasks >
# > = only new messages, never-delivered to this group
# A specific ID replays pending (recovery mode) for this consumer
XACK queue:tasks workers <id>
```

On failure, do NOT ack - message stays in pending list (PEL), reclaimable.

Pending / reclaim:

```
XPENDING queue:tasks workers                        # summary
XPENDING queue:tasks workers - + 10 [consumer]      # detail
XCLAIM queue:tasks workers <new-consumer> <min_idle_ms> <id> [<id>...]
XAUTOCLAIM queue:tasks workers <new-consumer> <min_idle_ms> <cursor> [COUNT N] [JUSTID]
```

`XAUTOCLAIM` uses SCAN-like cursor, returns **three** values: `[next_cursor, claimed_entries, deleted_ids]`.
- `next_cursor == "0-0"` -> sweep complete.
- `deleted_ids` - entries still in PEL but removed from the stream (usually by XDEL); ack them out-of-band to drop them from PEL scans.
- `JUSTID` returns IDs only and **does not increment delivery counter** (inspect without side effects).

Trimming:

```
XTRIM queue:tasks MAXLEN ~ 10000          # approximate, efficient
XTRIM queue:tasks MAXLEN  10000           # exact, slower
XTRIM queue:tasks MINID   ~ <ms>-<seq>    # time-based cutoff
XADD queue:tasks MAXLEN ~ 10000 * ...     # trim inline with produce
```

`~` keeps at-least-N entries, may keep more - server skips full listpack boundaries for much lower cost.

Dead-letter: `XPENDING stream group - + N` exposes per-message delivery count. On threshold: `XADD queue:tasks:dlq * ...` then `XACK` the original.

Multiple consumer groups: independent offsets and PEL per group. One stream serves analytics + processing simultaneously.

### Priority queue via ZSET

```
ZADD queue:priority 1  <job>      # low score = high priority
ZPOPMIN queue:priority            # returns [member, score]
BZPOPMIN queue:priority 30        # blocking, timeout_s; returns [key, member, score]
```

`BZPOPMIN` accepts multiple keys for multi-queue polling; returns the first key that has a member.

### Delivery guarantee summary

| Pattern | Delivery | ACK | Replay | Priority |
|---------|----------|-----|--------|----------|
| LPUSH + BRPOP | at-most-once | - | - | no |
| LPUSH + BLMOVE + LREM | at-least-once | manual LREM | via processing list | no |
| Stream + XREADGROUP + XACK | at-least-once | XACK | full history | no |
| ZADD + BZPOPMIN | at-most-once | - | - | yes |

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

## Leaderboards

ZSET operations, all O(log N):

```
ZADD leaderboard 2500 "player:alice"
ZINCRBY leaderboard 100 "player:alice"
ZREVRANK leaderboard "player:alice"             # 0-indexed top rank
ZRANGE leaderboard 0 9 REV WITHSCORES           # top 10 (preferred since 6.2)
ZREVRANGE leaderboard 0 9 WITHSCORES            # legacy, still supported
ZSCORE leaderboard "player:alice"
```

"Around me" window: fetch `ZREVRANK`, then `ZRANGE leaderboard <rank-k> <rank+k> REV WITHSCORES`.

### Time-bucketed aggregation

`ZUNIONSTORE merged 7 daily:Mon daily:Tue ... AGGREGATE SUM` - combine daily buckets into weekly. Auto-clean with `EXPIRE` on bucket keys.

### Composite score tiebreak

IEEE 754 doubles give ~15 significant digits. Pack primary + tiebreaker in one score:

```
score = points * 10^10 + (MAX_TIMESTAMP - timestamp_seconds)
```

Higher points win; within a tie, earlier timestamp wins (subtract from MAX so earlier has a larger remainder).

### Cluster

A sorted set is one key -> one slot -> one shard. For 100M+ members, shard by score range or by bucket (`leaderboard:{tier:gold}`, `leaderboard:{tier:silver}`) and merge client-side for global views.

## Pub/Sub

Fire-and-forget (at-most-once). Disconnected subscribers miss everything published during the gap. For durable messaging use Streams.

Subscriber connections are **monopolized** - cannot run regular commands. Dedicate a connection or pool.

`PSUBSCRIBE <pattern>` is **O(N)** per publish, matched against all registered patterns across all clients. Prefer exact `SUBSCRIBE` when channel names are known.

Subscriber output buffer default hard limit **32 MB** - slow consumers are disconnected. Tune `client-output-buffer-limit pubsub <hard> <soft> <soft_seconds>`.

### Cluster: sharded pub/sub

Regular `PUBLISH` fans out across the cluster bus to **every node** - wastes bandwidth at scale.

`SPUBLISH`/`SSUBSCRIBE`/`SUNSUBSCRIBE` slot-hash the channel name (respects `{tag}`) - only the node owning the slot is involved.

```
SSUBSCRIBE orders:region:us-east
SPUBLISH   orders:region:us-east '{"order_id":5678}'

# Co-locate related channels with hash tags
SSUBSCRIBE {user:1000}:notifications
SSUBSCRIBE {user:1000}:presence
```

`cluster-allow-pubsubshard-when-down yes` (default): sharded pub/sub keeps serving channels whose slot is covered, even when some cluster slots are not.

### Keyspace notifications

Disabled by default (per-op CPU overhead). Enable: `CONFIG SET notify-keyspace-events Ex`.

Channels:
- `__keyspace@<db>__:<key>` - notifications about this key (payload = event name).
- `__keyevent@<db>__:<event>` - notifications about this event type (payload = key name).

Flags:

| Flag | Meaning |
|------|---------|
| `K` | keyspace channel |
| `E` | keyevent channel |
| `g` | generic: DEL, EXPIRE, RENAME |
| `$` | string commands |
| `l` | list commands |
| `s` | set commands |
| `h` | hash commands |
| `z` | sorted-set commands |
| `t` | stream commands |
| `x` | expired |
| `e` | evicted |
| `m` | key miss (must enable explicitly) |
| `n` | new key creation (must enable explicitly) |
| `A` | alias for `g$lshzxetd` - **excludes `m` and `n`** |

At least one of `K` or `E` must be present alongside event flags.

Cluster: **notifications are local to each node** - subscribe on every primary (via sharded pub/sub) to cover the keyspace.

### PUBSUB introspection

- `PUBSUB CHANNELS [pattern]` - currently subscribed regular channels.
- `PUBSUB NUMSUB [ch...]` - subscriber counts per channel.
- `PUBSUB NUMPAT` - number of clients using PSUBSCRIBE patterns.
- `PUBSUB SHARDCHANNELS [pattern]` / `PUBSUB SHARDNUMSUB` - sharded equivalents.

## Search and autocomplete

### Prefix autocomplete

Store terms with score 0 so sorted-set order is purely lexicographic. Query by prefix with `ZRANGE ... BYLEX` (canonical since 6.2; legacy `ZRANGEBYLEX` still works):

```
ZADD autocomplete 0 "apple"
ZADD autocomplete 0 "application"
ZRANGE autocomplete "[app" "[app\xff" BYLEX LIMIT 0 10
```

Store terms lowercased for case-insensitive matching. For ranked results, keep a separate scored sorted set and join in application code.

### Tag filtering

`SINTER` for AND, `SUNION` for OR. `SINTERCARD` (Redis/Valkey 7.0) returns count without fetching members - useful for "X results" UI counters with an early-stop LIMIT.

Cluster: multi-key set operations require all keys in same slot - use hash tags `{ns}:tag:electronics`.

### When to use valkey-search instead

| Need | Approach |
|------|----------|
| Simple prefix autocomplete | `ZRANGE ... BYLEX` |
| Tag AND/OR filtering | `SINTER` / `SUNION` |
| Full-text, fuzzy, stemming | valkey-search module (`FT.SEARCH`) |
| Relevance scoring with field weights | valkey-search module |
| Vector similarity | valkey-search module |
