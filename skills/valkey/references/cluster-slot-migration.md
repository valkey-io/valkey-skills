# Cluster slot migration

Use for Valkey 9.0 atomic slot migration, CLUSTER MIGRATESLOTS command shape, migration state inspection, legacy resharding caveats, and cutover behavior.

## Atomic slot migration (9.0+)

```
CLUSTER MIGRATESLOTS SLOTSRANGE <start> <end> NODE <target-node-id> [SLOTSRANGE ... NODE ...]
CLUSTER GETSLOTMIGRATIONS
CLUSTER CANCELSLOTMIGRATIONS
```

Ranges inclusive. Node-id = 40-hex from `CLUSTER NODES`. Multiple ranges/targets in one call. `CANCELSLOTMIGRATIONS` is a safe no-op after cutover.

### Protocol

1. Source snapshots the slot (AOF-format rewrite internally).
2. Streams snapshot + live writes to target.
3. At catch-up, source briefly pauses writes on that slot (milliseconds).
4. Ownership transfers atomically.
5. Next hit on the old owner gets a single `MOVED`.

**No per-key ASK redirects.** Other slots unaffected.

### `CLUSTER GETSLOTMIGRATIONS` fields

- `operation` - `IMPORT`/`EXPORT`.
- `slot_ranges` - e.g. `"0-4095"`.
- `source_node`, `target_node` - 40-char node IDs.
- `state` - snapshotting / streaming / paused / failover / cleaning up / finished / cancelled / failed.
- `remaining_repl_size` (9.1+) - source-side output-buffer bytes remaining before the failover pause; compare with `slot-migration-max-failover-repl-bytes`.
- `last_update_time` - detect stalls.

### Footgun

Valkey 9.0 `valkey-cli --cluster reshard` used the legacy per-key path. Valkey 9.1+ adds `--cluster-use-atomic-slot-migration` for `--cluster reshard` / `--cluster rebalance`; use it or call `CLUSTER MIGRATESLOTS` directly for atomic behavior.

Latency-sensitive: schedule migrations outside peak traffic (ms-scale cutover pause).
