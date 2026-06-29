# Leaderboards

Use for sorted-set rankings, around-me windows, time-bucketed leaderboard aggregation, score tiebreaks, and cluster sizing.

## Leaderboards

ZSET operations, all O(log N):

```
ZADD leaderboard 2500 "player:alice"
ZINCRBY leaderboard 100 "player:alice"
ZREVRANK leaderboard "player:alice"             # 0-indexed top rank
ZRANGE leaderboard 0 9 REV WITHSCORES           # top 10 (preferred since 6.2)
ZREVRANGE leaderboard 0 9 WITHSCORES            # legacy, still supported
ZSCORE leaderboard "player:alice"
```

"Around me" window: fetch `ZREVRANK`, then `ZRANGE leaderboard <rank-k> <rank+k> REV WITHSCORES`.

### Time-bucketed aggregation

`ZUNIONSTORE merged 7 daily:Mon daily:Tue ... AGGREGATE SUM` - combine daily buckets into weekly. Auto-clean with `EXPIRE` on bucket keys.

### Composite score tiebreak

IEEE 754 doubles give ~15 significant digits. Pack primary + tiebreaker in one score:

```
score = points * 10^10 + (MAX_TIMESTAMP - timestamp_seconds)
```

Higher points win; within a tie, earlier timestamp wins (subtract from MAX so earlier has a larger remainder).

### Cluster

A sorted set is one key -> one slot -> one shard. For 100M+ members, shard by score range or by bucket (`leaderboard:{tier:gold}`, `leaderboard:{tier:silver}`) and merge client-side for global views.
