# Rate limiting

Use for fixed windows, sliding logs, token buckets, per-field TTL rate counters, cluster pinning, and rate-limit hygiene.

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
