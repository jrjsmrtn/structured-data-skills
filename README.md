# structured-data-skills

Claude Code skills for working with files whose **structure matters** — querying and editing JSON,
YAML, Markdown, TOML and SQLite with format-aware tools, and extracting text from documents
including scanned PDFs.

## Skills

| Skill | For |
|---|---|
| **`process-structured-data`** | Reading and editing structured files without corrupting them; YAML frontmatter; metadata embedded in prose; safe bulk edits; checks that actually check |
| **`extract-document-text`** | Getting text out of PDFs, office documents and images — including scans that need OCR |

## Install

```text
/plugin marketplace add jrjsmrtn/jrjsmrtn-skills
/plugin install structured-data-skills@jrjsmrtn-skills
```

## Why it exists

Line-based tools have no model of the document. `sed`, `awk` and shell redirection operate on bytes,
so they damage structured files in ways that stay invisible until something reads them closely —
usually long after the edit.

The skills carry the tool choices, the non-obvious calling conventions, and the failure modes that
produce a **confident wrong answer** rather than an error: an unasserted string replacement that
silently matches nothing, a PDF extractor that exits 0 with an empty result, an exit status taken
from the wrong end of a pipeline.

## What it is not

- **Not a tool catalogue.** Only tools for reading and writing structured data. Build systems, forge
  CLIs, container runtimes and linters belong elsewhere.
- **Not platform-specific.** No macOS- or Linux-only utilities, so the guidance travels.
- **Not a benchmark.** Where processors are compared, the skills carry the *method* and not the
  numbers — measured rankings reverse, and a frozen figure is a stale claim with a timer on it.

## Licence

MIT — see [LICENSE](LICENSE).
