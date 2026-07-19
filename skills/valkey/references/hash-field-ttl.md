# Hash field TTL and key TTL semantics

Use for Valkey 9.0 hash-field expiration, Valkey 9.1 HGETDEL, read-and-refresh session fields, per-field counters, and key-level TTL return-code comparisons.

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
# Valkey 9.0.x
HSETEX key [FNX|FXX] [EX s|PX ms|EXAT unix-s|PXAT unix-ms|KEEPTTL]
          FIELDS n field value [field value...]

# Valkey 9.1+ (released as tag 9.1.0)
HSETEX key [NX|XX] [FNX|FXX] [EX s|PX ms|EXAT unix-s|PXAT unix-ms|KEEPTTL]
          FIELDS n field value [field value...]

HGETEX key [EX s|PX ms|EXAT unix-s|PXAT unix-ms|PERSIST]
          FIELDS n field [field...]

# Valkey 9.1+
HGETDEL key FIELDS n field [field...]
```

Valkey 9.1+: `NX`/`XX` on HSETEX are key-level: NX requires the hash key to be missing, XX requires it to exist. `FNX`/`FXX` are field-level and apply to the whole op: FNX requires **none** of the fields exist, FXX requires **all** exist. Conditions may be combined; any failure -> **nothing** written (atomic).

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

**HSETEX** (scalar): `1` all written; `0` condition failed (FNX/FXX, plus NX/XX on Valkey 9.1+); nothing written.

**HGETEX** (array, same shape as HMGET): values in field order; `nil` for missing or already-expired fields.

**HGETDEL** (array, same shape as HMGET): values in field order; `nil` for missing fields; existing fields are deleted. If the last field is deleted, the hash key is removed. Wrong type -> `WRONGTYPE`; `FIELDS n` mismatch -> `ERR numfields should be greater than 0 and match the provided number of fields`.

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
3. **`HSETEX` creates the key if missing unless `XX` is used (9.1+).** "Refresh CSRF token" on a logged-out session silently re-creates it. Use `HSETEX key XX ...` on 9.1+, or gate refresh writes on `EXISTS <key>` on 9.0.x.
4. **HINCRBY replication rewrite**: on a hash with volatile fields, `HINCRBY` replicates as `HSETEX ... PXAT ... FIELDS 1 <field> <new>`, not as `HINCRBY`. AOF grep for `HINCRBY` misses it.
5. **`HGETDEL` is destructive.** Use for one-shot fields, not read-and-refresh; it deletes fields even when some requested siblings are missing.

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
