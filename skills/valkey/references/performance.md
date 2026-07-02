# Performance diagnosis routing

Use this as the performance router. Pick the file matching the diagnosis area instead of loading all memory, latency, throughput, and benchmark guidance together.

| Diagnosis area | Open | Common triggers |
|----------------|------|-----------------|
| Memory encoding and key sizing | `performance-memory-encoding.md` | listpack, hashtable, skiplist, intset, quicklist, embstr, 9.1 small strings, top-level key overhead, hash-bucketing, size limits, OBJECT ENCODING, bitmaps, large values |
| Eviction, TTL, and big-key deletion | `performance-eviction-ttl.md` | TTL, PTTL, EXPIRETIME, jitter, maxmemory, allkeys-lru, allkeys-lfu, volatile-lru, noeviction, evicted_keys, UNLINK, drain big key |
| Fragmentation and defrag | `performance-fragmentation.md` | used_memory_rss, mem_fragmentation_ratio, allocator_frag_ratio, active-defrag, MEMORY DOCTOR, MEMORY MALLOC-STATS |
| Latency diagnosis | `performance-latency.md` | intrinsic latency, latency monitor, LATENCY DOCTOR, COMMANDLOG, fork, expiration storm, aof-write-pending-fsync, swap, p99 |
| Throughput, pipelining, and hot keys | `performance-throughput.md` | lazyfree defaults, SCAN semantics, MSETEX, pipelining, connection pooling, io-threads, events-per-io-thread, hot key, CLIENT TRACKING |
| Benchmarking | `performance-benchmarking.md` | valkey-benchmark, --warmup, --duration, --rps, memtier_benchmark, pipeline depth, client threads, warm-up, R/W mix |
