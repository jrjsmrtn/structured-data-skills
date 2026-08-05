---
name: process-structured-data
description: Query and safely edit JSON, YAML, Markdown, TOML and SQLite using format-aware tools. Use when reading a value out of a config file, editing YAML frontmatter, bumping a version across files, scripting a change to structured data, or when sed/grep has mangled a file.
metadata:
  author: "Georges Martin <jrjsmrtn@gmail.com>"
  version: "0.1.0"
license: MIT
---

# Process Structured Data

Query and edit structured files with tools that have a model of the document, rather than tools that
operate on bytes and lines.

## When to Use

- Reading a value out of JSON, YAML, TOML, Markdown or SQLite
- Editing YAML frontmatter in a Markdown file
- Applying the same change across many files (a version bump, a field rename)
- After `sed`, `awk` or `>>` has damaged a file
- Writing a script that will edit structured data unattended

## The core rule

**A line-based tool has no model of the document.** `sed`, `awk`, `grep` and shell redirection
operate on bytes, so they damage structured files in ways that look fine until something reads them
closely — usually much later than the edit.

A worked example, from a real repository:

```sh
echo "x" >> CHANGELOG.md     # file had no trailing newline
                             # -> "x" joined onto the last line instead of starting a new one
sed '$ d' CHANGELOG.md       # "just delete the bad line"
                             # -> also deleted a link definition that shared it
```

Two commands, two silent corruptions, neither visible until the file was rendered.

## Pick by format

| Format | Tool | Notes |
|---|---|---|
| JSON | `jq` | Round-trips faithfully. `jq -e` sets exit status from the result — useful in checks |
| YAML | `yq` (v4) | Mike Farah's Go implementation. An unrelated Python `yq` exists; confirm with `yq --version` |
| Markdown | `mq` | jq-like query syntax; `mq -F json` for scripting |
| **YAML frontmatter in Markdown** | **`yq -f process`** | The right tool, and non-obvious — see below |
| TOML | `yq -p toml` | Or `taplo` where available |
| SQLite | `sqlite3` | `-json` output pipes straight into `jq` |

```sh
jq '.version' package.json
yq '.jobs.build.steps[0].uses' .github/workflows/ci.yml
mq '.h1' README.md
sqlite3 -json data.db 'select id,name from items limit 5' | jq '.[].name'
```

## Frontmatter: the case people get wrong

A Markdown file with YAML frontmatter is *two* documents. A YAML tool applied to the whole file
destroys the body; a text tool corrupts the YAML.

```sh
yq -f process -i '.metadata.version = "0.2.0"' SKILL.md   # edit frontmatter, body untouched
yq -f extract '.description' SKILL.md                     # read frontmatter only
```

`-f process` edits the frontmatter and passes the body through unchanged. Verified on a file with a
21 KB body: preserved byte-for-byte.

## Editing prose bodies

**Prefer a surgical edit over a rewrite.** Any tool that parses Markdown and re-emits it normalises
something — list markers, emphasis characters, wrapping, link style. Acceptable once; corrosive in a
loop.

If you must rewrite programmatically, **measure the round-trip** rather than trusting a reputation:

> Run a specification suite (CommonMark, GFM) through the processor, render both the original and the
> processed output with a reference renderer such as `cmark-gfm`, and compare the **HTML**. Judge only
> the examples the reference renderer itself gets right — otherwise you are measuring the renderer's
> limits as if they were the processor's.

That measures whether **meaning** survived. A diff line-count measures churn, which is a different and
much easier question.

**Do not trust a remembered ranking.** These reverse: one processor moved from clearly-worst to
best-measured in a single release, after upstream fixed three defects. Re-measure, or cite a
measurement with a date on it.

## Editing by exact match, with an assertion

For surgical edits an exact-string replacement beats a pattern — but only if you check that it
matched.

```python
old = 'version: "0.1.0"'
assert old in text, "pattern not found — do not assume the edit applied"
text = text.replace(old, 'version: "0.2.0"', 1)
```

**An unasserted replace is a silent no-op.** It cannot distinguish *"already correct"* from *"my
pattern is wrong"*, and both return success. Three ways it bites, all observed:

- the search string spans a **line break** the file wraps differently (folded YAML scalars, wrapped
  prose)
- **indentation** differs from what was assumed (a bullet continuation, a nested block)
- the construct **is not in that file at all** (a link-reference style the file never used)

Each reported success and changed nothing.

**Changing one value across many files: count before and after** rather than eyeballing.

```sh
before=$(grep -rl '"0.1.0"' --include='*.json' --include='*.md' . | wc -l)
# ... apply the edit ...
# expect: zero files still match the old value, and exactly `before` files match the new one
```

## Checks that actually check

Four failure modes make a check report nothing useful:

- **Measuring the wrong signal.** "Exited 0" or "output was non-empty" instead of the thing you care
  about. State the criterion as a result: *how many rows came back, does this exact string appear,
  did the file change.*
- **`$?` after a pipeline** is the last command's status, not the one you meant. Capture the exit code
  of the command itself, or the check silently reports on `grep`.
- **A pattern that matches more than you think.** In `grep -E`, alternation is `|`; writing `\|`
  matches a **literal pipe** and finds nothing — reporting a clean result for a check that never ran.
- **Printing a check without gating on it.** If the output does not change what happens next, it is
  decoration. Branch on it, or delete it.

**Prove a clean result can fail.** Feed the check a known-bad input — a planted value, a deliberately
broken fixture — and confirm it says so. A check that has only ever passed is unproven.

## Related Skills

- `extract-document-text` — getting text *out of* PDFs, office documents and images first
