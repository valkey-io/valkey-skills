# CLAUDE.md

Guidance for AI agents working in this repo. `AGENTS.md` is a symlink to this file - one source of truth.

## What this repo is

valkey-io/valkey-skills is a domain-specific skills repo for the Valkey ecosystem. Single skill today (`skills/valkey/`, an application-developer reference). Classic skill layout: `skills/valkey/SKILL.md` router plus topic files under `skills/valkey/references/`. Packaging metadata lives in `.agents/plugins/`, `.codex-plugin/`, `.claude-plugin/`, and `skills/valkey/.claude-plugin/`.

## Ground rules

1. Plain text. No emojis, no ASCII art.
2. Em-dashes in prose: single dash with spaces ` - `, not ` -- `.
3. Keep transient reasoning in chat only - do not create summary, plan, audit, analysis, or temp files in the repo.
4. Always verify skill content against actual Valkey source at the baseline version tag; web research is not a substitute. Source links live in reference files.

## Working style

- No "break" / "sleep on it" / "wrap up" chatter. Next open task, ready to work.
- Concise messages. No wrapping sentences, no opinions unless asked.

## Think before coding

- State assumptions explicitly. Ask when unclear.
- Multiple interpretations -> present them, don't pick silently.
- Simpler approach exists -> say so.

## Simplicity first

Minimum content that makes the skill useful. No speculative coverage. No patterns added "for completeness."

The authoring lens: what a modern frontier LLM would not know or would get wrong. Keep exact defaults, error strings, INFO field names, version-gated behavior, footguns. Drop textbook explanations, per-language code samples, narrative prose.

If 200 lines could be 50, rewrite.

## Surgical changes

- Keep changes scoped to files needed by the task.
- Match existing style of SKILL.md + reference files.
- Orphan imports / stale cross-refs from your own changes -> remove. Pre-existing ones -> mention, don't auto-delete.
- Every changed line traces to the user's request.

## Skill architecture

- `skills/valkey/SKILL.md` - grep-optimized trigger-list router. Command names, config keys, error codes as literals. Minimal prose.
- `skills/valkey/references/` - topic-per-file; current set: `valkey-features`, `app-patterns`, `performance`, `cluster-and-ha`, `scripting`, `security`, `anti-patterns`.
- File count and per-file line count are deliberate. Preserve topic locality instead of fragmenting files to satisfy a generic 300-line heuristic.
- Keep plugin and marketplace manifests focused on packaging the existing skill content.

## Version baseline

Valkey 9.0.3. Tracked by `.github/workflows/version-watch.yml`; upstream bumps open a `version-update`-labeled issue. Bumping the baseline requires auditing skill content for changed/new commands, not just editing the workflow.

## CI

- `agnix . --strict` - runs on push to main and on PRs touching skill content, marketplace/plugin manifests, or `.agnix.toml`. Suppressions in `.agnix.toml` are documented inline; do not add new ones without a rationale comment.
- `.github/workflows/version-watch.yml` - weekly upstream check.

## Goal-driven execution

For multi-step work, state a brief plan:

```
1. [step] -> verify: [check]
2. [step] -> verify: [check]
```

Verifiable success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.
