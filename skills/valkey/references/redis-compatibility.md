# Redis compatibility and migration boundaries

Use for Redis/Valkey identity differences, RDB/AOF migration compatibility, extended Redis compatibility mode, client-library compatibility, and Redis 8 module gaps.

## Valkey 9.0 RDB magic change

RDB version **80+** uses `VALKEY` header; earlier versions keep `REDIS`. Both load in Valkey. Redis OSS cannot load version 80+.

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
