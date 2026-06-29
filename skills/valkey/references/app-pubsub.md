# Pub/Sub and keyspace notifications

Use for regular pub/sub, sharded pub/sub, subscriber connection constraints, output buffers, keyspace notifications, and PUBSUB introspection.

## Pub/Sub

Fire-and-forget (at-most-once). Disconnected subscribers miss everything published during the gap. For durable messaging use Streams.

Subscriber connections are **monopolized** - cannot run regular commands. Dedicate a connection or pool.

`PSUBSCRIBE <pattern>` is **O(N)** per publish, matched against all registered patterns across all clients. Prefer exact `SUBSCRIBE` when channel names are known.

Subscriber output buffer default hard limit **32 MB** - slow consumers are disconnected. Tune `client-output-buffer-limit pubsub <hard> <soft> <soft_seconds>`.

### Cluster: sharded pub/sub

Regular `PUBLISH` fans out across the cluster bus to **every node** - wastes bandwidth at scale.

`SPUBLISH`/`SSUBSCRIBE`/`SUNSUBSCRIBE` slot-hash the channel name (respects `{tag}`) - only the node owning the slot is involved.

```
SSUBSCRIBE orders:region:us-east
SPUBLISH   orders:region:us-east '{"order_id":5678}'

# Co-locate related channels with hash tags
SSUBSCRIBE {user:1000}:notifications
SSUBSCRIBE {user:1000}:presence
```

`cluster-allow-pubsubshard-when-down yes` (default): sharded pub/sub keeps serving channels whose slot is covered, even when some cluster slots are not.

### Keyspace notifications

Disabled by default (per-op CPU overhead). Enable: `CONFIG SET notify-keyspace-events Ex`.

Channels:
- `__keyspace@<db>__:<key>` - notifications about this key (payload = event name).
- `__keyevent@<db>__:<event>` - notifications about this event type (payload = key name).

Flags:

| Flag | Meaning |
|------|---------|
| `K` | keyspace channel |
| `E` | keyevent channel |
| `g` | generic: DEL, EXPIRE, RENAME |
| `$` | string commands |
| `l` | list commands |
| `s` | set commands |
| `h` | hash commands |
| `z` | sorted-set commands |
| `t` | stream commands |
| `x` | expired |
| `e` | evicted |
| `m` | key miss (must enable explicitly) |
| `n` | new key creation (must enable explicitly) |
| `A` | alias for `g$lshzxetd` - **excludes `m` and `n`** |

At least one of `K` or `E` must be present alongside event flags.

Cluster: **notifications are local to each node** - subscribe on every primary (via sharded pub/sub) to cover the keyspace.

### PUBSUB introspection

- `PUBSUB CHANNELS [pattern]` - currently subscribed regular channels.
- `PUBSUB NUMSUB [ch...]` - subscriber counts per channel.
- `PUBSUB NUMPAT` - number of clients using PSUBSCRIBE patterns.
- `PUBSUB SHARDCHANNELS [pattern]` / `PUBSUB SHARDNUMSUB` - sharded equivalents.
