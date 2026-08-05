---
name: extract-document-text
description: Extract text from PDFs, office documents and images, including scanned PDFs that need OCR. Use when a PDF returns no text, when reading a specification or datasheet programmatically, or when a document must be searched, quoted or checked rather than only viewed.
metadata:
  author: "Georges Martin <jrjsmrtn@gmail.com>"
  version: "0.1.1"
license: MIT
---

# Extract Document Text

Get text out of documents so it can be searched, quoted and verified — not merely displayed.

## When to Use

- A PDF must be read programmatically: a specification, standard, datasheet, enacting text
- `pdftotext` returned nothing, or returned layout garbage
- A document is a **scan** — an image wrapped in a PDF — and needs OCR
- Extracting from office formats, HTML or images

## Try the cheap path first

```sh
pdftotext -layout doc.pdf - | head -40
```

Fast, no dependencies, and correct whenever the PDF carries a text layer.

**Its failure mode is silence.** On a scanned PDF it exits **0** and produces an **empty file**. That
reads as "empty document" rather than "wrong tool", and a script that only checks the exit code will
report success having extracted nothing.

```sh
words=$(pdftotext doc.pdf - | wc -w)
[ "$words" -gt 0 ] || echo "no text layer — this needs OCR"
```

Check the **word count**, never the exit status.

## When there is no text layer

[**xberg**](https://docs.xberg.io) (renamed from `kreuzberg`, same author) extracts from ~100 formats
and **runs OCR automatically** when a PDF has no text layer.

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

Confirm OCR actually ran via `output.results[0].extraction_method` (`"ocr"` vs a text-layer method).
Output includes Markdown tables for tabular layout.

OCR needs an engine installed — Tesseract is the common default. Expect seconds per document, not
milliseconds, and expect OCR text to contain recognition errors: **quote from it only after checking
the passage against the rendered page**.

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
