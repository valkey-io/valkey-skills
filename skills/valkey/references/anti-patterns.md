# Anti-patterns

Use when checking production anti-patterns, non-obvious corrections, detection commands, and quick fixes for risky Valkey application usage.

## Non-obvious corrections

- **DEL is already async by default** (Valkey 8.0+): shipped config has `lazyfree-lazy-user-del yes`. `UNLINK` is **intent-visible in code** - a config flip can silently make DEL sync again; UNLINK cannot.
- **SCAN / HSCAN / SSCAN / ZSCAN can return duplicates and empty pages.** Dedupe client-side; iterate until cursor returns `0`.
- `--hotkeys` / `OBJECT FREQ` require `maxmemory-policy` = `*-lfu` (needs access-frequency counter, only tracked under LFU); error otherwise.
- `SLOWLOG` still works in 8.1+ as a legacy alias on the slow log; canonical name is **COMMANDLOG** - same data plus `large-request` / `large-reply` categories that SLOWLOG doesn't expose.

## Listpack encoding thresholds (below -> compact buffer, above -> hashtable/skiplist)

- `hash-max-listpack-entries 512`, `hash-max-listpack-value 64`.
- `set-max-listpack-entries 128`, `set-max-listpack-value 64`.
- `zset-max-listpack-entries 128`, `zset-max-listpack-value 64`.
- `list-max-listpack-size -2` (8 KB per node).

Past threshold, whole-collection commands (`HGETALL`, `SMEMBERS`, unbounded `ZRANGE`, `HKEYS`) become O(N) and scale with collection size.

Sizing: hashes <10K fields; lists <100K elements; sets/zsets <100K members. Split larger collections.

## Detection commands

```
valkey-cli --bigkeys      # largest key per data type
valkey-cli --memkeys      # top keys by memory
valkey-cli --hotkeys      # most-accessed keys (requires *-lfu policy)
valkey-cli CONFIG GET maxmemory
valkey-cli INFO memory
valkey-cli ACL LIST
valkey-cli COMMANDLOG GET 10 slow
```

## High-impact anti-patterns (unique to Valkey)

| Anti-pattern | Fix |
|--------------|-----|
| `KEYS *` in production | `SCAN` with cursor; variants `HSCAN`/`SSCAN`/`ZSCAN`. In cluster, use `CLUSTERSCAN` (9.1+) or scan every primary. |
| Single hot key (counter, rate-limit) | Shard: `counter:0..N`. Use hash tag `counter:{pool}:0` to co-locate shards on one slot for atomic MGET; omit tag only when cross-node spread is required. |
| `WATCH`+`MULTI` for read-then-write | `SET IFEQ` (8.1+) for CAS, `DELIFEQ` (9.0+) for safe release, or FUNCTION for complex atomic logic. |
| Pub/Sub for durable messaging | Streams with consumer groups (at-least-once). |
| Blocking `BLPOP`/`BRPOP`/`XREAD BLOCK` on a pooled connection | Dedicated connection / separate pool for blocking consumers. |
| `FLUSHALL` callable by app creds | Disable via `rename-command FLUSHALL ""` or ACL `-flushall`. |
| Missing TTL on cache entries + no eviction policy | Always `SET ... EX`; set `maxmemory-policy allkeys-lru` or `allkeys-lfu` as safety net. |
| `SORT` on large collections | O(N+M log M) on main thread; pre-sort via ZSET or sort client-side. |
| Unbounded lists / streams | `LTRIM` cap, `XADD ... MAXLEN ~ N` or `XTRIM MINID <ms>`. |
| Storing values > 1 MB | Compress or externalize to object store; keep only the pointer. |
