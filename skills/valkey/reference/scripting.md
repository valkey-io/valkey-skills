# Lua scripting and FUNCTIONs

## EVAL/EVALSHA vs FCALL

- `EVAL "<src>" N keys... args...` - inline. First call compiles; subsequent calls use `EVALSHA <sha1> N keys... args...`.
- `SCRIPT LOAD "<src>"` -> sha1. Cache is volatile: **NOSCRIPT** after restart or primary failover; client must be ready to reload.
- `FUNCTION LOAD "#!lua name=<lib>\n <src>"` - persists in RDB/AOF, survives restart/failover. Call via `FCALL <fname> N keys... args...`.
- Production scripts that must survive restarts/failover -> `FUNCTION LOAD`, not `SCRIPT LOAD`.

## Replication

Effects replication always on. `redis.replicate_commands()` is no-op. Replicas receive write commands, not Lua source.

## Read-only variants

`EVAL_RO`, `FCALL_RO` reject writes.
- Route to replicas in cluster mode; plain EVAL/FCALL require primary.
- ACL `@read`-only users can call them.
- `FCALL_RO` requires function registered with `flags = {'no-writes'}`.
- KEYS[] must hash to same slot.

## Script timeout / BUSY

`busy-reply-threshold` default 5000 ms (legacy alias `lua-time-limit`).

Exceeded -> BUSY state, commands reject with `-BUSY ...`:
- EVAL/EVALSHA: `SCRIPT KILL` or `SHUTDOWN NOSAVE`.
- FCALL/FCALL_RO: `FUNCTION KILL` or `SHUTDOWN NOSAVE`.
- Kill works **only before any write**. After first write: `SHUTDOWN NOSAVE` only (partial writes not rolled back).

Lua memory counts against `maxmemory` (not separately capped) - accumulating script can OOM the server.

## Native replacements (prefer over Lua)

| Pattern | Old Lua | Native |
|---------|---------|--------|
| CAS | GET + conditional SET | `SET key v IFEQ old` (8.1+) |
| Safe lock release | GET + DEL | `DELIFEQ key token` (9.0+) |

No VM overhead, auditable, single-key (no hash-tag juggling). Full command surface in valkey-features.md.

## Determinism

Avoid in write scripts (even under effects replication):
- `server.call('TIME')` -> pass timestamp as ARGV.
- `server.call('RANDOMKEY')`, `math.random()` -> generate client-side.
- `server.call('SRANDMEMBER', ...)` in write paths -> FCALL_RO or accept explicitly.

## Library registration

Shebang required on first line: `#!lua name=<libname>`.

```
FUNCTION LOAD "#!lua name=mylib\n
  local function cas(keys,args) ... end
  server.register_function('cas', cas)"
```

`server.register_function`:
- Positional: `server.register_function('cas', cas)`.
- Table form (required for flags/description): `server.register_function{function_name='getprofile', callback=getprofile, flags={'no-writes'}, description='...'}`.

`no-writes` flag required to expose via `FCALL_RO`.

Management: `FUNCTION LIST | DELETE <lib> | DUMP | RESTORE <bin>`. `FUNCTION LOAD REPLACE "..."` atomic overwrite. Versioning: embed in name (`mylib_v2`) or REPLACE.

## server API

`redis.*` is an alias of `server.*` (pre-Valkey compat). Prefer `server.*`.

- `server.call(...)` - raises on error, aborts script.
- `server.pcall(...)` - returns **single value**: on error a Lua table `{err = '...'}`, NOT a `(ok, val)` tuple:
```
local r = server.pcall('get', KEYS[1])
if type(r) == 'table' and r.err then
  return server.error_reply(r.err)
end
return r
```
- `server.error_reply("ERR ...")` / `server.status_reply("OK")`.

## Cluster rules

All KEYS[] must hash to one slot; use `{tag}` to co-locate.

## Anti-patterns

- Large aggregation returns -> serialization/timeout; paginate or aggregate client-side.
- Business logic / heavy compute -> blocks main thread for all clients.
- Iterating large collections in Lua -> SCAN-family in app code.
