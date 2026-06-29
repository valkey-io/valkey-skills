# Distributed locks

Use for SET NX PX locks, DELIFEQ safe release, SET IFEQ renewal, pre-version Lua fallbacks, replication caveats, Redlock, and fencing tokens.

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
