# Benchmarking Valkey workloads

Use for valkey-benchmark options, client/server thread matching, pipeline-depth pitfalls, warm-up, realistic workload mix, and memtier_benchmark.

## valkey-benchmark

```
valkey-benchmark -t SET,GET        # cmd list
                 -c 50              # concurrent connections (default 50)
                 -n 100000          # total requests (default 100000)
                 -P 16              # pipeline depth (default 1)
                 --threads 4        # client-side threads
                 -d 64              # payload size in bytes (default 3)
                 --tls              # enable TLS
                 -q                 # quiet: ops/sec only
```

Pitfalls:
- Default `--threads 1` + server `io-threads 8` -> client is the bottleneck. Match `--threads` to server `io-threads`.
- One connection + deep pipeline saturates the socket buffer. Prefer `-c 100 -P 16` over `-c 1 -P 1000`.
- Warm-up pass first, then measure.
- Bench your real R/W mix, not GET-only.
- Pipeline plateau: linear P1->P16, flattens by P64-128; deeper only adds per-batch latency.

For realistic mixed workloads: `memtier_benchmark --ratio 4:1` (80/20 R/W) validates io-threads / MPTCP / TLS-offload changes better than single-command benches.
