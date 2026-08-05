# Contributing to structured-data-skills

Thanks for your interest. This project is a [Claude Code](https://claude.com/claude-code) plugin made
of [Agent Skills](https://agentskills.io/specification) — Markdown `SKILL.md` instruction files, not
compiled code.

## What belongs here

Guidance for **reading and writing files whose structure matters**: JSON, YAML, TOML, Markdown,
SQLite, and documents that text must be extracted from.

What does not: build systems, forge CLIs, container runtimes, linters, or anything platform-specific.
The skills are meant to travel between machines and employers, so a tool that exists only on one
operating system belongs in local configuration instead.

## The bar for a new rule

**Every rule should name the failure it prevents.** The existing ones each trace to a concrete case —
a file corrupted by shell redirection, a replacement that matched nothing, an extractor that returned
empty output with a successful exit code. A rule that cannot name its failure is a preference, and
preferences make skills longer without making them better.

Prefer rules that are **checkable**. If a hook or a one-line command can enforce it, say so.

## Measurements

**Do not commit comparative benchmarks.** Carry the method instead. Rankings between processors
change, and a figure frozen into a skill becomes a confident false claim on a schedule — one
processor here went from worst to best-measured in a single upstream release.

## Testing a change

Install the plugin locally and confirm the skill triggers on the phrasings a user would actually
reach for:

```sh
claude --plugin-dir /path/to/structured-data-skills
```

The `description` field is the activation trigger, so changes to it matter more than changes to the
body.
