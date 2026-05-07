# Cluster, replication, HA, persistence

Topology, failover, durability. Deployment and survival.

## Cluster slot model

16384 slots, CRC16. Multi-key commands only work when all keys hash to the same slot.

Hash tag rule: if a key contains `{...}`, CRC16 is taken over the substring between the first `{` and the next `}`. Otherwise the full key is hashed.

Gotchas:
- Only the **first** `{...}` pair counts. `{a}.{b}` hashes on `a`.
- Empty tag (`{}` or `{}.foo`) is treated as **no tag** - full key is hashed.
- Verify with `CLUSTER KEYSLOT "<key>"`.
- Hot-slot risk: co-locating too many keys under one tag pins that slot to one shard.

Co-location patterns:
- `{user:1000}.profile`, `{user:1000}.cart` -> MGET user data.
- `{order:5678}.header`, `{order:5678}.items` -> atomic transactions.
- `{ratelimit:api}.shard:0` ... `.shard:15` -> MGET sum across shards.
- `{tags}.electronics`, `{tags}.wireless` -> SINTER across tag sets.

## CROSSSLOT

Error: `(error) CROSSSLOT Keys in request don't hash to the same slot`.

Commands requiring same slot (server-enforced via `clusterSlotByCommand` in `src/cluster.c` - generic across commands):
- `MGET`, `MSET`, `MSETNX`
- `SINTER`, `SUNION`, `SDIFF` + their `*STORE` variants
- `ZINTER`, `ZUNION`, `ZDIFF` + their `*STORE` variants
- `LMOVE`, `SMOVE`, `RENAME`, `RENAMENX`
- `EVAL`, `FCALL` with multiple KEYS
- `COPY`

Client-side fan-out (cross-slot "works" because the client splits by slot):

Server always rejects a multi-slot command. Cluster-aware clients (valkey-glide, Redisson, cluster-mode ioredis, ...) group keys by slot and issue one request per slot:
- `DEL`/`UNLINK` with multiple keys.
- `SCAN` - per primary node.
- Single-key commands hit the owning node directly.

Fixes: redesign keys with hash tags; replace multi-key with pipelined single-key calls; rely on cluster-aware client fan-out.

## Redirects

`-MOVED <slot> <host:port>` - slot ownership changed permanently. Client updates slot->node map and retries on the new node.

`-ASK <slot> <host:port>` - key is mid-migration. Client sends `ASKING` + command to target **once**; future requests for the slot still go to the original node until migration completes.

**9.0+ atomic slot migration**: `CLUSTER MIGRATESLOTS` moves slots atomically - **no ASK window**. Legacy `CLUSTER SETSLOT MIGRATING/IMPORTING` still uses ASK during manual resharding. Which one surfaces depends on operator tooling. See valkey-features.md for command surface.

## Read-from-replica

Default: all reads go to the primary that owns the slot.

On the replica connection: `READONLY` to accept reads, `READWRITE` to revert.

Client read-from modes:
- **valkey-glide**: `ReadFrom.Primary | PreferReplica | AZAffinity | AZAffinityReplicasAndPrimary`.
- **ioredis cluster**: `scaleReads: 'master' | 'slave' | 'all' | <fn>`.
- **valkey-py cluster**: `read_from_replicas=True`.

Staleness: sub-ms under normal load; can reach seconds during heavy writes or full resync. Never assume zero lag.

Read-your-writes: `WAIT <numreplicas> <timeout_ms>` - blocks the writer until N replicas ACK (or timeout). Use before the immediate read-back. For AOF durability: `WAITAOF <numlocal> <numreplicas> <timeout_ms>`.

## Pipelining in cluster

Cluster-aware clients group commands by target node and send one pipeline per node in parallel; reassemble in original order. Throughput depends on **per-node batch size**, not total pipeline depth (100 cmds across 3 nodes ≈ 33/node). GLIDE multiplexes internally; explicit pipelining usually not needed.

## SCAN across cluster

`SCAN` iterates a **single node**. To cover the keyspace, loop `CLUSTER NODES` (or the client's primary list) and SCAN each primary until cursor=0.

Gotchas:
- SCAN on replicas works but may miss/dup keys due to replication lag.
- Topology change mid-scan (failover or slot move) may lose or duplicate keys. Pause resharding for critical scans.
- `KEYS *` in cluster mode only hits the one node it was sent to, and blocks it. Never use.

## Pub/Sub in cluster

Regular `PUBLISH` fans out across the cluster bus to every node - scales poorly. Use **sharded pub/sub** (`SSUBSCRIBE` / `SPUBLISH` / `SUNSUBSCRIBE`); channel name is slot-hashed (respects `{tag}`) so messages stay on one shard. See app-patterns.md for full pub/sub details.

## Cluster pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Multi-key without hash tag | `CROSSSLOT` | `{tag}` |
| All data under one tag | One node overloaded | Distribute tags per entity |
| Stale reads after write | Inconsistent read-back | Primary read or `WAIT` |
| SCAN missing keys | Partial list | Iterate every primary |
| Lua touching keys across slots | Rejected or slow | Keys must hash to one slot |
| Regular pub/sub | Bus storm | `SPUBLISH`/`SSUBSCRIBE` |

## Replication internals

### PSYNC2: partial vs full resync

Partial: replica sends last repl ID + offset; if both match and offset is still in primary's backlog, only missing commands stream.

Full-resync triggers:
- First-ever connection.
- Replication ID mismatch (primary restarted/replaced).
- Offset outside backlog (disconnected too long).
- Client-output-buffer-for-replica overflow killed the connection; backlog may have rotated by reconnect.
- Explicit `PSYNC ? -1`.

### Dual replication IDs

After failover, the new primary keeps the old primary's repl ID as a secondary ID. Replicas of the old primary partial-resync to the new primary if their offset is still in the backlog. Why Sentinel failovers usually don't cascade full resyncs.

### Replication backlog

- `repl-backlog-size` default **10 MB** (too small for production).
- `repl-backlog-ttl` default 3600s - retention after last replica disconnects.
- Sizing: `repl-backlog-size >= write_rate_bytes_per_sec * max_disconnect_seconds * 2`.
- Measure write rate: sample `master_repl_offset` from `INFO replication` twice, take delta / seconds.
- Production floor: 256 MB. Write-heavy: 1 GB+.

### Client-output-buffer-for-replica

`client-output-buffer-limit replica 256mb 64mb 60` (default). Buffers primary writes arriving during RDB transfer of a full resync.

Overflow -> connection killed -> replica reconnects -> another full resync -> more buffered data -> **resync loop**, replica never catches up.

Detect via `INFO stats`: `sync_full` climbing while same replica reconnects -> buffer too small. Also `sync_partial_ok`, `sync_partial_err`.

Fixes: raise the limit, enable diskless replication (shorter transfer), or dual-channel (8.0+, eliminates this buffer).

### Diskless replication

`repl-diskless-sync yes` (**Valkey default**; Redis was `no`). Streams RDB from fork memory directly over socket, skipping disk.

- `repl-diskless-sync-delay 5` - wait N s for more replicas; arriving replicas share one RDB stream.
- Replica `repl-diskless-load swapdb` - load RDB while serving old data, atomic swap; needs 2x dataset memory during swap.

Keep disk-based when you need the RDB file for backups, or when replicas connect at very different times.

### Dual-channel replication (8.0+)

See valkey-features.md for setup. Eliminates primary output-buffer-replica overhead during full resync; no resync loop.

### Replica priority / Sentinel selection

`replica-priority` default 100. Lower = higher priority. `0` = never promote (use for backup/analytics replicas).

Sentinel selection order:
1. Lowest `replica-priority` (excluding 0).
2. Most advanced repl offset (least data loss).
3. Smallest run ID (tiebreaker).

### Replica-of-replica chains

Primary -> A -> B reduces primary egress but:
- Each hop adds lag.
- If A full-resyncs from primary, repl ID changes -> all downstream also full-resync.
- Sentinel does not auto-rewire chains when an intermediate fails.

<=3 replicas: direct from primary is simpler.

### min-replicas safety

`min-replicas-to-write 0` (default, disabled). Set N to require N ACKing replicas.
`min-replicas-max-lag 10` - replica counts as lagging if no `REPLCONF ACK <offset>` in this many seconds (replicas ACK 1/s).

Below threshold, writes return `(error) NOREPLICAS Not enough good replicas to write.` - app must catch and retry/fail.

During Sentinel failover the old primary may briefly reject writes - desirable (prevents split-brain).

## High availability (Sentinel)

Client connects to Sentinels (default port 26379), asks for the current primary by **group name** (not hostname). Sentinel and Valkey auth are independent passwords.

Configure multiple Sentinel addresses; never hardcode the primary address - it changes on failover.

Failover timeline (tunable via `down-after-milliseconds`, `failover-timeout`): detection ~down-after-ms; quorum + promotion: seconds. Total outage: 5-30 s.

Replication is async by default: writes acked by the old primary but not yet replicated are lost. Bound with `WAIT` / `WAITAOF`.

### Retry error signals

| Error | Meaning | Action |
|-------|---------|--------|
| `READONLY` | Connected to a replica (post-promotion or stale slot map) | Retry - client rediscovers primary |
| `LOADING` | Server loading dataset from disk post-restart | Retry after delay |
| `ECONNREFUSED` | Node down | Retry with backoff |
| `ECONNRESET` | Connection dropped mid-command | Retry only idempotent commands |
| `CLUSTERDOWN` | Cluster cannot serve (not enough nodes) | Alert ops, long backoff |
| `MASTERDOWN` | Standalone with replica-serves-stale-data-no set | Retry; primary may be failing over |

### Idempotency rule for retries

Safe to retry: `SET`, `GET`, `HSET`, `ZADD` (idempotent under same key+value).

**Not safe** (retrying double-counts or duplicates): `INCR`, `DECR`, `INCRBY`, `DECRBY`, `LPUSH`, `RPUSH`, `XADD` (without explicit ID), `ZINCRBY`. `SADD` is safe (set semantics).

For non-idempotent writes, use an idempotency key (see app-patterns.md Counters section).

## WAIT - in-memory replication

`WAIT <numreplicas> <timeout_ms>` - blocks **only the caller** until N replicas ack all preceding writes on this connection. Returns count acked (0 on timeout).

- `timeout_ms = 0` blocks forever.
- Does **not** make replication globally sync - other clients can still read stale from replicas.
- In a pipeline, applies to all preceding writes; place after last critical write.

## WAITAOF (7.2+) - disk durability

`WAITAOF <numlocal> <numreplicas> <timeout_ms>` - blocks until AOF fsync completes on N local instances (0 or 1) and N replicas. Returns `[local_fsyncs, replica_fsyncs]`.

Stronger than WAIT (in-memory ack only). Required for durability across primary loss.

### When to use

- Read-after-write from replica: `WAIT 1 100`.
- Financial / critical ledger: `WAITAOF 1 1 500`.
- Normal writes / cache: neither. Adds latency on every call.

## Sentinel vs Cluster

- **Sentinel**: single primary, all multi-key commands work, simpler client. Pick when data fits in one node.
- **Cluster**: sharded, only same-slot multi-key, cluster-aware client required. Pick when data size or write throughput exceeds one primary.

## Persistence

### Fsync policies (`appendfsync`)

| Policy | Behavior | Worst-case loss |
|--------|----------|-----------------|
| `everysec` (default) | Background thread fsyncs once/sec | **~2 seconds** (see below) |
| `always` | Fsync in the write path | One command |
| `no` | OS decides | ~30 seconds |

`everysec` 2-second trap: background fsync >1 s (disk contention, AOF rewrite) -> main thread delays new writes up to 1 additional second. Worst case 2 s lost, not 1 s. Signal: `aof-write-pending-fsync` in `LATENCY LATEST`.

`always` throughput: ~1000 writes/sec rotational; SSDs much better. Profile first.

### Hybrid (recommended prod config)

```
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes       # default yes - AOF base is RDB-format
save 3600 1 300 100 60 10000   # default snapshot schedule
```

`aof-use-rdb-preamble` lets startup load an RDB-formatted base then replay tail -> fast restart + high durability.

Startup rule: with both AOF and RDB, Valkey loads the **AOF** (more complete).

### Fork pause (RDB save / AOF rewrite)

~1-2 ms per GB; 64 GB -> 64-128 ms pause during which all clients block. `latest_fork_usec` reports last fork time. Disable THP to prevent COW blowup.

### Copy-on-write during snapshot

Parent + child share pages; each parent write duplicates a page.
- Read-heavy during save: ~0% extra.
- Moderate writes: +10-30%.
- Write-heavy: up to **2x** dataset memory during save.

Plan RAM headroom or the OS will OOM-kill Valkey mid-snapshot.

### Replica-backed durability

`WAIT <N> <timeout_ms>` confirms writes reached N replica **memory** (not disk).
`WAITAOF <local> <replicas> <timeout_ms>` (7.2+) confirms fsync to disk.

Replica AOF + primary AOF multiplies the failure domain required for data loss.

### Cache vs source-of-truth

Pure cache with a durable backing DB: persistence only affects **restart warmup**, not data safety. Tune for fork-pause amortization, not durability.
