# Queues and streams

Use for list queues, reliable queues with BLMOVE, streams consumer groups, pending/reclaim handling, trimming, dead letters, and priority queues.

## Queues

### List-based

- `LPUSH key job` + `BRPOP key <timeout_s>` - **at-most-once**. Consumer crash loses in-flight job.
- `LPUSH key job` + `BLMOVE src dst LEFT RIGHT <timeout_s>` + `LREM dst 1 job` on success - **at-least-once**; requires a recovery job scanning the processing list for stuck items.

Lists lack: retry counts, dead-letter, priority, scheduling. Use Streams for anything non-trivial.

### Stream-based (XADD / XREADGROUP)

Setup:

```
XGROUP CREATE queue:tasks workers $ MKSTREAM
```

- `$` = start from new messages. Use `0` to replay history.
- `MKSTREAM` creates the stream if missing.
- Duplicate create errors with `BUSYGROUP` - catch and ignore.

Produce:

```
XADD queue:tasks * type email to "u@x" subject "Hi"
# * = auto-generate ID (ms-seq, monotonic)
```

Consume:

```
XREADGROUP GROUP workers <consumer> COUNT 10 BLOCK 5000 STREAMS queue:tasks >
# > = only new messages, never-delivered to this group
# A specific ID replays pending (recovery mode) for this consumer
XACK queue:tasks workers <id>
```

On failure, do NOT ack - message stays in pending list (PEL), reclaimable.

Pending / reclaim:

```
XPENDING queue:tasks workers                        # summary
XPENDING queue:tasks workers - + 10 [consumer]      # detail
XCLAIM queue:tasks workers <new-consumer> <min_idle_ms> <id> [<id>...]
XAUTOCLAIM queue:tasks workers <new-consumer> <min_idle_ms> <cursor> [COUNT N] [JUSTID]
```

`XAUTOCLAIM` uses SCAN-like cursor, returns **three** values: `[next_cursor, claimed_entries, deleted_ids]`.
- `next_cursor == "0-0"` -> sweep complete.
- `deleted_ids` - entries that were pending but no longer exist in the stream (usually XDEL/XTRIM). Valkey removes them from the PEL while producing this response; XACK is unnecessary and returns 0.
- `JUSTID` returns IDs only and **does not increment delivery counter** (inspect without side effects).

Trimming:

```
XTRIM queue:tasks MAXLEN ~ 10000          # approximate, efficient
XTRIM queue:tasks MAXLEN  10000           # exact, slower
XTRIM queue:tasks MINID   ~ <ms>-<seq>    # time-based cutoff
XADD queue:tasks MAXLEN ~ 10000 * ...     # trim inline with produce
```

`~` keeps at-least-N entries, may keep more - server skips full listpack boundaries for much lower cost.

Dead-letter: `XPENDING stream group - + N` exposes per-message delivery count. On threshold: `XADD queue:tasks:dlq * ...` then `XACK` the original.

Multiple consumer groups: independent offsets and PEL per group. One stream serves analytics + processing simultaneously.

### Priority queue via ZSET

```
ZADD queue:priority 1  <job>      # low score = high priority
ZPOPMIN queue:priority            # returns [member, score]
BZPOPMIN queue:priority 30        # blocking, timeout_s; returns [key, member, score]
```

`BZPOPMIN` accepts multiple keys for multi-queue polling; returns the first key that has a member.

### Delivery guarantee summary

| Pattern | Delivery | ACK | Replay | Priority |
|---------|----------|-----|--------|----------|
| LPUSH + BRPOP | at-most-once | - | - | no |
| LPUSH + BLMOVE + LREM | at-least-once | manual LREM | via processing list | no |
| Stream + XREADGROUP + XACK | at-least-once | XACK | full history | no |
| ZADD + BZPOPMIN | at-most-once | - | - | yes |
