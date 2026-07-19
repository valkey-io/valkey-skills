# Cluster key and client behavior

Use for hash slots, hash tags, CROSSSLOT fixes, MOVED/ASK redirects, read-from-replica behavior, cluster pipelining, CLUSTERSCAN, per-primary SCAN, sharded pub/sub, and numbered DBs in cluster mode.

## Cluster slot model

16384 slots, CRC16. Multi-key commands only work when all keys hash to the same slot.

Hash tag rule: if a key contains `{...}`, CRC16 is taken over the substring between the first `{` and the next `}`. Otherwise the full key is hashed.

Gotchas:
- Only the **first** `{...}` pair counts. `{a}.{b}` hashes on `a`.
- Empty tag (`{}` or `{}.foo`) is treated as **no tag** - full key is hashed.
- Verify with `CLUSTER KEYSLOT "<key>"`. Valkey 9.1+ accepts it in standalone mode too; older versions require cluster mode.
- Hot-slot risk: co-locating too many keys under one tag pins that slot to one shard.

Co-location patterns:
- `{user:1000}.profile`, `{user:1000}.cart` -> MGET user data.
- `{order:5678}.header`, `{order:5678}.items` -> atomic transactions.
- `{ratelimit:api}.shard:0` ... `.shard:15` -> MGET sum across shards.
- `{tags}.electronics`, `{tags}.wireless` -> SINTER across tag sets.

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

`FLUSHDB`, `SCAN`, `DBSIZE` operate on **this node only** - not cluster-wide. Use `CLUSTERSCAN` in 9.1+ for key iteration, or iterate every primary on older versions / unsupported clients.

### Isolation caveat

No resource isolation across DBs - shared memory/CPU/connections. Valkey 9.1+ adds database-level ACLs (`db=...`), but that is an access guard only. For real isolation use separate instances or clusters.

## CROSSSLOT

Error: `(error) CROSSSLOT Keys in request don't hash to the same slot`.

Commands requiring same slot (server-enforced via `clusterSlotByCommand` in `src/cluster.c` - generic across commands):
- `MGET`, `MSET`, `MSETNX`, `MSETEX`
- `SINTER`, `SUNION`, `SDIFF` + their `*STORE` variants
- `ZINTER`, `ZUNION`, `ZDIFF` + their `*STORE` variants
- `LMOVE`, `SMOVE`, `RENAME`, `RENAMENX`
- `EVAL`, `FCALL` with multiple KEYS
- `COPY`

Client-side fan-out (cross-slot "works" because the client splits by slot):

Server always rejects a multi-slot command. Cluster-aware clients (valkey-glide, Redisson, cluster-mode ioredis, ...) group keys by slot and issue one request per slot:
- `DEL`/`UNLINK` with multiple keys.
- `SCAN` - per primary node.
- Single-key commands hit the owning node directly.

Fixes: redesign keys with hash tags; replace multi-key with pipelined single-key calls; rely on cluster-aware client fan-out.

## Redirects

`-MOVED <slot> <host:port>` - slot ownership changed permanently. Client updates slot->node map and retries on the new node.

`-ASK <slot> <host:port>` - key is mid-migration. Client sends `ASKING` + command to target **once**; future requests for the slot still go to the original node until migration completes.

**9.0+ atomic slot migration**: `CLUSTER MIGRATESLOTS` moves slots atomically - **no ASK window**. Legacy `CLUSTER SETSLOT MIGRATING/IMPORTING` still uses ASK during manual resharding. Which one surfaces depends on operator tooling. See `cluster-slot-migration.md` for command surface.

## Read-from-replica

Default: all reads go to the primary that owns the slot.

On the replica connection: `READONLY` to accept reads, `READWRITE` to revert.

Client read-from modes:
- **valkey-glide**: `ReadFrom.Primary | PreferReplica | AZAffinity | AZAffinityReplicasAndPrimary`.
- **ioredis cluster**: `scaleReads: 'master' | 'slave' | 'all' | <fn>`.
- **valkey-py cluster**: `read_from_replicas=True`.

Staleness: sub-ms under normal load; can reach seconds during heavy writes or full resync. Never assume zero lag.

Read-your-writes: `WAIT <numreplicas> <timeout_ms>` - blocks the writer until N replicas ACK (or timeout). Use before the immediate read-back. For AOF durability: `WAITAOF <numlocal> <numreplicas> <timeout_ms>`.

## Pipelining in cluster

Cluster-aware clients group commands by target node and send one pipeline per node in parallel; reassemble in original order. Throughput depends on **per-node batch size**, not total pipeline depth (100 cmds across 3 nodes ≈ 33/node). GLIDE multiplexes internally; explicit pipelining usually not needed.

## CLUSTERSCAN / SCAN across cluster

Valkey 9.1+: `CLUSTERSCAN cursor [MATCH pattern] [COUNT n] [TYPE t] [SLOT slot]` iterates across cluster-owned slots and returns `[cursor, keys]`.

- Cursor is opaque (`0-{hash-tag}-local_cursor` internally); feed it back unchanged until it returns `"0"`.
- `SLOT` restricts to one slot. `MATCH` with a single hash tag can also start at that slot; conflicting `SLOT` + tag returns an empty scan.
- `COUNT` is still a hint; empty pages are valid.

`SCAN` still iterates a **single node**. For Valkey <9.1 or clients without `CLUSTERSCAN`, loop `CLUSTER NODES` (or the client's primary list) and SCAN each primary until cursor=0.

Gotchas:
- SCAN on replicas works but may miss/dup keys due to replication lag.
- Topology change mid-scan (failover or slot move) may lose/duplicate keys or invalidate a `CLUSTERSCAN` cursor. Pause resharding for critical scans.
- `KEYS *` in cluster mode only hits the one node it was sent to, and blocks it. Never use.

## Pub/Sub in cluster

Regular `PUBLISH` fans out across the cluster bus to every node - scales poorly. Use **sharded pub/sub** (`SSUBSCRIBE` / `SPUBLISH` / `SUNSUBSCRIBE`); channel name is slot-hashed (respects `{tag}`) so messages stay on one shard. See `app-pubsub.md` for full pub/sub details.

## Cluster pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Multi-key without hash tag | `CROSSSLOT` | `{tag}` |
| All data under one tag | One node overloaded | Distribute tags per entity |
| Stale reads after write | Inconsistent read-back | Primary read or `WAIT` |
| SCAN missing keys | Partial list | Iterate every primary |
| Lua touching keys across slots | Rejected or slow | Keys must hash to one slot |
| Regular pub/sub | Bus storm | `SPUBLISH`/`SSUBSCRIBE` |
