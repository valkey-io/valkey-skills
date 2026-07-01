# Conditional writes and safe compare/delete

Use for Valkey-native CAS, safe single-key lock release/renewal primitives, and Lua fallback decisions. For full distributed-lock workflow, also open `app-locks.md`.

## SET IFEQ (8.1+) - conditional update

```
SET key new_value IFEQ expected_value [EX s | PX ms | EXAT unix-s | PXAT unix-ms | KEEPTTL] [GET]
```

Returns `OK` on match-and-store; `nil` on mismatch **or missing key**.

- **`IFEQ never creates a missing key`**: a CAS retry that treats `nil` as "lost the race" must also handle **deleted-since-read**. Check `EXISTS` after or re-bootstrap via `SET ... NX`.
- With `GET`, the reply is the old value. Compare it to `expected_value`: equal means match-and-store happened; different or nil means no write (mismatch or missing key).
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
