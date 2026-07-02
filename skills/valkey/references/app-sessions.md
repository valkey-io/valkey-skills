# Application sessions

Use for hash-backed sessions, sliding TTL, session rotation, per-field TTL session fields, concurrent session indexes, and session ID hygiene.

## Sessions

### Classic hash sessions

```
HSET session:abc123 user_id 1000 role admin ip "10.0.0.1"
EXPIRE session:abc123 1800

# Sliding TTL on each authenticated request
EXPIRE session:abc123 1800
```

Rotation on privilege escalation (session fixation prevention): HGETALL old -> HSET new + EXPIRE -> UNLINK old. Use MULTI/EXEC or Lua for steps 2-3 atomicity; pipelining alone allows observing intermediate state. In cluster mode, co-locate old+new keys via hash tag: `session:{user:1000}:abc123`, `session:{user:1000}:xyz789`.

### Per-field TTL (9.0+)

See `hash-field-ttl.md` for full HSETEX/HGETEX/HEXPIRE surface and gotchas. Session use case: different lifetimes per field (csrf_token 5 min, auth_token 30 min, profile_data stable).

Valkey 9.1+: use `HSETEX session:abc XX ...` for refresh/update paths that must not recreate a deleted session key.

### HGETEX: read-and-refresh atomic

Replaces HMGET + HEXPIRE pipeline. Each listed field's TTL resets; unlisted fields untouched. True per-field sliding window.

```
HGETEX session:abc EX 3600 FIELDS 2 user_id email
```

### HGETDEL: one-time fields (9.1+)

Use for one-shot CSRF/challenge fields or consume-and-remove session metadata:

```
HGETDEL session:abc FIELDS 1 csrf_token
```

Returns values in field order, `nil` for missing. It deletes the fields it returns and removes the hash key if it was the last field, so keep stable session identity fields separate from one-shot fields.

### Concurrent-session tracking

Side index of session IDs per user:

```
SADD user:1000:sessions abc123
EXPIRE user:1000:sessions 86400   # sweep orphans
SCARD user:1000:sessions          # count
SMEMBERS user:1000:sessions       # list
SREM user:1000:sessions abc123    # on destroy
SPOP user:1000:sessions           # evict random for max-N enforcement
```

### Identity hygiene

Session IDs: 128+ bits of crypto randomness. UUIDv4 is 122 bits - acceptable but tight. Never expose internal hash structure in API responses.
