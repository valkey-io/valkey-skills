# Eviction, TTL, and big-key deletion

Use for TTL return codes, atomic TTL writes, expiration jitter, maxmemory policies, LFU/LRU footguns, and draining large keys before deletion.

## TTL rules

- Set TTL at write time: `SET key val EX 3600` (atomic). `SET` + separate `EXPIRE` leaks keys if `EXPIRE` fails.
- `TTL key` / `PTTL key` returns `-1` (no TTL), `-2` (key missing), otherwise remaining seconds/ms.
- `EXPIRETIME`/`PEXPIRETIME` return absolute Unix expiry.
- Jitter identical TTLs (`EX 3600 + rand(0, 300)`) to avoid expiration storms on the main thread.

## Drain-before-UNLINK for big keys

`UNLINK` on a multi-million-entry hash queues a large background free that still competes with other ops. Drain first:

```
HSCAN bigkey 0 COUNT 100; HDEL bigkey <fields>  # repeat until cursor=0
UNLINK bigkey
```

Same for `SSCAN`+`SREM`, `ZSCAN`+`ZREM`, or `LPOP`/`RPOP` in batches for lists.

## Eviction policies (triggered when `used_memory > maxmemory`)

| Policy | Evicts | Use |
|--------|--------|-----|
| `allkeys-lru` | LRU across all keys | General cache default |
| `allkeys-lfu` | LFU across all keys | Power-law access |
| `volatile-lru` | LRU among keys with TTL | Mixed cache + persistent |
| `volatile-lfu` | LFU among keys with TTL | Mixed + power-law |
| `volatile-ttl` | Shortest remaining TTL first | Priority-by-TTL |
| `volatile-random` / `allkeys-random` | Random | Last resort |
| `noeviction` | Nothing; rejects writes with OOM error | Must-not-lose data |

### Footguns

- `volatile-*` with **no TTL-bearing keys** -> no eviction candidates -> behaves like `noeviction` (writes rejected).
- `maxmemory` unset -> Valkey grows until OS OOM-kills the process. Set to ~75% of available RAM.
- Monitor `evicted_keys` in `INFO stats`: sudden spikes mean working set > memory.
- `OBJECT FREQ` and `--hotkeys` require an LFU policy; `OBJECT IDLETIME` requires LRU (or noeviction).
