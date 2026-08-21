# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
During development the patch level is bumped per change (`0.1.x`).

## [Unreleased]

## [0.1.7] - 2026-08-21

### Added

- **`process-structured-data` — *When the metadata lives in the prose*.** The neighbouring case to a
  rendering: a document with no separate metadata at all. **Architecture Decision Records as the
  worked case** — the number lives in the filename *and* the `# N.` heading, the status is a word
  inside a `## Status` section that also holds prose, the date is a bare line. Each is a field to a
  reader and prose to a text tool, and nothing makes the copies agree.
  - **`mq` addresses them by structure**, via a `section` module the skill had not mentioned:
    `mq -A 'section::section(self, "Status", 2)'`. `-A` is not optional — without it a bare
    `section::…` fails with `"section" is not defined`, and an explicit `import "section"` warns
    *once per node*.
  - **A status word parsed out of a paragraph is ambiguous, and one tool resolves it silently.**
    Measured against `adrs` 0.11.0: `Accepted.` with a trailing period parses as **`Proposed`**, and
    `Accepted` followed by wrapped prose whose next line *begins* `superseded for the…` parses as
    **`Superseded`** — the rule is positional, the last line whose first word is a status keyword
    wins. **A hard wrap in the wrong place changes a record's status and nothing reports it.** `mq`
    returns the paragraph's leading text instead, leaving the ambiguity visible. Write the token
    alone on its line.
  - **Two parsers over one corpus, not one stricter parser.** Derive the facts from the documents,
    compare against the authored index in both directions. Verified on a 13-record log: 0
    mismatches, then 2 once a heading title and a status token were planted — a comparison that has
    only ever agreed is not yet a check.
  - **`to_text()` and `to_string()` are not interchangeable.** The same heading yields
    `13. Rename setup-pre-commit to setup-git-hooks` (inline code flattened) or
    ``# 13\. Rename `setup-pre-commit` …`` (Markdown escape added). Mixing them reports mismatches
    on a corpus that is fine — which is how the rule was found.
  - **`adrs` cited as reference tooling, with its boundary stated**: use it for the corpus-as-graph
    questions `mq` cannot cheaply reach — unresolved cross-references, duplicate and skipped numbers,
    filename-vs-heading drift, a superseded record naming no successor — and not for reading a
    status. Two operational notes: it blocks on `$EDITOR` unless `no_edit = true`, and 0.11.0's
    table-of-contents generator emits title-slug links, 11 of 13 of which did not resolve.

## [0.1.6] - 2026-08-20

### Added

- **`process-structured-data` — *A rendered document is not the document*.** The skill's core rule
  moved one step earlier: before choosing a tool, choose the artifact. When a document is published
  in several formats, one is usually canonical and machine-readable and the rest are renderings, and
  parsing a rendering means reverse-engineering layout decisions that were never content.
  - **IETF RFCs as the worked case**, because all four formats are one request apart and only one is
    authoritative. RFC 7990: *"The canonical format for RFCs is XML using the xml2rfc version 3
    (xml2rfc v3) vocabulary."* The boundary is exact — **RFC 8650 (Nov 2019) is the first with
    `.xml`**, RFC 8649 has none, and RFC 7990 itself predates the rollout and is not available in the
    formats it defines.
  - The published XML is **self-contained** (zero entity declarations), so an ordinary XML parser in
    any language is enough — and `<name slugifiedName="…">` *is* the HTML anchor, so parsing the
    canonical form yields citable deep links that no amount of text parsing gives you.
  - **Check status before you quote.** `rfc-index.xml` (9,828 entries) carries `current-status` and
    `obsoleted-by`. RFC 2616 reports `obsoleted-by: RFC7230 … RFC7235` — **a verbatim quotation from
    a superseded document is still a wrong citation, and it passes every string check**, because the
    text really does say that. Generalised: any corpus of citations wants a status check beside its
    text check.
  - Two traps recorded so nobody re-derives them: old plain text carries form-feed page furniture
    while text from 8650 has a UTF-8 BOM, no form feeds and real non-ASCII (RFC 9309 contains
    `Zürich`) — two normalisations split at one boundary; and searching any package ecosystem for an
    RFC *parser* returns protocol implementations instead. Checked 2026-08-20: no Go or Rust tool
    parses RFC documents; Python's `id2xml` is the only option and crashes on RFC 791.
- **`extract-document-text` — an escalation ladder for fetching.** The skill described the failure
  (challenge pages, SPA shells and cookie walls are large, well-formed and contain nothing you asked
  for) and offered no escalation. Four rungs now: a published machine-readable form, `curl`,
  `xberg extract --url`, then **a real browser (Playwright)** for content that does not exist until
  JavaScript runs.
  - **Escalate on evidence, not on annoyance**: a `200` carrying a challenge page is the signal; a
    `200` carrying the document in an ugly layout is an extraction problem a browser will not fix.
  - **Rung 4 produces a different kind of evidence.** A browser renders, so what you captured is what
    that build produced on that date — record the browser and version beside the date, or the
    observation cannot be reproduced.
- **TSV in `process-structured-data`.** `.tsv` is auto-detected by duckdb; any other extension needs
  `read_csv(…, delim='\t')`. The trap is sharper than CSV's: a tab-separated file may hold commas
  *inside* fields, so `-F,` splits a value that was never a boundary.

## [0.1.5] - 2026-08-20

### Added

- **`extract-document-text` now documents both xberg distributions.** It described the Python
  package's API without ever saying that was the one it meant — so a reader who installed
  `xberg-cli`, the Rust binary, found none of it applicable and nothing telling them why. Same
  project, same version line, no shared interface.
  - **The CLI section is measured, not transcribed**: every command was run. `--format json` returns
    `{"result": {…}, "extraction_time_ms": N}`, so the text is at **`.result.content`**, beside
    `extraction_method`, `quality_score`, `tables`, `counts` and `metadata`.
  - **`--ocr-scanned-pages` is the flag worth knowing**, not `--ocr`: it OCRs only pages that look
    like scans, keeping native text elsewhere, and catches a scan whose junk hidden text layer would
    otherwise pass the quality check and return garbage as native text. Its
    `--scanned-min-confidence` defaults to 0.7, and 0.50 or lower also OCRs born-digital slides with
    a full-bleed background — a false positive you opt into rather than discover.
  - `serve`/`mcp` expose extraction as an HTTP or MCP server — a third integration route, worth
    knowing before anyone writes a wrapper.

### Fixed

- **`--stdin` without `-m` emits the raw file bytes and exits 0.** Found while running the commands
  for this section rather than by reading the help: piping a PDF in with no MIME hint returns
  `%PDF-1.3 … stream` — the container instead of the document, as a success. Detection works from a
  path and not from a pipe, so the hint is load-bearing there. It is the same failure shape the skill
  already warns about for `pdftotext` on a scan, which is why it belongs in this file and not in a
  bug report.
- **Guidance that applies to both distributions no longer sits under the Python example.**
  `extraction_method`, structure recovery, the OCR engine note and the not-quotable warning moved to
  their own section, and the `extraction_method` line now names both accessors.

### Notes

- Measured: macOS, `xberg-cli` 1.0.14, 2026-08-20, **one text-layer PDF** — `xberg extract` and
  `pdftotext` returned the same 101 words in 0.034 s and 0.039 s. Enough to say the native path costs
  nothing next to `pdftotext`; nothing about the OCR path, nothing about a corpus. The existing
  two-document caveat stands unchanged.

## [0.1.4] - 2026-08-05

### Added

- **JSON Schema validation with `boon`** (drafts 4 – 2020-12), covering three jobs that are usually
  conflated
  - **Diagnose** — the under-used one. When a tool says "invalid input" and nothing more, a schema
    turns that into a location and a reason, and reports **every** violation in one run rather than
    stopping at the first. Verified: three defects, three JSON Pointers, one invocation
  - **Validate** — exit `0` valid, `2` invalid, so it gates cleanly; `-o detailed`/`basic` when
    something downstream parses the result
  - **Author** — write a schema for files your project hand-edits and nothing checks. **Then prove
    the schema can fail**: an over-permissive schema is worse than none, because it certifies
- **Schema discovery via [SchemaStore](https://www.schemastore.org/)** — 1,400+ catalogued schemas,
  machine-readable. Check before concluding none exists; GitHub workflows, `renovate.json`,
  `.gitleaks.toml`, CycloneDX BOMs and lefthook configs are all covered
- **The YAML path**: `yq -o json config.yml | boon schema.json /dev/stdin` — verified to report the
  same violations at the same pointers, which is the one place these tools chain

## [0.1.3] - 2026-08-05

### Added

- **Tabular data** in `process-structured-data` — `duckdb` for CSV, Parquet and JSONL, queried **in
  place** with no import step, over disk or HTTP. The format most often attacked with `cut` and
  `awk`, and the one where they most reliably fail
  - With a worked example that was **run, not asserted**: on `"Smith, J",150`, `awk -F,` returns
    `` J"`` and `duckdb` returns `150`. The `awk` answer is not an error — it is a plausible wrong
    value that flows downstream
  - **`sqlite3` versus `duckdb` is not a preference**: `sqlite3` for a database you own and mutate,
    `duckdb` for files you want to interrogate

### Changed

- **`extract-document-text` now recommends `xberg` first**, with `pdftotext` as the zero-dependency
  fallback — the inverse of 0.1.2. Measured rather than assumed: **the native path is 0.1s**, the
  same as `pdftotext`, so the "cheap path first" argument was about a cost that does not exist
  outside OCR

### Fixed

- **Two overstated claims**, both corrected by measuring
  - *"Output includes Markdown tables"* — structure recovery is **document-dependent**, not
    automatic. OCR performs layout analysis and can emit tables; the native path passes through
    whatever the PDF contains. A scan produced tables; a text-layer policy document produced none
  - *"Expect seconds per document"* — true of the **OCR path only**, stated too broadly
- **OCR text is marked not-quotable without verification.** Recognition silently substitutes
  characters, and a wrong character inside quotation marks is citable and looks authoritative

### Notes

The comparison behind the inverted default is **n=2 documents** — one text-layer, one scan. Enough to
correct an overstatement and flip a default argued from assumption; not enough to claim generally.
The skill says so.

## [0.1.2] - 2026-08-05

### Fixed

- **The 0.1.1 version bump corrupted this plugin's own examples** — a blanket replace of the old
  version string across `*.md` rewrote two *illustrative* version numbers in `process-structured-data`
  as well as the frontmatter metadata. That is exactly the failure the skill documents: **a pattern
  that matched more than intended**
  - Examples now use a version this plugin will never hold, so a future bump cannot reach them
  - The skill gains the rule the incident produced: **scope a replacement to where the value is data,
    not prose**, and count what changed rather than trusting the pattern

## [0.1.1] - 2026-08-05

### Added

- **Tool-specific gotchas** carried over from working notes rather than lost when those notes were
  retired: `yq`'s low-precedence `|` (a selective update without parenthesised LHS returns the
  fragment, not the document — and looks successful), `=` versus `|=` context, `mq`'s `None`
  propagation and CWD-relative `file_exists`, and where `mq`'s authoritative builtin list actually
  lives when the rendered reference is JavaScript-only
- A pointer to check for `llms.txt` / `llms-full.txt` / single-page `print.html` before inferring a
  library's behaviour from experiments

## [0.1.0] - 2026-08-05

Initial release. Two skills, extracted from practices proven across a multi-repository portfolio
rather than written speculatively.

### Added

- **`process-structured-data`** — tool selection by format (`jq`, `yq`, `mq`, `sqlite3`), YAML
  frontmatter via `yq -f process`, surgical editing with assertions, and the four ways a check
  reports nothing useful
  - The rule that earns its place: **an unasserted string replacement is a silent no-op**, unable to
    distinguish "already correct" from "wrong pattern", both returning success
- **`extract-document-text`** — `pdftotext` first, `xberg` with automatic OCR when there is no text
  layer, and the calling conventions that are non-obvious enough to cost an hour each
  - `pdftotext`'s failure mode is **silence**: exit 0, empty output, which reads as "empty document"
    rather than "wrong tool". Check the word count, never the exit status

### Notes

Comparative measurements are deliberately excluded. Where processors differ, the skills carry the
**method** — spec suite through the processor, rendered by a reference renderer, compared as HTML —
because a recorded ranking goes stale: one processor moved from worst to best-measured in a single
release.
