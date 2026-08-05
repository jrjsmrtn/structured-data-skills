---
name: extract-document-text
description: Extract text from PDFs, office documents and images, including scanned PDFs that need OCR. Use when a PDF returns no text, when reading a specification or datasheet programmatically, or when a document must be searched, quoted or checked rather than only viewed.
metadata:
  author: "Georges Martin <jrjsmrtn@gmail.com>"
  version: "0.1.4"
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

## Using it

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

**Always check `output.results[0].extraction_method`.** It reports `native` for a text layer and
`ocr` when the page was recognised — and that distinction changes how far you can trust the text.

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
