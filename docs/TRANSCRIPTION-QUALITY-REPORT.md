# Problem report: Markdown transcription quality

Written 2026-11-01 from the Markdown cleanup side (`c:\Users\roman\SynologyDrive\Code\py\markdown_cleaner`).
This report is evidence plus a proposal. Nothing in this folder has been changed.

## Setup the findings come from

- `config.ps1`: `$keepescape = 3` (keep all Pandoc escapes), `$keepspaces = 1`,
  `$conversion = 'markdown-simple_tables-multiline_tables-grid_tables+pipe_tables-smart'`.
- Output scanned: 392 `.md` notes in `c:\Users\roman\SynologyDrive\Dokumente\onenote2md` (read only).
- Example page: `Notizbuch-von-Roman\Teufel\JIRA-Prozess-Wiki.md` (list-heavy, badly affected).
- Not verified: whether the `.docx` of these pages still exists (`usedocx = 2`); I could not find one.
  The Pandoc statements below are from reading the converter code and from Pandoc's known writer behaviour,
  not from re-running a conversion.

## Findings

Counts are pages affected / occurrences, from a regex scan of the 392 notes.

| # | Artifact in the output | Pages / occurrences | Effect when rendered |
|---|---|---|---|
| 1 | Escaped list marker at line start: `\- item` | 45 / 597 | Literal "- item" paragraphs, not a list |
| 2 | List items separated by a blank line (loose, mostly because of #1) | 65 / 369 | Wide, spaced list; in source one blank line per item |
| 3 | Line-end backslash (Pandoc `escaped_line_breaks`) | 89 / 1810 | `\` is a hard break only in CommonMark-style renderers; at the end of a paragraph it renders as a literal `\` |
| 4 | Lone `-` or `\-` line | 23 / 35 | Empty bullet; a bare `-` directly below a text line is a setext underline and turns that line into a heading |
| 5 | Escaped number: `7\.` | 22 / 133 | Not a numbered list; in `JIRA-Prozess-Wiki.md` the number even sits alone in a paragraph, with the item text in the next one |
| 6 | Needless escapes: `\>` (142 pages / 598), `\*` (33 / 489), `\_` (19 / 182) | | Noise in the source (`--\>` for `-->`, `snake\_case`) |

## Cause (read from the code)

1. **`keepspaces = 1` cannot see escaped markers.**
   The list regexes in `ConvertOneNote2MarkDown-v2.ps1` (around line 932) are
   `(\s*)- ([^\r\n]*)\r*\n\r*\n(?=\s*-)` and the numbered variant. With `keepescape = 3` the next item starts with `\-`,
   so the look-ahead `(?=\s*-)` fails and the blank line stays. The same applies to `\d+\.` vs `\d+\\\.`.
2. **`\-` and `7\.` come from paragraphs that start with literal text "- " or "7."**, not from real OneNote lists
   (a real list would come out of Pandoc as `- item`). Pandoc is faithful here and escapes the text so it does not
   turn into a list. The converter could recognise this pattern and rebuild a real list. This is the main transcription gap.
   To confirm, run Pandoc on the `.docx` of one of the affected pages and look at the paragraph styles.
3. **`keepescape` offers only coarse choices.**
   `1` deletes every `\` in the file (`[regex]::Escape('\')` → empty). That also removes backslashes that carry
   meaning: Windows paths, UNC paths (`\\server\share`), a deliberately escaped `\*`, and the `\` hard breaks.
   `2` keeps escapes before alphanumerics only. `3` keeps everything, which is where the artifacts above come from.
4. **The `\` hard break** is Pandoc's `escaped_line_breaks` extension, enabled in the default `markdown` flavour.
   Pandoc documents that disabling it (`markdown-escaped_line_breaks`) makes the writer emit two trailing spaces
   instead. I did not test this against a real page.

## Proposal

Add a fourth `keepescape` mode (or make it the default) that removes only escapes which cannot change the rendering,
and rebuilds lists from escaped markers. A working Python reference implementation with 24 unit tests exists:

- `c:\Users\roman\SynologyDrive\Code\py\markdown_cleaner\qon_helper.py`
  - `escapes`: unescapes `_` `*` `.` `-` `+` `=` `#` `>` `!` and a few more only where that cannot change the
    rendering (not at line start, not before `[`, `*` only when it cannot pair).
  - `fix_lists`: `\- ` → `- `, `7\.` → `7.` (also when the number stands alone before its text),
    drops lone `-` lines, drops blank lines between list items, turns line-end `\` into two trailing spaces
    or removes it at paragraph end. Code fences, inline code and front matter are skipped.
  - `repair` = `fix_lists` + escapes + collapse blank lines + hard breaks.
- Dry run of `repair` over the 392 notes: 245 pages changed, 0 errors; `repair` run twice differs in 4 pages
  (whitespace only).
- Porting the rules to PowerShell (as post-processing in the existing replacement list) is the minimal change.
  Alternative: run the Python filter as a post-step. That adds a Python dependency to this tool, so the
  PowerShell port is probably the better fit.

Also worth checking in the converter:

- The first lines of many pages are `| **Tags:** #Teufel #Prozess |` followed by `|---|`, a one-cell table
  with only a header row. I did not find where it is produced. If it is meant to be a tags line, a plain line
  would render better and QOwnNotes could read the tags.
- `keepspaces` also strips `>` after lists. That is documented as deliberate; leave it.

## Acceptance criteria

- Re-export of the 392-page sample (or at least `JIRA-Prozess-Wiki.md`):
  the scan counts for findings 1, 2, 4, 5 drop to zero; #3 and #6 drop to the genuinely needed cases.
- No intended backslash is lost (Windows paths, `\\server\share`, `\*` in math or code-like text).
  Add Pester cases for these.
- Existing Pester tests stay unchanged; new behaviour gets new tests.
- Default behaviour stays the same unless the user picks the new mode (it changes rendering for pages that
  start lines with "- ").
- Record the change in `LOCAL-CHANGES.md`.
