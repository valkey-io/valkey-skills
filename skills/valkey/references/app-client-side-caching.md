# Client-side caching with CLIENT TRACKING

Use for RESP3/RESP2 invalidation wiring, BCAST/OPTIN/OPTOUT choices, tracking-table sizing, connection-pool pitfalls, and client support.

## CLIENT TRACKING (server-assisted client-side caching)

### Protocol

RESP3: push frames on the same connection. `HELLO 3` + `CLIENT TRACKING ON`. Invalidations arrive as push frames with a key array (nil = full flush).

RESP2: no push. Two-connection REDIRECT:
- Inval conn: `CLIENT ID` -> id; `SUBSCRIBE __redis__:invalidate` (pub/sub mode - dedicate, never reuse for data).
- Data conn: `CLIENT TRACKING ON REDIRECT <id> NOLOOP`.
- Messages on `__redis__:invalidate`: array of keys to evict, or nil -> full local flush.

`__redis__:invalidate` channel name is fixed in Valkey (not renamed).

`NOLOOP`: suppress self-invalidation when the tracking client writes a key it read. Default choice for write-through caches.

### Modes

| Mode | Enable | Server cost | Precision |
|------|--------|-------------|-----------|
| Default | `CLIENT TRACKING ON` | per (key, client) | exact |
| BCAST | `CLIENT TRACKING ON BCAST PREFIX user: PREFIX session:` | per prefix subscription | prefix-wide |
| OPTIN | `CLIENT TRACKING ON OPTIN` + `CLIENT CACHING YES` before each tracked read | per opted key | exact |
| OPTOUT | `CLIENT TRACKING ON OPTOUT` + `CLIENT CACHING NO` before excluded reads | per tracked key | exact |

BCAST is the only mode that scales for high-cardinality keyspaces and is compatible with interchangeable connection pools.

### Tracking table limits

`tracking-table-max-keys` default 1000000; default mode only.

At limit: server evicts a random key and emits a **spurious invalidation** (not an error) - extra misses, not failures.

`INFO stats`:
- `tracking_total_keys` - distinct keys; bounds against `tracking-table-max-keys`.
- `tracking_total_items` - per-(key, client) entries; `items >= keys`.
- `tracking_total_prefixes` - active BCAST prefix subscriptions.

Sizing: `clients * avg_tracked_keys_per_client`. 1000 clients x 5000 keys = 5M -> raise limit or BCAST.

Per-entry ~64-128 B; 1M entries -> ~64-128 MB. BCAST stores only per-client prefix subscriptions.

### Consistency caveats

- Brief stale window between write ack and invalidation arrival.
- No ordering across multi-key writes; invalidations are independent.
- Reconnect drops pending invalidations -> **flush local cache on reconnect**, then re-run setup.
- Primary failover / restart wipes tracking table -> flush on reconnect.
- RESP2 inval connection must stay alive; if it drops, cache silently goes stale. Use `CLIENT NO-EVICT ON` on it.

### Connection-pool pitfall

Default mode is per-connection - interchangeable pools break it. Dedicate one long-lived connection per app instance, or use BCAST.

### Do-not-track

Write-heavy keys; TTL < 1 s; high-cardinality volatile keys; locks / coordination keys.

### Client library support

| Client | Server-assisted tracking |
|--------|--------------------------|
| valkey-glide (7 langs) | yes (CacheConfig) |
| valkey-go | yes |
| redisson (Java) | yes |
| lettuce (Java) | partial |
| redis-py / valkey-py | no built-in; manual RESP2 wiring |
| ioredis / iovalkey | no built-in; manual RESP2 wiring |
| node-redis | no built-in |
