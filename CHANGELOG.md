# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
During development the patch level is bumped per change (`0.1.x`).

## [Unreleased]

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
