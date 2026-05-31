# valkey-skills

Domain-specific AI skills for the Valkey ecosystem, authored for AI coding agents and developers.

First release ships one skill: `valkey` - an application-developer reference. Installable into Claude Code, Codex, Cursor, Cline, GitHub Copilot, OpenCode, Kiro, and 50+ other AgentSkills-compatible agents.

## Why skills

A skill is a reference the agent loads on demand. `SKILL.md` is a router: trigger phrases and a table of focused topic files. The agent scans the router, reads only the topic file that matches the current task, and leaves the rest out of context. This keeps model attention on your code instead of on generic domain background.

The file format is the [AgentSkills](https://agentskills.io) open standard. One skill works across 50+ agents unmodified.

## What this skill gives you

Seven topic files under `skills/valkey/reference/`, routed from `skills/valkey/SKILL.md`:

| File | Contents |
|------|----------|
| `valkey-features.md` | Version-gated commands (`IFEQ`, `DELIFEQ`, HSETEX family, `COMMANDLOG`, `GEOSEARCH BYPOLYGON`, `CLUSTER MIGRATESLOTS`, MPTCP), per-release changelog, Redis compatibility |
| `app-patterns.md` | Caching, CLIENT TRACKING, sessions, locks (Redlock, fencing), rate limiting, queues/streams, counters, leaderboards, pub/sub, search |
| `performance.md` | Encoding, eviction, fragmentation and defrag, LATENCY + COMMANDLOG diagnosis, pipelining, I/O threads |
| `cluster-and-ha.md` | Hash tags, CROSSSLOT, MOVED/ASK, replication internals, Sentinel, WAIT/WAITAOF, persistence |
| `scripting.md` | Lua, FUNCTION LOAD, BUSY/KILL rules, native replacements |
| `security.md` | AUTH, ACL, TLS |
| `anti-patterns.md` | Non-obvious corrections, listpack thresholds, detection commands |

The authoring lens: what a frontier LLM would not know or would get wrong. Kept are exact defaults, error strings, INFO field names, version-gated behavior, and footguns. For example: `DEL` is already async-by-default in Valkey 8.0+; `IFEQ` never creates a missing key; `HSET` strips field TTL while `HSETEX ... KEEPTTL` preserves it; `PFCOUNT` is RW at the key-spec level; `BITFIELD` caps at `u63`; `valkey-cli --cluster reshard` still uses the legacy per-key path in 9.0. Dropped are textbook explanations, per-language client code, and narrative prose.

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

Issues and PRs welcome.

## License

BSD-3-Clause. See [COPYING](COPYING).
