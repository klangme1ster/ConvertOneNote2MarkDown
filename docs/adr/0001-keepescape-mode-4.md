# 0001: keepescape mode 4 (repair pandoc escapes)

## Status
Proposed (2026-11-01)

## Context
With `keepescape = 3` the exported Markdown keeps every escape that Pandoc writes. OneNote pages often contain paragraphs that start with a literal `- ` or `7.`; Pandoc escapes them (`\- item`, `7\. item`), so the notes render as plain paragraphs instead of lists, with blank lines between the items. Pandoc's line-end `\` and many needless escapes (`\>`, `\_`) add noise. `keepescape = 1` removes every backslash, which also destroys Windows paths and intended escapes; `2` and `3` don't fix the list problem. See `docs/TRANSCRIPTION-QUALITY-REPORT.md`.

## Decision
Add `keepescape = 4`, implemented as the function `Repair-MarkdownEscapes` that runs on the Pandoc output before the existing mutations. It rebuilds lists, converts line-end backslashes to two spaces, and removes only escapes that cannot change the rendering. Code is shielded. The default stays `1`; modes 1 to 3 are untouched.

Rejected alternatives:
- Regex entries in the mutation list like modes 1 and 2: they cannot skip code blocks and inline code.
- Running the Python `markdown_cleaner` as a post-step: adds a Python dependency to this tool.
- Only `-escaped_line_breaks` in `$conversion`: fixes the line-end backslash, nothing else.

## Consequences
- Opt-in, so no breaking change for existing users. Notes exported with mode 4 render lists as intended.
- The rules exist twice (here and in `qon_helper.py`); on all 197 pages of the test notebook both give identical output. Changes to one should be mirrored.
- Pages where a literal `- ` was meant as text, not as a list, become lists.
