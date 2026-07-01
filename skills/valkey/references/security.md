# Auth / ACL / TLS

Use when configuring AUTH, ACL users, command categories, key/channel/database permissions, application user templates, CLIENT identity, or TLS.

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
| `db=0,1` | Valkey 9.1+: allow only listed database IDs |
| `alldbs` / `resetdbs` | Valkey 9.1+: all databases / remove database access |

## Command categories

`@read @write @string @hash @list @set @sortedset @stream @pubsub @connection @transaction @scripting @admin @dangerous @slow @fast @geo @keyspace @bitmap @hyperloglog`

Runtime: `ACL CAT` - list categories; `ACL CAT <cat>` - commands in category; `ACL WHOAMI`, `ACL LIST`, `ACL GETUSER <name>`.

## DB restrictions

Valkey 9.0.x: ACLs do **not** restrict by database number. A user with `+@write ~*` can write any numbered DB the server exposes.

Valkey 9.1+ (released as tag `9.1.0`): ACLs can restrict database IDs with `db=...`, `alldbs`, and `resetdbs`.

```
ACL SETUSER db0-app on >pw ~app:* +@read +@write +@connection db=0
ACL SETUSER db12-reader on >pw ~* +@read +@connection db=1,2
```

Default is `alldbs` for backward compatibility, and `ACL LIST` omits implicit `alldbs`. `resetdbs` leaves the user unable to access numbered DBs until another `db=` or `alldbs` rule is added.

This is an ACL guard, not a tenant-isolation boundary. It is mainly relevant to standalone numbered DBs; cluster deployments normally use DB 0. Prefer separate Valkey instances or clusters when tenants require independent durability, memory, persistence, failover, or admin blast-radius controls.

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
