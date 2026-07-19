# Memory fragmentation and defrag

Use for INFO memory fragmentation fields, active defrag tuning, per-key memory diagnosis, when to skip defrag, and Valkey version notes that change fragmentation behavior.

## Memory fragmentation

### `INFO memory` fields

- `used_memory` - bytes allocated for data.
- `used_memory_rss` - OS-level RSS.
- `mem_fragmentation_ratio` = RSS / used_memory. `mem_fragmentation_bytes` = absolute overhead.
- `allocator_frag_ratio` / `allocator_frag_bytes` - jemalloc internal (allocated vs active). **Defrag fixes this.**
- `allocator_rss_ratio` / `allocator_rss_bytes` - RSS vs jemalloc resident. Kernel has not reclaimed pages yet. **Defrag cannot fix; resolves on its own or on restart.**

### Ratio interpretation

| Ratio | Meaning | Action |
|-------|---------|--------|
| < 1.0 | Swapping. Severe. | Increase RAM or reduce `maxmemory` now. |
| 1.0 - 1.1 | Healthy | - |
| 1.1 - 1.5 | Normal | Monitor |
| 1.5 - 2.0 | Significant | Consider active defrag |
| > 2.0 | Severe | Enable defrag or restart |

Driver: delete/churn workloads leave live allocations scattered, so pages can't return to OS.

### Active defrag config

Default off. All runtime-tunable via `CONFIG SET`.

| Key | Default |
|-----|---------|
| `activedefrag` | `no` |
| `active-defrag-threshold-lower` | 10 (%) - start |
| `active-defrag-threshold-upper` | 100 (%) - full effort |
| `active-defrag-cycle-min` | 1 (% CPU) |
| `active-defrag-cycle-max` | 25 (% CPU) |
| `active-defrag-ignore-bytes` | 104857600 (100 MB) - don't start below this overhead |

CPU scales linearly between lower and upper thresholds. Below either -> no run. Pauses during RDB save / AOF rewrite (avoids inflating COW). Runs on main thread in slices; requires jemalloc.

### `INFO stats` defrag fields

- `active_defrag_running` - current % CPU used (0 when idle).
- `active_defrag_hits` - allocations relocated.
- `active_defrag_misses` - allocations scanned, already optimal.
- `active_defrag_key_hits` / `active_defrag_key_misses` - same per-key.

### Per-key diagnosis

- `MEMORY USAGE key [SAMPLES N]` - bytes incl overhead. Default `SAMPLES 5` (approximate); `SAMPLES 0` = every element (exact, slowest).
- `valkey-cli --bigkeys` - largest key per data type.
- `MEMORY DOCTOR` - auto-checks fragmentation, peak vs current, defrag effectiveness.
- `MEMORY MALLOC-STATS` - jemalloc dump; in bins, high `nslabs` relative to `curregs` = fragmented size class.

### Skip defrag when

- Instance < 1 GB used_memory - overhead negligible.
- Low-churn / stable key population.
- CPU-bound deployments - defrag competes on the main thread.
- Short-lived instances (daily restart resets fragmentation).
- Fragmentation is in `allocator_rss_ratio` (not `allocator_frag_ratio`) - defrag can't help.

### Version notes

- **8.1** new hashtable (64 B buckets, 7 entries/bucket, chain, SIMD presence scan) -> fewer small allocations -> less fragmentation under churn. Many workloads no longer need defrag after 8.1; re-measure first.
- **9.0** Reply Copy Avoidance reduces alloc churn on read-heavy workloads marginally.
- **9.1** incremental page release during rehashing reduces latency spikes from large hash table shrink/expand work; still watch `active_defrag_*` and p99 during churn.
