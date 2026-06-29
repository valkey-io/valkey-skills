# Replication, Sentinel, and retry semantics

Use for PSYNC/resync behavior, replication backlog sizing, diskless and dual-channel replication, MPTCP replication transport, Sentinel failover, retryable errors, and idempotency rules.

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

Two TCP connections during full resync - one for RDB, one for live repl stream. Eliminates primary output-buffer-replica overhead -> no resync loop.

- `dual-channel-replication-enabled yes` on replica (default no).
- Primary must have `repl-diskless-sync yes`.
- Negotiated in PSYNC handshake; falls back to single-channel if either side lacks it.

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

### MPTCP (9.0+)

Kernel **5.6+** required. Both keys are **immutable** (startup-only, not `CONFIG SET`):

```
mptcp yes          # listener accepts MPTCP from clients
repl-mptcp yes     # replica opens MPTCP to primary
```

Startup fails with `MPTCP is not supported on this platform` if kernel lacks it. Enable at runtime: `sysctl -w net.mptcp.enabled=1`.

Clients do **not** need MPTCP support - server MPTCP socket negotiates in the handshake; non-MPTCP clients fall back to standard TCP, no failure.

Benefit: **loss resilience / latency-variance reduction**, not bandwidth. Near-zero delta on clean single-path. Pays off with packet loss + ≥2 reachable paths.

Verify: `ss -M` shows active MPTCP subflows.

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

For non-idempotent writes, use an idempotency key (see `app-counters-dedup.md`).

## Sentinel vs Cluster

- **Sentinel**: single primary, all multi-key commands work, simpler client. Pick when data fits in one node.
- **Cluster**: sharded, only same-slot multi-key, cluster-aware client required. Pick when data size or write throughput exceeds one primary.
