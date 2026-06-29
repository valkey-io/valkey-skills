# Latency diagnosis

Use for intrinsic latency, RTT checks, LATENCY monitor, COMMANDLOG during p99 investigation, fork/expiration/AOF/swap signals, I/O-thread queue latency, and slow-command replacements.

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

See `commandlog.md` for full command surface (8.1+, replaces SLOWLOG). Use during latency diagnosis:

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
