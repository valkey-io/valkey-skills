# Durability and persistence

Use for WAIT, WAITAOF, AOF fsync policies, hybrid AOF/RDB setup, fork pauses, copy-on-write memory, replica-backed durability, and cache-vs-source-of-truth choices.

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
