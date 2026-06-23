# Cluster, replication, HA, and durability routing

Use this as the cluster/HA router. Split cluster key/client questions from replication, failover, and persistence questions.

| User task | Open | Common triggers |
|-----------|------|-----------------|
| Cluster key design and client behavior | `cluster-key-client-behavior.md` | hash slot, CRC16, hash tag, CROSSSLOT, MOVED, ASK, READONLY, read-from-replica, cluster pipeline, SCAN every primary, sharded pub/sub, numbered DBs |
| Slot migration and resharding | `cluster-slot-migration.md` | CLUSTER MIGRATESLOTS, GETSLOTMIGRATIONS, CANCELSLOTMIGRATIONS, no ASK window, reshard, slot migration state |
| Replication, Sentinel, and retries | `replication-sentinel-retries.md` | PSYNC2, backlog, full resync, diskless replication, dual-channel replication, MPTCP, Sentinel failover, READONLY, LOADING, idempotent retry |
| Durability and persistence | `durability-persistence.md` | WAIT, WAITAOF, appendfsync, AOF, RDB, fork pause, copy-on-write, replica-backed durability, source-of-truth |
