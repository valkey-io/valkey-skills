# Benchmarking Valkey workloads

Use for valkey-benchmark options, client/server thread matching, pipeline-depth pitfalls, warm-up, realistic workload mix, and memtier_benchmark.

## valkey-benchmark

```
valkey-benchmark -t SET,GET        # cmd list
                 -c 50              # concurrent connections (default 50)
                 -n 100000          # total requests (default 100000)
                 --duration 60      # run by time instead of -n (mutually exclusive)
                 --warmup 10        # discard first N seconds
                 -P 16              # pipeline depth (default 1)
                 --threads 4        # client-side threads
                 -d 64              # payload size in bytes (default 3)
                 --rps 50000        # rate limit; report RPS histogram
                 --tls              # enable TLS
                 -q                 # quiet: ops/sec only
```

Pitfalls:
- Default `--threads 1` + server `io-threads 8` -> client is the bottleneck. Match `--threads` to server `io-threads`.
- One connection + deep pipeline saturates the socket buffer. Prefer `-c 100 -P 16` over `-c 1 -P 1000`.
- Use `--warmup` instead of a separate warm-up pass when available (9.1+); stats reset after warm-up.
- `-n` and `--duration` are mutually exclusive. Prefer `--duration` for steady-state comparisons.
- Bench your real R/W mix, not GET-only.
- Pipeline plateau: linear P1->P16, flattens by P64-128; deeper only adds per-batch latency.
- `--rps` limits target throughput and prints target/avg/p50/p95/p99/p100 RPS distribution - useful for jitter, not max-throughput tests.

For realistic mixed workloads: `memtier_benchmark --ratio 4:1` (80/20 R/W) validates io-threads / MPTCP / TLS-offload changes better than single-command benches.
