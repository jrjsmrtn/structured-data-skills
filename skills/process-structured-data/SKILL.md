---
name: process-structured-data
description: Query, safely edit and lint JSON, YAML, Markdown, TOML and SQLite using format-aware tools. Use when reading a value out of a config file, editing YAML frontmatter, bumping a version across files, scripting a change to structured data, linting Markdown or checking its relative links, or when sed/grep has mangled a file.
metadata:
  author: "Georges Martin <jrjsmrtn@gmail.com>"
  version: "0.1.8"
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
- Linting Markdown, finding broken relative links, or moving off markdownlint

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
| Markdown | `mq` | jq-like query syntax; `mq -F json` for scripting; `-A` + `section::` to address by heading |
| Markdown — *checking* it | `rumdl` | Lint and format; markdownlint rule IDs and config. See below |
| **YAML frontmatter in Markdown** | **`yq -f process`** | The right tool, and non-obvious — see below |
| TOML | `yq -p toml` | Or `taplo` where available |
| SQLite | `sqlite3` | A database you own and mutate. `-json` output pipes straight into `jq` |
| CSV, TSV, Parquet, JSONL | `duckdb` | Files you want to *interrogate* — queried in place, no import step |

```sh
jq '.version' package.json
yq '.jobs.build.steps[0].uses' .github/workflows/ci.yml
mq '.h1' README.md
sqlite3 -json data.db 'select id,name from items limit 5' | jq '.[].name'
duckdb -c "select * from 'data.csv' where amount > 100"
```

### Tabular data: stop reaching for `cut` and `awk`

CSV is the format most often attacked with line tools, and the one where they most reliably fail —
quoted fields containing commas, embedded newlines, a BOM, `\r\n` endings. All of it parses
correctly in a tool that knows the format and silently corrupts in one that does not.

```sh
duckdb -c "select count(*) from 'logs/*.parquet'"          # globs, no import
duckdb -c "select * from read_json_auto('big.json') limit 5"
duckdb -c "copy (select * from 'in.csv' where ok) to 'out.parquet'"
```

Worked example — a quoted field containing a comma, which is ordinary CSV:

```sh
printf 'name,amount\n"Smith, J",150\n' > t.csv
awk -F, 'NR==2{print $2}' t.csv                 # ->  J"      split inside the quotes
duckdb -noheader -list -c "select amount from 't.csv'"   # -> 150     correct
```

The `awk` answer is not an error — it is a plausible wrong value that flows downstream.

**TSV is the same job**, and the trap is sharper: a tab-separated file may hold commas *inside*
fields, so `-F,` on a TSV splits a value that was never a boundary. `.tsv` is auto-detected;
anything else needs the delimiter named.

```sh
duckdb -c "select * from 'data.tsv'"                                     # detected by extension
duckdb -c "select * from read_csv('data.txt', delim='\t', header=true)"  # any extension
```

**`sqlite3` versus `duckdb`** is not a preference: `sqlite3` is for a database you own, write to and
keep; `duckdb` is for files you want to ask questions of. DuckDB reads CSV, Parquet, JSON and
newline-delimited JSON **directly from disk or over HTTP**, with no load step — so it is usually the
faster answer for "what is in this file", even when the file is not a database at all.

## A rendered document is not the document

The rule above, one step earlier: **before choosing a tool, choose the artifact.** When something is
published in several formats, one is usually canonical and machine-readable and the rest are
renderings. Parsing a rendering means reverse-engineering layout decisions — page breaks,
hyphenation, column alignment — that were never content.

IETF RFCs are the worked case, because all four formats are one request apart and only one is
authoritative. [RFC 7990](https://www.rfc-editor.org/rfc/rfc7990.html) states it: *"The canonical
format for RFCs is XML using the xml2rfc version 3 (xml2rfc v3) vocabulary."* HTML, plain text and
PDF are publication formats rendered from it.

| Want | Fetch |
|---|---|
| RFC **8650 or later** | `rfc-editor.org/rfc/rfcNNNN.xml` — canonical v3, self-contained |
| RFC **before 8650** | `.txt` — no `.xml` or `.pdf` exists for these |
| ever | **not the `.pdf`** — its line breaks and hyphenation are layout, not text |

The boundary is exact and worth knowing: RFC 8650 (November 2019) is the first with `.xml`; RFC 8649
(August 2019) has none. Even RFC 7990, which *defines* the formats, predates the rollout and is not
available in them.

**The published XML is self-contained** — zero entity declarations — so an ordinary XML parser is
enough, in any language. And section names carry a slug that *is* the HTML anchor:

```sh
curl -sS https://www.rfc-editor.org/rfc/rfc9309.xml \
  | grep -oE '<name slugifiedName="[^"]+">[^<]+'      # -> name-introduction, "Introduction"
# so the citable deep link is rfc9309.html#name-introduction — verified to resolve
```

Parsing the canonical form yields citable links for free; no amount of text parsing does.

### Check status before you quote

A verbatim quotation from a superseded document is still a wrong citation, and it passes every
string check you have — the text really does say that. `rfc-index.xml` answers it in one lookup:

```sh
curl -sS https://www.rfc-editor.org/rfc-index.xml > rfc-index.xml   # ~13 MB, 9,828 entries
```

Namespace `https://www.rfc-editor.org/rfc-index`; each `rfc-entry` carries `current-status`,
`publication-status` and, when superseded, `obsoleted-by`. RFC 2616 (HTTP/1.1) reports
`obsoleted-by: RFC7230 … RFC7235` — quoting it accurately is still quoting the wrong document.
**Generalise it: any corpus of citations wants a status check beside its text check.**

### Two traps, so nobody re-derives them

- **Old plain text carries page furniture, new plain text does not.** Below RFC 8650 the `.txt` has
  form-feed page breaks with headers and footers around them; from 8650 the `.txt` starts with a
  UTF-8 BOM (`EF BB BF`), has no form feeds, and may hold genuine non-ASCII — RFC 9309 contains
  `Zürich`. Two different normalisations, split at the same boundary. A pipeline assuming ASCII
  corrupts the second kind silently.
- **Searching for a parser finds protocol implementations instead.** In every package ecosystem
  `rfc` means *"implements the protocol specified in RFC N"* — RFC 3339 dates, RFC 5545 iCalendar.
  Checked 2026-08-20: no Go or Rust tool parses RFC *documents*. For structure from pre-8650 text
  there is one option, Python's `id2xml`, and it wants the input filename to start with `rfc` or
  `draft-`, emits **v2** XML needing `xml2rfc --v2v3`, references entities it fetches over the
  network, and crashes on RFC 791. Stripping form feeds yourself is a dozen lines and no dependency.

## When the metadata lives in the prose

The neighbouring case to a rendering: a document with **no separate metadata at all**. An
Architecture Decision Record is the worked example. Its number lives in the filename *and* in the
`# N.` heading, its status is a word inside a `## Status` section that also carries explanatory
prose, and its date is a bare line. Each is a field to a reader and prose to a text tool, and
**nothing makes the copies agree**.

Address them by structure, never by line offset. `mq`'s `section` module splits a document by
heading:

```sh
mq -A 'section::section(self, "Status", 2)' 0002-govern-the-workspace.md   # the whole section
mq -A 'first(get(first(section::section(self,"Status",2)), "children")) | to_text()'
# -> Accepted
```

`-A` is not optional. Section functions need every node of the document at once; without it a bare
`section::…` fails with `"section" is not defined`, and an explicit `import "section"` instead prints
`section functions expect all document nodes` **once per node**.

### The status token must stand alone

A status word parsed out of a paragraph is ambiguous, and tools resolve the ambiguity differently —
one of them silently. Measured against `adrs` 0.11.0 (git `655552f`) on 2026-08-21:

| `## Status` content | parsed as |
|---|---|
| `Accepted` | `Accepted` |
| `Accepted.` | **`Proposed`** — a trailing period is enough |
| `Accepted. Amended by [ADR-0010](…)` | **`Proposed`** |
| `Accepted`, then wrapped prose where a line *begins* `superseded for the…` | **`Superseded`** |
| `Accepted`, then prose where `superseded` *ends* a line | `Accepted` |

The rule being applied is positional: inside `## Status`, the last line whose **first word** is a
status keyword wins. A hard wrap falling in the wrong place therefore changes a record's status, and
nothing reports it — the document reads correctly to a human and parses wrongly to a machine. `mq`
given the same input returns `Accepted. Amended by`, the paragraph's leading text: the ambiguity
stays visible instead of being guessed.

**So write the token alone on its line and put the prose in the next paragraph.** Then every reading
agrees, and the fix is a line break rather than a rewording.

### Two parsers over one corpus

The check worth having is not a stricter parser but an **independent** one. A corpus like this
usually has an authored index beside it; derive the same facts from the documents and compare both
directions — every indexed record exists, every document is indexed, and the fields match.

```sh
# per file: the heading number and title, and the status token
mq '.h1 | to_string()' "$f"                                                    # 13\. Rename `a` to `b`
mq -A 'first(get(first(section::section(self,"Status",2)),"children")) | to_text()' "$f"
```

Join those against the index in whatever language is to hand, and assert the count and both
directions. Verified on a 13-record log: 0 mismatches, then 2 when a heading title and a status token
were planted — **a comparison that has only ever agreed is not yet a check**.

### `to_text()` and `to_string()` are not interchangeable

Extracting the same heading three ways gives three different strings, and picking the wrong one
reports mismatches on a corpus that is fine:

| form | result |
|---|---|
| the file | ``# 13. Rename `setup-pre-commit` to `setup-git-hooks` `` |
| `to_text()` | `13. Rename setup-pre-commit to setup-git-hooks` — **inline code flattened** |
| `to_string()` | ``# 13\. Rename `setup-pre-commit` to `setup-git-hooks` `` — **`\.` escape added** |

`to_text()` renders to plain text; `to_string()` re-serialises to Markdown, escaping what Markdown
would otherwise interpret. For identity comparison against a raw string, take `to_string()` and undo
the escaping — or normalise both sides through `to_text()`. Do not mix them.

### Reference tooling

[`adrs`](https://github.com/joshrotenberg/adrs) (Rust, `MIT OR Apache-2.0`) reads and writes the
classic layout and answers the questions `mq` cannot cheaply reach, because they are about the corpus
as a graph rather than about one document: unresolved cross-references, duplicate and skipped
numbers, filename-number versus heading-number drift, and a superseded record that names no
successor. It also writes both halves of a supersession in one command, which is the pairing that
otherwise drifts.

Use it for those, and not for reading a status — that is the one thing measured above as wrong.
Two operational notes, both observed: it opens `$EDITOR` and blocks unless `no_edit = true` is set in
`adrs.toml`, and 0.11.0's table-of-contents generator links to title slugs rather than to filenames,
so 11 of 13 links did not resolve until the fix now on `main`.

## Frontmatter: the case people get wrong

A Markdown file with YAML frontmatter is *two* documents. A YAML tool applied to the whole file
destroys the body; a text tool corrupts the YAML.

```sh
yq -f process -i '.metadata.version = "0.2.0"' SKILL.md   # edit frontmatter, body untouched
yq -f extract '.description' SKILL.md                     # read frontmatter only
```

`-f process` edits the frontmatter and passes the body through unchanged. Verified on a file with a
21 KB body: preserved byte-for-byte.

## Validate against a schema — three different jobs

If a JSON Schema exists for a format, it does something no amount of reading can: it names **which
field** is wrong and **which constraint** it violated. `boon` covers drafts 4 through 2020-12.

### 1. Diagnose — why was this rejected?

The most under-used case. When a tool says "invalid input" and nothing more, a schema turns that into
a location and a reason, and reports **every** violation rather than stopping at the first:

```
$ boon schema.json bad.json
- at '/name': 'Structured Data Skills' does not match pattern '^[a-z0-9-]+$'
- at '/source/sha': length must be >=40, but got 7
- at '/version': '0.1' does not match pattern '^\d+\.\d+\.\d+$'
```

Three defects, three JSON Pointers, one run. Guessing would have found one and re-run twice.

### 2. Validate — gate it

Exit status is **0** for valid and **2** for invalid, so it drops straight into a hook or CI. Use
`-o detailed` or `-o basic` when something downstream needs to parse the result.

### 3. Author — write a schema for your own documents

Worth doing for any file **your project hand-edits and nothing checks**: an authored config, a
registry overlay, a manifest of your own design. Start from a known-good instance, then constrain
progressively — `required`, then `type`, then `pattern`/`enum` for the fields that actually have
rules.

**Then prove the schema can fail.** Validate a deliberately broken instance and confirm it is caught
at the field you expect. A schema that has only ever seen valid input is as unproven as any other
check — and an over-permissive schema is worse than none, because it certifies.

### Finding out whether a schema already exists

Check before concluding there is none: [SchemaStore](https://www.schemastore.org/) catalogs
**1,400+** schemas for common config formats, and its catalog is machine-readable.

```sh
curl -sSL https://www.schemastore.org/api/json/catalog.json \
  | jq -r '.schemas[] | select(.name | test("lefthook";"i")) | .url'
```

Formats you may not realise are covered: GitHub workflows, `renovate.json`, `.gitleaks.toml`,
CycloneDX BOMs, `tsconfig.json`, `package.json`, lefthook configs.

### YAML and TOML

`boon` takes JSON, so convert first — which is the one place these tools chain:

```sh
yq -o json config.yml | boon schema.json /dev/stdin
```

Verified: the same three violations are reported through the YAML path, at the same pointers.

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

## Checking Markdown: `rumdl`

`mq` reads **what a document says**; `rumdl` checks **how it is written**. They do not overlap: one
is a query language, the other a linter and formatter using markdownlint's rule IDs (`MD001`…).

```sh
uvx rumdl check docs/                 # exit 1 on any violation, 0 when clean
uvx rumdl fmt docs/                   # rewrite in place — exits 0 even if violations remain
```

It is a single Rust binary — `uvx rumdl` from PyPI, or the `rumdl` crate — with no Node. **Gate on
`check`, never on `fmt`**: `fmt` reports success while leaving what it could not fix.

**Moving off markdownlint.** `rumdl` discovers an existing `.markdownlint.json` by itself. Prove it
took effect rather than assuming: run once in a directory *without* the config and confirm a rule it
disables (`MD013`, say) appears, then with it and confirm it is gone. Then run both linters over one
tree, reduce each output to `file:line rule`, and `comm` the two sets — the disagreements are the
migration. Two that recur, measured 2026-11-20 against markdownlint-cli 0.39.0 and rumdl 0.3.5:

- **`MD024` defaults differ.** markdownlint flags a heading repeated *anywhere*; rumdl only one
  repeated under the *same parent*. A Keep a Changelog file, with `### Added` under every version,
  fails the first and passes the second.
- **`MD057` exists only in rumdl**: a relative link whose target file is missing. It resolves
  **relative to the linking file**, whatever the working directory — the check `mq`'s `file_exists`
  cannot make across a tree (see Gotchas). It checks the file, not the `#fragment`.

As a pre-commit hook: `repo: https://github.com/rvben/rumdl-pre-commit`, hook id `rumdl-check`.

**`rumdl fmt` is a rewrite, so the rule in *Editing prose bodies* applies.** Measured on a 28-file
documentation tree (rumdl 0.3.5, 2026-11-20): it fixed 892 of 928 violations, a second run changed
nothing — and the rendered HTML (`pandoc -f gfm -t html`, whitespace collapsed) changed in **2 of
the 28 files**. Some fixes are meant to change content: `MD026` deleted the `...` ending a heading,
and `MD034` turned a bare URL inside an unfenced code sample into a link. Others changed structure:
a tight list became loose. **Run `fmt` as a commit of its own, and diff the rendering before and
after** — a clean lint result says nothing about whether the meaning survived.

## Editing by exact match, with an assertion

For surgical edits an exact-string replacement beats a pattern — but only if you check that it
matched.

```python
old = 'version: "1.2.3"'
assert old in text, "pattern not found — do not assume the edit applied"
text = text.replace(old, 'version: "1.3.0"', 1)
```

**An unasserted replace is a silent no-op.** It cannot distinguish *"already correct"* from *"my
pattern is wrong"*, and both return success. Three ways it bites, all observed:

- the search string spans a **line break** the file wraps differently (folded YAML scalars, wrapped
  prose)
- **indentation** differs from what was assumed (a bullet continuation, a nested block)
- the construct **is not in that file at all** (a link-reference style the file never used)

Each reported success and changed nothing.

**Changing one value across many files: count before and after** rather than eyeballing — and
**scope the replacement to where the value is data, not prose.** A blanket replace across `*.md`
will happily rewrite the version numbers in your own examples; this document was corrupted exactly
that way, which is why the examples below use a version no real file here will ever hold.

```sh
before=$(grep -rl '"1.2.3"' --include='*.json' --include='*.md' . | wc -l)
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

## Gotchas that cost real time

**`yq`**

- **`|` has low precedence**, so a selective update needs the whole left-hand side parenthesised:
  `(.[] | select(.name == "sally") | .fruit) = "mango"`. Without the parens you get back only the
  matched fragment, not the updated document — and it looks like a successful edit.
- **`=` evaluates its right-hand side against the *root* context; `|=` evaluates it against *each*
  left-hand result.** Reach for `|=` when the new value depends on the old one.

**`mq`**

- **A selector yields `None` for every non-matching node**, and those `None`s flow downstream. Guard
  with `compact()` or `if (self != None): …`, or a later stage will operate on them.
- **`file_exists` is CWD-relative, not file-relative** — so it cannot check relative links across a
  tree in a single pass. `rumdl`'s `MD057` can (see *Checking Markdown*).
- **The rendered builtin reference is JavaScript-only** and its `llms.txt` URLs 404. The authoritative
  list, with doc-comments, is the source itself:
  `raw.githubusercontent.com/harehare/mq/main/crates/mq-lang/builtin.mq`; the whole book as one page
  is `mqlang.org/book/print.html`.

**`pandoc`** is the right tool for *conversion* rather than round-tripping — and its **Lua filters**
give full AST access, which is strictly more capable than a query language when a transformation
needs context.

**Before fighting any of these**, check whether the project publishes `llms.txt`, `llms-full.txt` or
a single-page `print.html`. A rendered docs site that needs JavaScript is often available as plain
text one URL away, and that is cheaper than inferring behaviour from experiments.

## Related Skills

- `extract-document-text` — getting text *out of* PDFs, office documents and images first
