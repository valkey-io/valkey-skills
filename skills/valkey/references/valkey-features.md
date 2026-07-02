# Valkey-specific features and version matrix

Use this as the quick availability matrix. Open the focused reference file in the final column for task details.

## Release history

| Version | Highlights |
|---------|------------|
| 7.2.4 | Fork baseline (March 2024). Wire-compatible with Redis OSS 7.2. |
| 8.0 | I/O threading default-on, dual-channel replication, `lazyfree-lazy-*` defaults flipped to `yes`, `repl-diskless-sync` default `yes`. |
| 8.1 | `SET IFEQ`, cache-line hashtable, COMMANDLOG, TLS handshake offload, iterator prefetch, ZRANK/BITCOUNT/HLL SIMD improvements. |
| 9.0 | `DELIFEQ`, hash-field TTL, numbered DBs in cluster mode, atomic slot migration, `GEOSEARCH BYPOLYGON`, MPTCP replication, Reply Copy Avoidance, pipeline memory prefetch, RDB magic `VALKEY` at version 80+. |
| 9.0.4 | Security patch release: CVE-2026-23479, CVE-2026-25243, CVE-2026-23631. No app-facing command/API additions. |
| 9.1 | `HGETDEL`, `MSETEX`, `CLUSTERSCAN`, HSETEX key-level `NX`/`XX`, database-level ACLs, TLS SAN-user auth/reload/expiry INFO, `INFO` scripting engines, JSON logs, benchmark warmup/duration/RPS metrics, CLI atomic slot migration flag. |

## Version-gated feature routing

| Feature / question | Version | Open |
|--------------------|---------|------|
| `SET IFEQ`, CAS, byte-exact comparison, GET old-value handling, native Lua replacement | 8.1+ | `conditional-writes.md` |
| `DELIFEQ`, safe conditional delete, safe lock release primitive, replication rewrite | 9.0+ | `conditional-writes.md` and, for lock workflow, `app-locks.md` |
| Hash-field TTL, `HSETEX`, `HGETEX`, `HEXPIRE`, `HTTL`, per-field expiration return codes | 9.0+ | `hash-field-ttl.md` |
| `HGETDEL`, atomic hash field read-and-delete | 9.1+ | `hash-field-ttl.md` |
| `HSETEX NX|XX`, key-level hash existence conditions | 9.1+ | `hash-field-ttl.md` |
| `MSETEX`, multi-key string set with shared TTL / NX / XX / KEEPTTL | 9.1+ | `app-caching.md` and `performance-throughput.md` |
| COMMANDLOG, SLOWLOG compatibility, slow / large request / large reply inspection | 8.1+ | `commandlog.md` |
| `GEOSEARCH BYPOLYGON`, polygon geo queries, antimeridian caveats | 9.0+ | `geosearch-bypolygon.md` |
| Numbered DBs in cluster mode, `cluster-databases`, cluster `MOVE` reply ambiguity | 9.0+ | `cluster-key-client-behavior.md` |
| Database-level ACLs, `db=...`, `alldbs`, `resetdbs` | 9.1+ | `security.md` |
| `CLUSTERSCAN`, cluster-wide key scanning with `SLOT`, `MATCH`, `COUNT`, `TYPE` | 9.1+ | `cluster-key-client-behavior.md` |
| Atomic slot migration, `CLUSTER MIGRATESLOTS`, `GETSLOTMIGRATIONS`, CLI atomic migration flag, no ASK window | 9.0+ / CLI flag 9.1+ | `cluster-slot-migration.md` |
| Dual-channel replication, MPTCP replication transport | 8.0+ / 9.0+ | `replication-sentinel-retries.md` |
| Pipeline memory prefetch, Reply Copy Avoidance, I/O threading behavior and metrics | 9.0+ / 9.1 metrics | `performance-throughput.md` |
| `valkey-benchmark --warmup`, `--duration`, `--rps` histogram | 9.1+ | `performance-benchmarking.md` |
| TLS certificate auto-reload, SAN URI auth, certificate expiry INFO | 9.1+ | `security.md` |
| Cache-line hashtable, BITCOUNT SIMD, compact encoding impact | 8.1+ | `performance-memory-encoding.md` and `performance-fragmentation.md` |
| RDB magic change, Redis OSS/CE migration compatibility, identity surfaces, module gap | 9.0+ / migration | `redis-compatibility.md` |

## Module and compatibility routing

| Question | Open |
|----------|------|
| Redis OSS 2.x-7.2 compatibility, Redis CE 7.4+ incompatibility, RIOT/RedisShake migration | `redis-compatibility.md` |
| `extended-redis-compatibility yes`, clients that string-match Redis identity | `redis-compatibility.md` |
| Valkey module gap vs Redis 8 built-ins, search/Bloom/JSON/time-series availability | `redis-compatibility.md`, then `app-search.md` or `app-counters-dedup.md` |
