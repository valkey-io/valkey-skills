# COMMANDLOG and slow/large command inspection

Use for Valkey 8.1+ COMMANDLOG, SLOWLOG compatibility, latency outlier lookup, large request/reply diagnosis, and cluster aggregation behavior.

## COMMANDLOG (8.1+, replaces SLOWLOG)

### Commands

```
COMMANDLOG GET <count> <slow|large-request|large-reply>
COMMANDLOG LEN <slow|large-request|large-reply>
COMMANDLOG RESET <slow|large-request|large-reply>
COMMANDLOG HELP
```

`COMMANDLOG GET` requires explicit count. `SLOWLOG GET` (legacy) defaults count to 10.

### Thresholds / retention (all `CONFIG SET`-able)

| Config | Default | Type |
|--------|---------|------|
| `commandlog-execution-slower-than` | 10000 | microseconds |
| `commandlog-request-larger-than` | 1048576 | bytes |
| `commandlog-reply-larger-than` | 1048576 | bytes |
| `commandlog-slow-execution-max-len` | 128 | entries |
| `commandlog-large-request-max-len` | 128 | entries |
| `commandlog-large-reply-max-len` | 128 | entries |

Threshold `-1` **disables** that log type. `0` logs every command of that type (rarely intended).

### Legacy config aliases

| Legacy | Canonical |
|--------|-----------|
| `slowlog-log-slower-than` | `commandlog-execution-slower-than` |
| `slowlog-max-len` | `commandlog-slow-execution-max-len` |

No legacy alias for `-request-larger-than` or `-reply-larger-than` (new in 8.1). `SLOWLOG GET/LEN/RESET` still work and read the same slow log.

### Entry shape

```
1) id             - unique entry ID
2) timestamp      - Unix seconds
3) duration/size  - us (slow) or bytes (large-*)
4) command_args   - command + args
5) "client_addr"  - IP:port
6) "client_name"  - CLIENT SETNAME value
```

### Cluster

`COMMANDLOG GET/LEN/RESET` carry `REQUEST_POLICY: ALL_NODES`; `LEN` also `RESPONSE_POLICY: AGG_SUM`. Cluster-aware clients (`valkey-cli -c`, smart SDKs) dispatch to every shard and aggregate - don't loop per-node manually.
