# Search and autocomplete

Use for prefix autocomplete, tag filtering, SINTER/SUNION/SINTERCARD, and deciding when to use valkey-search.

## Search and autocomplete

### Prefix autocomplete

Store terms with score 0 so sorted-set order is purely lexicographic. Query by prefix with `ZRANGE ... BYLEX` (canonical since 6.2; legacy `ZRANGEBYLEX` still works):

```
ZADD autocomplete 0 "apple"
ZADD autocomplete 0 "application"
ZRANGE autocomplete "[app" "[app\xff" BYLEX LIMIT 0 10
```

Store terms lowercased for case-insensitive matching. For ranked results, keep a separate scored sorted set and join in application code.

### Tag filtering

`SINTER` for AND, `SUNION` for OR. `SINTERCARD` (Redis/Valkey 7.0) returns count without fetching members - useful for "X results" UI counters with an early-stop LIMIT.

Cluster: multi-key set operations require all keys in same slot - use hash tags `{ns}:tag:electronics`.

### When to use valkey-search instead

| Need | Approach |
|------|----------|
| Simple prefix autocomplete | `ZRANGE ... BYLEX` |
| Tag AND/OR filtering | `SINTER` / `SUNION` |
| Full-text, fuzzy, stemming | valkey-search module (`FT.SEARCH`) |
| Relevance scoring with field weights | valkey-search module |
| Vector similarity | valkey-search module |
