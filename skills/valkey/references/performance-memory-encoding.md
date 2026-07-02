# Memory encoding and key sizing

Use for compact encodings, one-way conversion, top-level key overhead, hash bucketing, collection size rules, OBJECT introspection, bitmaps, and large-value limits.

## Encoding thresholds (compact -> full; conversion is one-way)

| Type | Compact | Threshold | Full | Config keys |
|------|---------|-----------|------|-------------|
| Hash | listpack | entries <= 512 AND each field/value <= 64 B | hashtable | `hash-max-listpack-entries 512`, `hash-max-listpack-value 64` |
| Sorted set | listpack | entries <= 128 AND each member/score <= 64 B | skiplist + hashtable | `zset-max-listpack-entries 128`, `zset-max-listpack-value 64` |
| Set (strings) | listpack | entries <= 128 AND each member <= 64 B | hashtable | `set-max-listpack-entries 128`, `set-max-listpack-value 64` |
| Set (integers) | intset | entries <= 512 | hashtable | `set-max-intset-entries 512` |
| List | quicklist of listpacks (always) | node cap `list-max-listpack-size -2` (8 KB) | - | - |

**Conversion is permanent per key.** Removing elements below threshold does NOT revert encoding. Delete+recreate to restore compact form.

Past threshold, whole-collection commands (`HGETALL`, `SMEMBERS`, unbounded `ZRANGE`, `HKEYS`) become O(N) and scale with collection size.

Inspect: `OBJECT ENCODING key` -> listpack / hashtable / skiplist / intset / quicklist / embstr / int / raw / stream.

## String encoding (not configurable)

| Condition | Encoding | Alloc |
|-----------|----------|-------|
| Value fits a C `long` | `int` | 8 B inline |
| Small string whose object + embedded SDS fits <= 128 B (9.1+) | `embstr` | single alloc (object + data) |
| Larger string | `raw` | two allocs (object + data) |

Valkey 9.1 embeds when `shouldEmbedStringObject()` fits the combined object/SDS allocation in 128 B. Do not use Redis's old 44-byte `embstr` rule as a Valkey 9.1 sizing boundary.

## Top-level key overhead

~70-80 bytes per top-level key (dictEntry + SDS + object header). Dominates at millions of tiny keys.

### Hash-bucketing (Instagram: 21 GB -> 5 GB, 4x)

Bucket millions of top-level keys into listpack-sized hashes:

```
SET media:1234 <value>                       # original
HSET media:12 34 <value>                     # key = id/100, field = id%100
```

Each bucket stays under ~100 fields -> listpack -> ~5-10x smaller than individual strings.

Same reason: consolidate same-entity fields into one hash (`HSET user:1000 name ... email ...`) rather than separate keys.

Valkey 9.1 also embeds key / expiry metadata with small string objects when the whole allocation fits, reducing per-key overhead for many sub-128 B string workloads.

## Size limits (rules of thumb)

| Type | Recommended max | Why |
|------|-----------------|-----|
| Hash | < 10K fields | HGETALL latency + encoding flip |
| Set / Sorted set | < 100K members | SMEMBERS / range cost |
| List | < 100K elements | LRANGE cost |
| String value | < 1 MB | Network + memory pressure |

Split larger collections by time bucket or id range (e.g. `user:1000:events:2026-03`).

## OBJECT introspection

| Command | Returns | Requires |
|---------|---------|----------|
| `OBJECT ENCODING key` | listpack / hashtable / skiplist / intset / quicklist / embstr / int / raw | - |
| `OBJECT FREQ key` | LFU access frequency counter | `maxmemory-policy = *-lfu` |
| `OBJECT IDLETIME key` | seconds since last access | `maxmemory-policy = *-lru` (or `noeviction`) |
| `OBJECT REFCOUNT key` | refcount | - |
| `OBJECT HELP` | subcommands | - |

`valkey-cli --hotkeys` also requires an LFU policy (uses `OBJECT FREQ`). `--bigkeys` and `--memkeys` have no policy requirement.

## Bitmaps for boolean populations

100M users as bits = 12 MB per key (100e6 / 8 bytes).

```
SETBIT active:2026-03-29 <user_id> 1
GETBIT active:2026-03-29 <user_id>
BITCOUNT active:2026-03-29
BITOP AND active:both active:2026-03-28 active:2026-03-29
```

BITCOUNT SIMD (8.1+): ~6x @ 1 MB / ~10x @ 10 MB on AVX2/NEON.

## Large values

- < 100 KB: fine.
- 100 KB - 1 MB: compress client-side (clients don't auto-compress).
- > 1 MB: externalize to object store, keep only the reference in Valkey.
