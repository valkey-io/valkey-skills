# Valkey-specific features and version matrix

Version-gated commands, per-release changelog, Redis compatibility, module gap.

## Release history

| Version | Highlights |
|---------|------------|
| 7.2.4 | Fork baseline (March 2024). Wire-compatible with Redis OSS 7.2. |
| 8.0 | I/O threading default-on (~3x: 360K -> 1.2M RPS), dual-channel replication, all `lazyfree-lazy-*` defaults flipped to `yes`, `repl-diskless-sync` default `yes`. |
| 8.1 | `SET IFEQ`, new cache-line hashtable (64 B buckets, 7 entries/bucket, SIMD presence scan, incremental rehash), COMMANDLOG (replaces SLOWLOG), TLS handshake offload to I/O threads (~10% gain), iterator prefetch (~3.5x SCAN/KEYS/HGETALL/HSCAN/SSCAN/ZSCAN), ZRANK ~45% faster on unique scores, BITCOUNT SIMD (~6x @ 1 MB / ~10x @ 10 MB, AVX2/NEON), PFMERGE/PFCOUNT SIMD (~12x multi-HLL dense merge). |
| 9.0 | `DELIFEQ`, hash-field TTL (11 new commands), numbered DBs in cluster mode, atomic slot migration (`CLUSTER MIGRATESLOTS`), `GEOSEARCH BYPOLYGON`, MPTCP replication, Reply Copy Avoidance, additional SIMD (BITCOUNT, HLL/hash `findBucket`), pipeline memory prefetch, un-deprecated 23 commands. RDB magic -> `VALKEY` at version 80+. |

## SET IFEQ (8.1+) - conditional update

```
SET key new_value IFEQ expected_value [EX s | PX ms | EXAT unix-s | PXAT unix-ms | KEEPTTL] [GET]
```

Returns `OK` on match-and-store; `nil` on mismatch **or missing key**.

- **`IFEQ never creates a missing key`**: a CAS retry that treats `nil` as "lost the race" must also handle **deleted-since-read**. Check `EXISTS` after or re-bootstrap via `SET ... NX`.
- **`GET` ambiguity**: returns old value before IFEQ is evaluated. Caller cannot distinguish match+set from mismatch+no-change from the GET reply alone. Use non-GET form for the outcome.
- **Mutually exclusive** with `NX` and `XX` (syntax error if combined).
- **Byte-exact** comparison: `"100"` != `"100 "` (trailing space).
- String-only; non-string key returns `WRONGTYPE`.

## DELIFEQ (9.0+) - conditional delete

```
DELIFEQ key expected_value
```

Returns `1` if matched and deleted; `0` if missing OR mismatch; no change.

String-only; WRONGTYPE on non-string.

Redlock unlock: `0` from any of the N instances is **expected, not a failure** (lock expired or different owner). Continue; don't retry the 0.

## IFEQ/DELIFEQ replication rewrite

`SET ... IFEQ` and `DELIFEQ` are **rewritten to plain `SET` / `DEL`** in the replication stream and AOF. Conditional is evaluated once on the primary.

- Replicas don't re-run the comparison -> deterministic under replay/reconnect.
- AOF grep for `DELIFEQ` / `SET ... IFEQ` misses them - grep `SET` / `DEL`.
- `MONITOR` on primary shows original command; on replica shows the rewritten one.

## Native replacements for Lua

| Pattern | Lua | Native |
|---------|-----|--------|
| CAS | `if call('get',K)==V then call('set',K,V') end` | `SET K V' IFEQ V` (8.1+) |
| Safe lock release | `if call('get',K)==V then call('del',K) end` | `DELIFEQ K V` (9.0+) |

Native: no Lua VM, no script caching, single-key (no hash-tag slot constraints).

## Hash field TTL (9.0+) - 11 new commands

### Setters on existing fields

```
HEXPIRE     key <s>        [NX|XX|GT|LT] FIELDS n field [field...]
HPEXPIRE    key <ms>       [NX|XX|GT|LT] FIELDS n field [field...]
HEXPIREAT   key <unix-s>   [NX|XX|GT|LT] FIELDS n field [field...]
HPEXPIREAT  key <unix-ms>  [NX|XX|GT|LT] FIELDS n field [field...]
```

`NX` = only if no TTL; `XX` = only if TTL; `GT` = only if new > current; `LT` = only if new < current. Mutually exclusive. Per-field.

### Inspection

```
HTTL          key FIELDS n field...   - remaining seconds per field
HPTTL         key FIELDS n field...   - remaining ms per field
HEXPIRETIME   key FIELDS n field...   - absolute Unix s
HPEXPIRETIME  key FIELDS n field...   - absolute Unix ms
```

### Clear TTL

```
HPERSIST key FIELDS n field...
```

### Combined set-and-expire, read-and-refresh

```
HSETEX key [FNX|FXX] [EX s|PX ms|EXAT unix-s|PXAT unix-ms|KEEPTTL]
          FIELDS n field value [field value...]

HGETEX key [EX s|PX ms|EXAT unix-s|PXAT unix-ms|PERSIST]
          FIELDS n field [field...]
```

`FNX`/`FXX` on HSETEX apply to the whole op: FNX requires **none** of the fields exist, FXX requires **all** exist. Condition failure -> **nothing** written (atomic).

`FIELDS n` count must match the number of field names - mismatch = syntax error.

### Return codes

**HEXPIRE/HEXPIREAT/HPEXPIRE/HPEXPIREAT** (array, per field):
- `1` - TTL applied.
- `2` - TTL was 0 or absolute time in the past - **field deleted immediately**.
- `0` - NX/XX/GT/LT condition not met; no change.
- `-2` - field (or hash key) does not exist.
- **`-1` is NOT a setter code** - checking for it is a common mistake; `-1` is an HTTL/HPTTL code.

**HTTL/HPTTL/HEXPIRETIME/HPEXPIRETIME** (array, per field):
- Positive - remaining TTL / absolute expiry.
- `-1` - field exists, no TTL.
- `-2` - field does not exist.

**HPERSIST** (array, per field):
- `1` - TTL removed.
- `-1` - field exists, had no TTL.
- `-2` - field does not exist.

**HSETEX** (scalar): `1` all written; `0` FNX/FXX failed; nothing written.

**HGETEX** (array, same shape as HMGET): values in field order; `nil` for missing or already-expired fields.

### `HEXPIRE key 0` = conditional delete

TTL 0 (or past absolute time) deletes the field immediately and returns `2`. Existence-conditional multi-field delete pattern.

### Visibility semantics

- Lazy + active expiration, same as key-level TTL.
- `HGETALL`, `HSCAN`, `HKEYS`, `HVALS` skip expired fields.
- `HLEN` does **NOT** filter expired fields - physical count until next sweep. Accurate live count: `len(HKEYS)` or sum `HEXISTS`.
- Just-expired window: field may return once, then disappear. Strict aliveness -> `HTTL`.
- Keyspace notifications fire when `notify-keyspace-events` enabled.

### Footguns

1. **`HSET` strips field TTL.** Use `HSETEX ... KEEPTTL FIELDS n field value` for in-place value updates that must preserve TTL.
2. **HEXPIRE is field-level, not key-level.** Set a key `EXPIRE` as safety net - all-persistent fields + no key TTL -> session lives forever.
3. **`HSETEX` creates the key if missing.** "Refresh CSRF token" on a logged-out session silently re-creates it. Gate refresh writes on `EXISTS <key>`.
4. **HINCRBY replication rewrite**: on a hash with volatile fields, `HINCRBY` replicates as `HSETEX ... PXAT ... FIELDS 1 <field> <new>`, not as `HINCRBY`. AOF grep for `HINCRBY` misses it.

### Interaction with key-level TTL

Key `EXPIRE` still works; whichever triggers first wins. Key expiration drops all fields including those with field TTL. `PERSIST key` affects only key TTL - field TTLs unchanged.

### Memory

Only TTL-bearing fields pay metadata cost. Adding TTLs promotes hash to volatile-capable encoding (small per-field overhead). Per-field metadata via an ebuckets structure alongside the hash.

## Key-level TTL reference

| Command | Positive | `-1` | `-2` |
|---------|----------|------|------|
| `TTL key` | seconds remaining | key has no TTL | key missing |
| `PTTL key` | ms remaining | key has no TTL | key missing |
| `EXPIRETIME key` | absolute Unix seconds | key has no TTL | key missing |
| `PEXPIRETIME key` | absolute Unix ms | key has no TTL | key missing |

`PERSIST key` returns `1` if a TTL was removed, `0` for **both** "no TTL" and "key missing" - use `EXISTS` to distinguish.

`EXPIRETIME`/`PEXPIRETIME` (Redis 7.0+) - prefer the absolute form when passing expiration to another service (relative TTL decays in transit).

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

## GEOSEARCH BYPOLYGON (9.0+)

```
GEOSEARCH key
  BYPOLYGON <num-vertices> lon1 lat1 lon2 lat2 ... lonN latN
  [ASC | DESC]
  [COUNT count [ANY]]
  [WITHCOORD] [WITHDIST] [WITHHASH]
```

- `num-vertices >= 3`.
- Polygon is **auto-closed** - do NOT repeat the first vertex.
- Must be **simple** (non-self-intersecting). Winding (CW/CCW) doesn't matter.
- `FROMMEMBER` / `FROMLONLAT` are **rejected** with BYPOLYGON (syntax error); they're mandatory for BYRADIUS/BYBOX.

### WITHDIST quirk

With BYPOLYGON, `WITHDIST` returns distance from the **polygon's computed centroid**, not from any user-supplied point. For distance from a known location, compute client-side or issue a separate `GEODIST`.

### Coordinate storage

- Longitude: -180 to 180.
- Latitude: **-85.05112878 to 85.05112878** (Web Mercator limits).
- Internally a sorted set (score = geohash); sorted-set commands work on geo keys.

### Footguns

- **Antimeridian wrap**: bounding box is min/max lon/lat; a polygon spanning 170° to -170° through the Pacific produces a box covering the whole globe, over-matches. Split into two polygons (east/west of 180°), union client-side.
- **Self-intersecting** polygons produce even-odd fill. Keep simple.
- **Very small polygons** (< 100 m) hit geohash grid precision - verify with `GEODIST`.

### GEOSEARCHSTORE

Supports BYPOLYGON with same geometry, but **rejects `WITHCOORD`, `WITHDIST`, `WITHHASH`** with an error. Store-and-count XOR annotated-results.

### Complexity

`GEOSEARCH` is O(N + log M) where N = elements in the grid-aligned bounding box around the shape, M = matches actually inside. For large geo sets, always pass `COUNT`.

## Numbered DBs in cluster mode (9.0+)

Pre-9.0: only DB 0 in cluster; SELECT errored.

`cluster-databases <N>` enables DBs 0..N-1. Default 1.

- **Immutable at runtime** - set in valkey.conf or cmd line; `CONFIG SET` is rejected; change requires restart.
- Hash slot is computed from the key name alone -> same slot across DBs (but namespaces are independent).

### MOVE reply semantics

`MOVE key db` returns:
- `1` - moved.
- `0` - **ambiguous**: either source key missing OR destination already holds that key. Caller must disambiguate (EXISTS before/after).
- Cluster redirect error if the slot is currently mid-migration on this node.

### Per-node scope

`FLUSHDB`, `SCAN`, `DBSIZE` operate on **this node only** - not cluster-wide. Iterate every primary.

### Isolation caveat

No resource isolation across DBs - shared memory/CPU/connections. ACL granularity is coarse. For real isolation use separate instances or clusters.

## Atomic slot migration (9.0+)

```
CLUSTER MIGRATESLOTS SLOTSRANGE <start> <end> NODE <target-node-id> [SLOTSRANGE ... NODE ...]
CLUSTER GETSLOTMIGRATIONS
CLUSTER CANCELSLOTMIGRATIONS
```

Ranges inclusive. Node-id = 40-hex from `CLUSTER NODES`. Multiple ranges/targets in one call. `CANCELSLOTMIGRATIONS` is a safe no-op after cutover.

### Protocol

1. Source snapshots the slot (AOF-format rewrite internally).
2. Streams snapshot + live writes to target.
3. At catch-up, source briefly pauses writes on that slot (milliseconds).
4. Ownership transfers atomically.
5. Next hit on the old owner gets a single `MOVED`.

**No per-key ASK redirects.** Other slots unaffected.

### `CLUSTER GETSLOTMIGRATIONS` fields

- `operation` - `IMPORT`/`EXPORT`.
- `slot_ranges` - e.g. `"0-4095"`.
- `source_node`, `target_node` - 40-char node IDs.
- `state` - snapshotting / streaming / paused / failover / cleaning up / finished / cancelled / failed.
- `last_update_time` - detect stalls.

### Footgun

**`valkey-cli --cluster reshard` still uses the legacy per-key path** in 9.0 - does NOT call MIGRATESLOTS. Call `CLUSTER MIGRATESLOTS` directly for atomic behavior.

Latency-sensitive: schedule migrations outside peak traffic (ms-scale cutover pause).

## Dual-channel replication (8.0+)

Two TCP connections during full resync - one for RDB, one for live repl stream. Eliminates primary output-buffer-replica overhead -> no resync loop.

- `dual-channel-replication-enabled yes` on replica (default no).
- Primary must have `repl-diskless-sync yes`.
- Negotiated in PSYNC handshake; falls back to single-channel if either side lacks it.

## MPTCP (9.0+)

Kernel **5.6+** required. Both keys are **immutable** (startup-only, not `CONFIG SET`):

```
mptcp yes          # listener accepts MPTCP from clients
repl-mptcp yes     # replica opens MPTCP to primary
```

Startup fails with `MPTCP is not supported on this platform` if kernel lacks it. Enable at runtime: `sysctl -w net.mptcp.enabled=1`.

Clients do **not** need MPTCP support - server MPTCP socket negotiates in the handshake; non-MPTCP clients fall back to standard TCP, no failure.

Benefit: **loss resilience / latency-variance reduction**, not bandwidth. Near-zero delta on clean single-path. Pays off with packet loss + ≥2 reachable paths.

Verify: `ss -M` shows active MPTCP subflows.

## Valkey 9.0 RDB magic change

RDB version **80+** uses `VALKEY` header; earlier versions keep `REDIS`. Both load in Valkey. Redis OSS cannot load version 80+.

## Valkey 9.0 pipeline memory prefetch

Parser reads multiple commands from query buffer; keys prefetched in batches. `prefetch-batch-max-size` default **16**, max **128**. Also helps MGET/MSET/DEL when pipelined or under I/O threads.

## Valkey 9.0 Reply Copy Avoidance

Skips a payload copy for large bulk-string replies under I/O threads. Internal configs `min-io-threads-avoid-copy-reply`, `min-string-size-avoid-copy-reply`.

## Redis compatibility

### Baseline

Compatible with **Redis OSS 2.x through 7.2.x**.
- RESP2 and RESP3 wire protocol identical.
- All Redis 7.2 commands work unchanged.
- Existing Lua scripts work unchanged.
- Module API compatible.
- RDB (pre-9.0 magic) and AOF files from Redis <= 7.2 load directly.
- Port defaults: 6379 client, 26379 Sentinel, cluster bus = port + 10000.

### Incompatible: Redis CE 7.4+

Redis Community Edition **7.4+** (post-license-change) is NOT compatible:
- Proprietary code paths.
- RDB versions in reserved foreign range **12-79**; Valkey rejects these by default.
- Proprietary data formats.

Direct file copy or `REPLICAOF` **will not work**. Use third-party migration tools: **RIOT** or **RedisShake**.

### Identity surfaces that differ

- `valkey-server`, `valkey-cli` binaries.
- `valkey.conf` (same format as `redis.conf`).
- `INFO server` reports `valkey_version` (not `redis_version`).
- Default paths: `/var/lib/valkey`, `/var/log/valkey`, user `valkey`.

### `extended-redis-compatibility yes`

Runtime-modifiable (`CONFIG SET`, no restart). Valkey reports as "redis" with `REDIS_VERSION 7.2.4` in `HELLO`, `INFO`, `LOLWUT`, `CLIENT SETNAME`. Use for clients that string-match the server identity.

### Client libraries

Generic Redis clients work unchanged: ioredis, node-redis, redis-py, Jedis, Lettuce, Redisson, go-redis, rueidis, StackExchange.Redis.

Valkey-specific forks/ports with extra features: **valkey-py**, **valkey-glide** (official, 7 languages).

## Module gap vs Redis 8+ built-ins

Redis 8+ bundles full-text search, vector search, time series, extended probabilistic structures. Valkey splits these into separate modules:

- **valkey-search** - full-text and vector search.
- **valkey-bloom** - Bloom filter, cuckoo filter.
- **valkey-json** - JSON document type.
- **Time series** - no Valkey equivalent today.

Valkey identity: forked from Redis 7.2.4 (March 2024), BSD 3-clause, Linux Foundation.
