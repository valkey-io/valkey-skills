# Auth / ACL / TLS

## AUTH forms

- `AUTH <password>` - default user.
- `AUTH <username> <password>` - named ACL user.
- Server-side: `requirepass <pw>` sets the default-user password. Named users via `ACL SETUSER`.

## ACL syntax atoms

| Atom | Meaning |
|------|---------|
| `on` / `off` | enable / disable user |
| `>password` | add password (hashed `#hash` form also accepted) |
| `<password` | remove password |
| `nopass` | allow empty-password login (testing only) |
| `resetpass` | remove all passwords |
| `+cmd` / `-cmd` | allow / deny a specific command |
| `+@category` / `-@category` | allow / deny a command category |
| `+cmd\|subcommand` | allow only a subcommand (e.g. `+client\|setname`) |
| `~pattern` | key pattern (e.g. `~app:*`) |
| `%R~pattern` | **read-only** access to pattern |
| `%W~pattern` | **write-only** access to pattern |
| `allkeys` / `resetkeys` | all keys / remove key patterns |
| `&channel-pattern` | pub/sub channel pattern |
| `allchannels` / `resetchannels` | all channels / remove channel patterns |

## Command categories

`@read @write @string @hash @list @set @sortedset @stream @pubsub @connection @transaction @scripting @admin @dangerous @slow @fast @geo @keyspace @bitmap @hyperloglog`

Runtime: `ACL CAT` - list categories; `ACL CAT <cat>` - commands in category; `ACL WHOAMI`, `ACL LIST`, `ACL GETUSER <name>`.

## DB isolation footgun

**ACLs do NOT restrict by database number.** A user with `+@write ~*` writes in **any** numbered DB the server has. Use separate Valkey instances or separate cluster deployments for true DB-level isolation. Valkey 9.0+ cluster multi-DB does not change this.

## Canonical user templates

```
# Read-write application
ACL SETUSER appuser on >pw ~app:* +@read +@write +@connection -@admin -@dangerous

# Read-only replica reader
ACL SETUSER reader on >pw ~* +@read +@connection

# Cache-only, narrow command set
ACL SETUSER cacheuser on >pw ~cache:* +GET +SET +DEL +UNLINK +EXPIRE +TTL +@connection

# Queue worker (streams + blocking list ops)
ACL SETUSER worker on >pw ~queue:* +XREADGROUP +XACK +XADD +BLPOP +RPUSH +@connection

# Separate read and write surfaces within one user
ACL SETUSER dual on >pw %R~read:* %W~write:* +@read +@write +@connection
```

`-@admin` + `-@dangerous` exclude FLUSHALL/CONFIG/DEBUG/KEYS etc. - apply to any application user.

## Connection stdlib knobs

- `CLIENT SETNAME <name>` - trace client in COMMANDLOG / CLIENT LIST. Set per service.
- `CLIENT NO-EVICT ON` on long-lived pub/sub subscriber / tracking invalidation connections to prevent server eviction under memory pressure.

## TLS

`tls-port` (commonly 6380) + certs on the server. Client provides CA (and cert+key for mTLS). 6379 stays plaintext unless `tls-port` replaces it.

**8.1+ offloads TLS handshakes to I/O threads** - connection bursts no longer block main thread. Tune `io-threads` for TLS-heavy connection churn. Per-command TLS cost after handshake is minimal.
