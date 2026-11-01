# What this fork changes

This is a fork of [theohbrothers/ConvertOneNote2MarkDown](https://github.com/theohbrothers/ConvertOneNote2MarkDown) (the PowerShell script that exports OneNote pages through Pandoc to Markdown). It is based on upstream `master` v2.23.1 (2026-09-24). Everything not listed here works as in upstream, and the defaults are unchanged: you only get a different result where a fix removes a bug, or when you switch on a new option.

The fixes were found by converting two real notebooks (about 200 pages each, German and English, many web clippings) and reading the result. Five of them were offered upstream as pull requests, which are still open.

## Do I need this?

Look at your existing export. If you see one of these, this fork probably helps:

| What you see in the exported notes | Fix | Where |
|---|---|---|
| Words run together: `Windowskey` instead of `Windows key` | [Non-breaking spaces](#words-were-glued-together) | PR [#191](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/191) |
| The first lines of some pages are missing (a table, a `Subject:` line, the first command) | [Page header](#page-content-was-deleted) | PR [#192](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/192) |
| Pages with the same title overwrite each other; attachment links like `[a.pdf]([b-a.pdf](...))`; images with `#` in the name show as broken | [File names](#file-names-and-links) | PR [#193](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/193) |
| Folder names like `AI ^M  ML` or `DECT^J Bluetooth` | [Section names](#section-names-with-onenote-escape-codes) | PR [#194](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/194) |
| Old notes, folders or images stay after you renamed or deleted them in OneNote | [`overwrite = 2`](#new-option-overwrite) | PR [#195](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/195) |
| Lists show as `\- item` or `7\. item`, backslashes at line ends, loose lists, `--\>` | [`keepescape = 4`](#new-option-keepescape--4) | PR [#217](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/217) |
| Lines of a paragraph run together in some programs, or show a `\` at the line end | [`hardbreaks = 2`](#new-option-hardbreaks) | this fork only |
| `Created:` / `Modified:` has the wrong time zone offset (for example +05:30) | [Time zone](#page-content-was-deleted) | PR #192 |

## How to use the fork

1. Get the script: `git clone https://github.com/klangme1ster/ConvertOneNote2MarkDown` and use branch `main`. It contains all fixes below. Each pull-request branch (`fix/...`, `enhancement/...`) contains one fix on top of upstream, if you only want some.
2. Copy `config.example.ps1` to `config.ps1` next to the script and edit it. The rest (requirements, running, FAQ) is as in the [README](README.md).

Recommended settings:

```powershell
$keepescape = 4        # rebuild lists, drop escapes that don't matter (see below). 3 keeps every backslash
$conversion = 'markdown-simple_tables-multiline_tables-grid_tables+pipe_tables-smart'   # '-smart' stops Pandoc escaping every quote as \"
$overwrite = 2         # when you re-export into an existing folder
```

Do not add `-link_attributes` or `-bracketed_spans` to `$conversion`: Pandoc then writes raw `<img style=...>` and `<span>` HTML instead of Markdown.

## The changes

### Words were glued together

*Symptom:* `the Windows key` became `Windowskey`. In one notebook 193 of 197 pages were affected.

*Cause:* with `keepspaces = 1` (the default) every non-breaking space was deleted. Web-clipped content often has a non-breaking space as the only space between two words.

*Fix:* they are now removed only at the start and end of a line (keeping the indentation, because a leading space could turn the line into a code block, a trailing double space into a hard line break). Anywhere else they become one normal space.

### Page content was deleted

*Symptom:* pages without a date line, such as emails sent to OneNote, lost real content (a Tags table, a `Subject:` line, the first command of the page).

*Cause:* the script always dropped the first 6 lines of Pandoc's output, assuming they are title, date and time.

*Fix:* the new function `Remove-PageHeader` removes the header only if a date and a time line are really there (any language, `9:19 AM` works too). Otherwise it removes a first line only if it repeats the page title.

Also in this PR: `Created:` / `Modified:` used the format `zz00`, which printed only the hours of the offset (wrong for +05:30). The output format stays `+0200`, but the value is now right.

### File names and links

Three bugs with the same cause, names built from the page path:

- **Duplicate page names overwrote each other.** The `-1`, `-2` postfix was added before the name was cut to `mdFileNameAndFolderNameMaxLength`, so the postfix could be cut off again and two pages were written to the same file. Duplicates are now detected on the cut name, and the postfix survives the cut.
- **Attachment links were nested.** With `11-36-00002.pdf` and `Datenblatt_11-36-00002.pdf` on one page, the first name also matched inside the second and the link was inserted into the other link. Now a name only matches if it is not part of a longer file name.
- **Image names broke links.** Names such as `c't-expert-community--Newsletter-#10-image1.jpg` contain `#`, which Markdown reads as a URL fragment. Image names now go through the same clean-up as attachments.

### Section names with OneNote escape codes

OneNote stores characters that are not allowed in a section file name as `^` codes: `AI / ML` is reported as `AI ^M  ML`, `DECT & Bluetooth` as `DECT^J Bluetooth`. These ended up in folder names. `ConvertFrom-OneNoteEscapedName` decodes `^M` to `/` (then `-`, as for any `/`) and `^J` to `&` for section, section group and notebook names. Only these two codes have been seen; others would still show as `^<letter>`.

*Note:* if you already exported before, these folders get new names (`AI ^M  ML` becomes `AI-ML`), so links that point to the old folder names need updating.

### New option: `overwrite`

`overwrite = 2` removes a notebook's previous output before converting it, so notes, folders and media that were renamed or deleted in OneNote since the last run do not linger. The notebook's `docx` folder is always kept, so `usedocx = 2` still reuses it. Only the notebook's own folder under `notesdestpath` is touched. Read-only files and paths over 260 characters are handled. The default `1` keeps the previous output (old behaviour).

### New option: `keepescape = 4`

*Symptom:* in a typical export many lists are broken. Many people type `- item` or `7. item` in OneNote without using real lists. Pandoc is faithful and escapes this text so it does not turn into a list:

```markdown
\- first point

\- second point

7\. numbered step\
next line --\> result
```

That renders as plain paragraphs with a hyphen in front, a literal `\` at line ends, and loose spacing. Setting `keepescape = 1` removes every backslash, but also those in Windows paths (`C:\Program Files`) and in text you meant to escape. `2` and `3` do not repair the lists.

*Fix:* `keepescape = 4` runs `Repair-MarkdownEscapes` on the Pandoc output:

```markdown
- first point
- second point

7. numbered step
next line --> result
```

- Rebuilds lists from `\- item` and `7\. item` (also inside block quotes, and when a non-breaking space follows the marker). Removes the blank lines between list items and lone `-` lines (empty bullets, or an accidental heading underline).
- A `\` at the end of a line (Pandoc's hard line break) becomes two trailing spaces, or is dropped at the end of a paragraph.
- Removes escapes that cannot change the rendering (`--\>`, `snake\_case`, `\.` in the middle of a line).
- Keeps escapes that matter: at the start of a line, `\[`, `\]`, `\*` pairs, `\\`, Windows and UNC paths.
- Code fences and inline code are never touched.
- Modes 1 to 3 and the default (`1`) are unchanged.

*Result on two real notebooks (393 pages, same `.docx` files, mode 3 against mode 4):*

| | mode 3 | mode 4 |
|---|---|---|
| escaped bullets `\-` | 601 | 0 |
| blank lines between list items | 67 | 0 |
| line-end backslashes | 1804 | 0 |
| lone `-` lines | 29 | 0 |
| escaped numbers `7\.` | 133 | 0 |

The letters and digits of every page are identical in both modes, so no text is lost. The remaining escapes (`\>`, `\*`, `\_`) are the ones kept on purpose. Details and the decision: [docs/adr/0001-keepescape-mode-4.md](docs/adr/0001-keepescape-mode-4.md) and [docs/TRANSCRIPTION-QUALITY-REPORT.md](docs/TRANSCRIPTION-QUALITY-REPORT.md).

*Caveat:* a literal `- ` at the start of a line is now treated as a list. If you meant it as plain text, keep `keepescape = 3`.

### New option: `hardbreaks`

*Symptom:* in some programs lines of one paragraph run together into a single line, in others they appear as separate lines. Markdown knows two kinds of line break: a single newline is a *soft* break (rendered as a space by strict CommonMark renderers, as a line break by others), and two spaces at the end of the line are a *hard* break that every program renders as a break. OneNote's line breaks (Shift+Enter) are written by Pandoc as a backslash at the line end, which is not rendered the same way everywhere either. `keepescape = 1` and `2` even delete that backslash, so the line break is lost.

*Fix:* `hardbreaks = 2` runs `Add-MarkdownHardBreaks` before the `keepescape` replacements:

```markdown
first line\
second line
third line
```

becomes (the two trailing spaces are invisible)

```markdown
first line  
second line  
third line
```

- A backslash at the end of a line (Pandoc's line break) becomes two spaces. A literal backslash is written by Pandoc as `\\` and stays as text.
- Every other line that is directly followed by another line of the same paragraph gets two spaces.
- Left alone: blank lines and paragraph ends, headings, tables, rules, HTML, code blocks and inline code, and a line before a list item.
- Default `1` changes nothing. The option works with every `keepescape` mode; with `keepescape = 4` the line-end backslash is converted anyway.
- Checked on one real notebook (196 pages, `keepescape = 1`): with `hardbreaks = 1` all 418 line breaks inside paragraphs are soft (the backslashes are gone); with `hardbreaks = 2` 415 are hard and 3 stay soft. The letters and digits of every page are identical.
- Limitation: a few lines that were separated only by a line of non-breaking spaces or a bare `>` end up next to each other after `keepspaces` removes that line; they stay soft.

### Not changed on purpose

- **`Modified` before `Created`:** on some pages the modified time is 1 to 2 hours before the created time. That is OneNote's own data (probably pages made by the web clipper, which store local time labelled as UTC); the script converts it correctly.
- **`>` removed at the start of lines** (`keepspaces = 1`): deliberate. Pandoc renders OneNote's indented paragraphs as block quotes, and this removes the indentation.

## Status of the pull requests

| PR | Branch | Contains |
|---|---|---|
| [#191](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/191) | `fix/keepspaces-keep-non-breaking-spaces-between-words-as-spaces` | Words glued together |
| [#192](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/192) | `fix/convert-remove-page-header-only-when-present` | Page content deleted, time zone offset |
| [#193](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/193) | `fix/file-names-keep-duplicate-postfix-and-avoid-broken-links` | Duplicate page names, nested attachment links, image names |
| [#194](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/194) | `fix/file-names-decode-onenote-escape-codes-in-section-names` | Section names with escape codes |
| [#195](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/195) | `enhancement/configuration-add-option-overwrite-to-remove-previous-output` | The `overwrite` option |

The five PRs are independent and merge cleanly in any order. Two more PRs were opened on 2026-11-01: [#217](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/217) for `keepescape = 4` and [#216](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/216), which removes the retired `windows-2019` job and the unusable PowerShell 6.x/7.0 container jobs from the CI workflow (this fork's `main` has the same CI change). #216 should be merged first, so the CI of the other PRs can finish. The PR texts are in [docs/pr-descriptions.md](docs/pr-descriptions.md). If upstream merges the PRs, `main` can simply be replaced by upstream's version.

`main` differs from the merged PR branches in one line: `Remove-PreviousNotebookOutput` also decodes the escape codes in the notebook name.

## Development notes

- **Keep both `.ps1` files pure ASCII.** Windows PowerShell 5.1 reads scripts without a BOM as ANSI, so a UTF-8 character (for example a literal non-breaking space in a regex) breaks parsing or silently changes the pattern. Write `\u00A0` in regexes and `[char]0x00A0` in strings. Check: `grep -c '[^ -~	]' *.ps1` must print `0`.
- **Run the tests without installing anything:** `test/test.ps1` installs Pester 4 into the user profile. Instead:

  ```powershell
  Save-PSResource -Name Pester -Version 4.10.1 -Path <somewhere> -TrustRepository   # PowerShell 7
  # then, in Windows PowerShell 5.1:
  Import-Module <somewhere>\Pester\4.10.1\Pester.psd1 -Force
  Invoke-Pester -Script . -Tag Unit
  ```

  Do not set `$ErrorActionPreference = 'Stop'` for the run: several tests expect non-terminating errors.
- **Test status:** 130 unit tests. The only failure, "Throw if -Length is greater than 255", depends on the system language: it expects the English .NET error message and fails on a German Windows, with or without these changes (see [docs/known-issues.md](docs/known-issues.md)).
