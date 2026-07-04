# valkey-skills

Domain-specific AI skills for the Valkey ecosystem, authored for AI coding agents and developers.

First release ships one skill: `valkey` - an application-developer reference. Installable into Claude Code, Codex, Cursor, Cline, GitHub Copilot, OpenCode, Kiro, and 50+ other AgentSkills-compatible agents.

## Why skills

A skill is a reference the agent loads on demand. `SKILL.md` is a router: trigger phrases and a table of focused topic files. The agent scans the router, reads only the topic file that matches the current task, and leaves the rest out of context. This keeps model attention on your code instead of on generic domain background.

The file format is the [AgentSkills](https://agentskills.io) open standard. One skill works across 50+ agents unmodified.

## What this skill gives you

32 topic files under `skills/valkey/references/`, routed from `skills/valkey/SKILL.md`. Files are split by user-task boundary, so a routed lookup lands in a file that is already the right semantic context:

| Area | Files | Contents |
|------|-------|----------|
| Features and versions | `valkey-features.md`, `conditional-writes.md`, `hash-field-ttl.md`, `commandlog.md`, `geosearch-bypolygon.md`, `redis-compatibility.md` | Version-gated commands (`IFEQ`, `DELIFEQ`, HSETEX family, `COMMANDLOG`, `GEOSEARCH BYPOLYGON`), per-release changelog, Redis migration |
| Application workflows | `app-caching.md`, `app-client-side-caching.md`, `app-sessions.md`, `app-locks.md`, `app-rate-limiting.md`, `app-queues.md`, `app-counters-dedup.md`, `app-leaderboards.md`, `app-pubsub.md`, `app-search.md` (router: `app-patterns.md`) | Caching, CLIENT TRACKING, sessions, locks (Redlock, fencing), rate limiting, queues/streams, counters, leaderboards, pub/sub, search |
| Cluster and HA | `cluster-key-client-behavior.md`, `cluster-slot-migration.md`, `replication-sentinel-retries.md`, `durability-persistence.md` (router: `cluster-and-ha.md`) | Hash tags, CROSSSLOT, MOVED/ASK, slot migration, replication internals, Sentinel, WAIT/WAITAOF, persistence |
| Performance | `performance-memory-encoding.md`, `performance-eviction-ttl.md`, `performance-fragmentation.md`, `performance-latency.md`, `performance-throughput.md`, `performance-benchmarking.md` (router: `performance.md`) | Encoding, eviction, fragmentation and defrag, LATENCY + COMMANDLOG diagnosis, pipelining, I/O threads, `valkey-benchmark` / `memtier_benchmark` pitfalls |
| Scripting, security, corrections | `scripting.md`, `security.md`, `anti-patterns.md` | Lua, FUNCTION LOAD, BUSY/KILL rules; AUTH, ACL, TLS; non-obvious corrections and detection commands |

Benchmarking has its own reference (`performance-benchmarking.md`) covering `valkey-benchmark` and `memtier_benchmark` syntax and the pipeline/client-thread pitfalls that skew results.

The authoring lens: what a frontier LLM would not know or would get wrong. Kept are exact defaults, error strings, INFO field names, version-gated behavior, and footguns. For example: `DEL` is already async-by-default in Valkey 8.0+; `IFEQ` never creates a missing key; `HSET` strips field TTL while `HSETEX ... KEEPTTL` preserves it; `PFCOUNT` is RW at the key-spec level; `BITFIELD` caps at `u63`; `valkey-cli --cluster reshard` still uses the legacy per-key path in 9.0. Dropped are textbook explanations, per-language client code, and narrative prose.

## Planned skills

The `valkey` skill targets application developers. Server-codebase topics are out of its scope and are planned as separate skills:

- `valkey-dev` - contributing to the Valkey server itself: data structure design and optimization (where memory fetches typically matter more than instruction count), the tcl integration-test framework, unit testing style, and coverage guidelines for changes.
- `valkey-ops` - operating Valkey in production: deployment, upgrades, monitoring, capacity.

## Install

### Universal (50+ agents)

```
npx skills add valkey-io/valkey-skills
```

The [`skills` CLI](https://github.com/vercel-labs/skills) writes the skill into each targeted agent's expected directory (e.g. `.claude/skills/`, `.cursor/skills/`, `.codex/skills/`, `.kiro/skills/`). Add `-g` for global scope or `-a <agent>` to target specific agents.

### Claude Code marketplace

```
/plugin marketplace add valkey-io/valkey-skills
/plugin install valkey
```

### Codex

```
codex plugin marketplace add valkey-io/valkey-skills
codex plugin install valkey
```

Pin to a specific commit or tag with `--ref`. The official OpenAI-curated plugin directory is in development; until then, `marketplace add` works against any public repo or local path.

### Any AgentSkills-compatible agent (manual)

Clone the repo and copy `skills/valkey/` into the agent's skills directory.

## Contributing

Authoring rules and ground rules for the repo live in [CLAUDE.md](CLAUDE.md) (which `AGENTS.md` symlinks to). Key points:

- Plain text, single-dash em-dashes.
- Verify skill content against the actual Valkey source at the declared baseline version; do not rely on web search.
- Keep SKILL.md a grep-optimized trigger list. Reference files are topic-per-file; do not fragment on a 300-line rule.
- No transient files in the repo (summary, plan, audit, analysis).

Issues and PRs welcome. For vulnerability reports, see [SECURITY.md](SECURITY.md).

## License

BSD-3-Clause. See [COPYING](COPYING).
