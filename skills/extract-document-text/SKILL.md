---
name: extract-document-text
description: Extract text from PDFs, office documents and images, including scanned PDFs that need OCR. Use when a PDF returns no text, when reading a specification or datasheet programmatically, or when a document must be searched, quoted or checked rather than only viewed.
metadata:
  author: "Georges Martin <jrjsmrtn@gmail.com>"
  version: "0.1.5"
license: MIT
---

# Extract Document Text

Get text out of documents so it can be searched, quoted and verified — not merely displayed.

## When to Use

- A PDF must be read programmatically: a specification, standard, datasheet, enacting text
- `pdftotext` returned nothing, or returned layout garbage
- A document is a **scan** — an image wrapped in a PDF — and needs OCR
- Extracting from office formats, HTML or images

## Reach for one tool, not a decision tree

[**xberg**](https://docs.xberg.io) (renamed from `kreuzberg`, same author) handles ~100 formats and
**falls through to OCR automatically** when a PDF has no text layer. Prefer it as the default:

- **It does not fail silently.** A PDF with no text layer gets OCR'd rather than returning nothing.
- **The native path is not slow.** Measured at **0.1s** on a text-layer PDF — the same as
  `pdftotext`. Only the OCR path costs seconds.
- **One call covers DOCX, HTML, images**, not just PDF.
- **Cleaner boundaries** — no form-feed page furniture, which matters when chunking.

`pdftotext` remains worth knowing as the **zero-dependency fallback**: it is everywhere, needs no
Python, and is correct whenever a text layer exists.

**If you use it, check the word count and never the exit status.** On a scanned PDF it exits **0**
and produces an **empty file** — which reads as "empty document" rather than "wrong tool".

```sh
words=$(pdftotext doc.pdf - | wc -w)
[ "$words" -gt 0 ] || echo "no text layer — this needs OCR"
```

> **How far this was measured**: two documents — one text-layer, one scan. Enough to prefer a
> default; not enough to claim it generally. Re-measure on your own corpus if the choice matters.

## Two distributions, and they share no interface

`xberg` ships as a **Python package** and as **`xberg-cli`, a Rust binary** — one project, one
version line (both 1.0.14 as of 2026-08-20), two entirely different calling conventions. Neither is
the real one. Pick by where you are calling from, and know which you installed: every note below is
specific to one of them.

| | Python package | `xberg-cli` |
|---|---|---|
| Get it | `uv tool install xberg`, `pip install xberg` | prebuilt binary from [docs.xberg.io](https://docs.xberg.io); also published on crates.io |
| Interface | async Python API | subcommands, text or JSON on stdout |
| Reach for it | you are already inside Python | a shell pipeline, a hook, a check, an agent |

The prebuilt binary is self-contained: it ships `libheif`, `libaom`, `libx265` and `libvmaf`
alongside it, so HEIC and AV1 inputs work with no system codecs — and it is ~90 MB for that reason.

### Using the CLI

```sh
xberg extract document.pdf                  # plain text on stdout
xberg extract document.pdf --format json    # structured
xberg extract --url https://…/spec.pdf      # fetch and extract
xberg extract --stdin -m application/pdf    # bytes on a pipe — the -m is not optional, see below
xberg detect document.pdf                   # MIME type only
xberg formats                               # 119 extension -> MIME rows
```

> **`--stdin` without `-m` emits the raw file bytes and exits 0.** Piping a PDF in with no MIME hint
> returns the file itself — `%PDF-1.3`, a binary run, `3 0 obj`, `/Filter /FlateDecode`, `stream` —
> the container rather than the document, reported as a success. Detection works from a path and not
> from a pipe, so on the `--stdin` path the hint is load-bearing. Same shape as `pdftotext` exiting 0
> on a scan: the failure arrives as content, not as an error.

`--format json` returns `{"result": {…}, "extraction_time_ms": N}`, so **the text is at
`.result.content`** — and the fields you need in order to trust it sit right beside it:

```sh
xberg extract doc.pdf --format json | jq -r '.result.extraction_method, .result.quality_score'
xberg extract doc.pdf --format json | jq -r '.result.content' > doc.txt
```

`result` carries `content`, `mime_type`, `metadata`, `extraction_method`, `tables`, `counts` and
`quality_score`.

**OCR is flags, not a separate mode** — and the flag worth knowing is not `--ocr`:

- `--ocr true|false`, with `--ocr-backend tesseract|paddle-ocr|vlm|candle-*`
- `--force-ocr true` — OCR even where text extraction succeeded
- **`--ocr-scanned-pages`** — OCR only the pages that *look* like scans, keeping native text
  elsewhere. The one to reach for on a mixed document, and it catches the nastiest case: a scanned
  page carrying a junk hidden text layer that would otherwise pass the quality check and return
  garbage as if it were native text
- `--scanned-min-confidence` defaults to **0.7**. Its help records that **0.50 or lower also OCRs
  born-digital slides using a full-bleed background image** — a false positive you opt into, not one
  you discover

`batch`, `chunk`, `embed` and `cache` exist too, and **`serve`/`mcp`** expose extraction as an HTTP
or MCP server — a third integration route, and the one to consider before writing a wrapper.

> **Measured** — macOS, `xberg-cli` 1.0.14, 2026-08-20, **one text-layer PDF**: `xberg extract` and
> `pdftotext` returned the **same 101 words**, in 0.034 s and 0.039 s wall clock; self-reported
> `extraction_time_ms` was 11.8. That is enough to say the native path costs nothing next to
> `pdftotext`. It says nothing about the OCR path, and nothing about a corpus.

### Using the Python package

```python
from xberg import ExtractInput, extract

output = await extract(ExtractInput(kind="uri", uri="document.pdf"))
print(output.results[0].content)
```

Three things about that call, each of which costs an hour if guessed:

- it is **async** — `await` it, or you measure a coroutine object and conclude the tool is broken
- input is an **`ExtractInput`**, not a path string; `kind="uri"` accepts a local path
- text is at **`output.results[0].content`** — the return value is a *container*; `output.single` is a
  **method**, so `output.single is not None` is always true and tells you nothing

## What to check, whichever one you used

**Always check `extraction_method`** — `output.results[0].extraction_method` in Python,
`.result.extraction_method` in the CLI's JSON. It reports `native` for a text layer and `ocr` when
the page was recognised, and that distinction changes how far you can trust the text.

**Structure recovery is document-dependent, not automatic.** OCR performs layout analysis and can
emit Markdown tables; the native path passes through whatever the PDF already contains. Measured: a
scan produced Markdown tables, a text-layer policy document produced none. Do not assume you will get
structure back.

OCR needs an engine installed — Tesseract is the common default. Expect seconds per document on that
path only.

> **OCR text is not quotable without verification.** Recognition silently substitutes characters, and
> a wrong character inside quotation marks is worse than no quote at all — it is citable and looks
> authoritative. Before quoting any passage where `extraction_method == "ocr"`, check it against the
> rendered page.

## Read the documentation before introspecting

The three notes above are all stated on the project's own quickstart page. They were instead
rediscovered by `dir()`-ing Rust-backed objects across four failed attempts, each producing a
plausible "0 characters" that looked like a broken tool and was a broken measurement.

**Before writing a line against an unfamiliar library**: find its quickstart, and check whether it
ships an assistant plugin or conventions file. Many now do, and that is the cheapest possible source
of the calling convention.

## Verifying an extraction

An extraction that returns *something* is not an extraction that returned *the document*.

- **Check the content, never the status code.** A `200` or an exit `0` proves the tool ran. Grep the
  output for text you know is in the document — a heading, a section number, a phrase.
- **Beware plausible non-documents.** Landing pages, SPA shells, cookie walls and proof-of-work
  challenge pages are large, well-formed, and contain none of what you asked for.
- **A challenge page also destroys the existence signal.** Hosts running proof-of-work interstitials
  return the same `200` for *any* path, including ones that do not exist — so "blocked" and "the URL
  is wrong" are indistinguishable. Confirm a URL from a page that actually resolves before recording
  a source as unreachable.
- **Prove absence rather than truncation.** If a passage seems missing, check for the sections that
  *bracket* it. Neighbours present and passage absent means genuinely absent; neighbours missing too
  means a partial fetch.

## Quoting from an extracted document

- **Count, do not skim.** To claim a term does not appear, count occurrences programmatically. *"I did
  not see it"* is not a finding.
- **One source, two formats, two answers.** A specification published as both HTML and PDF can
  disagree. Where both exist, **the comparison is the verification** — and where nothing settles which
  governs, record the disagreement rather than picking one silently.
- **Date what you checked.** Extracted text is a snapshot; the document is not.

## Related Skills

- `process-structured-data` — once text is out, querying and editing it without corrupting the file
