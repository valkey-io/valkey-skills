Contributing to valkey-skills
=============================

Welcome and thank you for wanting to contribute.

This repository hosts domain-specific AI skills for the Valkey ecosystem - packaged references that AI coding agents load on demand. The repo has no runtime code and no build step; packaging metadata is limited to agent marketplace and plugin manifests. Contributions are authoring and editing skill content plus the manifests that expose it.

## Get started

* Have a question? Ask it on
  [GitHub Discussions](https://github.com/valkey-io/valkey-skills/discussions)
  or [Valkey's Slack](https://valkey.io/slack)
* Found a content bug (wrong default, stale command, misleading example)? [Open an issue](https://github.com/valkey-io/valkey-skills/issues/new).
* Propose a new skill? Open an issue first with scope, target user, and what it replaces or complements - skills are narrow by design; please don't start writing a new one without alignment.
* Report a vulnerability? See [SECURITY.md](SECURITY.md).

## Developer Certificate of Origin

We respect the intellectual property rights of others and we want to make sure all incoming contributions are correctly attributed and licensed. A Developer Certificate of Origin (DCO) is a lightweight mechanism to do that. The DCO is a declaration attached to every commit. In the commit message of the contribution, the developer simply adds a `Signed-off-by` statement and thereby agrees to the DCO, which you can find below or at [DeveloperCertificate.org](http://developercertificate.org/).

```text
Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the
    best of my knowledge, is covered under an appropriate open
    source license and I have the right under that license to
    submit that work with modifications, whether created in whole
    or in part by me, under the same open source license (unless
    I am permitted to submit under a different license), as
    Indicated in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including
    all personal information I submit with it, including my
    sign-off) is maintained indefinitely and may be redistributed
    consistent with this project or the open source license(s)
    involved.
```

Every contribution to valkey-skills must be signed with a DCO. Use a known identity (real or preferred name); anonymous and pseudonymous contributions are not accepted. Every commit must end with a line like:

```text
Signed-off-by: Jane Smith <jane.smith@email.com>
```

If `user.name` and `user.email` are set in your git config, `git commit -s` (or `--signoff`) appends this line automatically. Revert commits must also be DCO-signed.

## What a skill is and what it should contain

A skill is a packaged reference an AI agent loads on demand. The file format follows the [AgentSkills](https://agentskills.io) open standard, so one skill works across Claude Code, Codex, Cursor, and 50+ other agents unmodified.

Layout:

```text
skills/<name>/
  SKILL.md                     # required: router with trigger phrases and a table of reference files
  references/
    <topic-one>.md             # focused topic file; required frontmatter: a "Use when" trigger line at the top
    <topic-two>.md
```

`SKILL.md` is a pure trigger-list router - command names, config keys, error codes as literals. Minimal prose. An agent greps the router, reads only the topic file that matches, and leaves the rest out of context.

The authoring lens: **what a frontier LLM would not know, or would get wrong.** Keep:

* Exact defaults (config keys and their default values)
* Error strings the user will see
* INFO / CLIENT / LATENCY field names the agent needs to grep for
* Version-gated behavior (what changed between releases, which version added a command, which flipped a default)
* Footguns - observed pitfalls that a textbook explanation would miss

Drop:

* Textbook explanations of what a concept is (an LLM already knows what a cache is)
* Per-language client-library code samples unless the snippet encodes a non-obvious invariant
* Narrative prose

Ground rules (also in [CLAUDE.md](CLAUDE.md), which `AGENTS.md` symlinks to):

1. Plain text. No emojis. No ASCII art.
2. Em-dashes in prose: single dash with spaces ` - `, not ` -- `.
3. Keep transient reasoning in chat only - do not commit summary, plan, audit, analysis, or temp files. Do not mention transient issues in committed content: bugs that were introduced and fixed during a working session, problems that only existed earlier in a conversation, or issues that were never part of a commit do not belong in skill files, comments, or docs.
4. Always verify skill content against actual Valkey source at the baseline version tag; web research is not a substitute.
5. Do not fragment topic files into smaller units to hit a generic 300-line heuristic; topic-per-file beats file-size limits.
6. Do not add plugin manifests beyond what packaging requires.

## How to provide a change

1. For a new skill or a scope change to an existing skill, open an issue first. Skill scope should be discussed before content is written.

2. For content fixes, corrections, new reference topics, typos, command updates:
   1. Fork valkey-skills on GitHub ([HOWTO](https://docs.github.com/en/github/getting-started-with-github/fork-a-repo)).
   1. Create a topic branch (`git checkout -b my_branch`).
   1. Make the needed changes and commit with a DCO (`git commit -s`).
   1. Push to your branch (`git push origin my_branch`).
   1. Open a pull request against `main`.

3. For minor fixes (typos, single-line corrections), a PR without a prior issue is fine.

Link a PR to an existing issue by writing `Fixes #xyz` in the PR description.

## CI: what must pass

Every PR runs:

* **`agnix --strict`** via `.github/workflows/agnix.yml` - validates skill frontmatter, directory layout, naming conventions, and cross-file consistency. Suppressions live in `.agnix.toml` and are documented inline; do not add new suppressions without a rationale comment in the PR.
* **Link check** via `.github/workflows/link-check.yml` - verifies Markdown links resolve.
* **Version watch** via `.github/workflows/version-watch.yml` - runs weekly, opens an issue when upstream Valkey has a release newer than the repo baseline. Not a PR gate.

If `agnix --strict` fails on your PR, fix the flagged issue rather than suppressing the rule. Suppressions should be architectural, not per-PR.

## Version baseline

The repo declares a Valkey baseline (currently 9.1.0) in `.github/workflows/version-watch.yml`. Bumping the baseline is a content commitment - every reference file must be audited against the new version's release notes for behavioral changes, renamed commands, new defaults, and deprecations. A baseline bump PR that only edits the workflow without touching content will be rejected.

Thanks!
