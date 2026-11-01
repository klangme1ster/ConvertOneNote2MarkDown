# Local changes

This copy of [ConvertOneNote2MarkDown](https://github.com/theohbrothers/ConvertOneNote2MarkDown) is based on upstream `master` as of 2026-09-24. Since 2026-11-01 the folder is a git repository (`origin` = the fork, `upstream` = the original project). `main` is upstream `master` plus the five PR branches below, merged; compare with `git diff upstream/master`. The former `.orig` copies are no longer needed.

All changes were checked against the real notebooks by converting them from the kept `.docx` files and comparing the result with an export from the original script:

- 138 of 197 pages were byte-identical.
- 57 pages differed only in spaces or image names.
- 2 pages got back content that the original script had deleted.

## Changes

### Fixes

- **Words were glued together** (`keepspaces = 1`). Non-breaking spaces were deleted, but web-clipped content often has them as the only space between two words: `the<nbsp>Windows<nbsp>key` became `Windowskey`. 193 of 197 pages contained some.
  - Now they are removed only at the start and end of a line, keeping the line's indentation.
  - Anywhere else they become a single normal space.
- **Page content was deleted.** The first 6 lines of Pandoc's output were always dropped, on the assumption that they are title, date and time. Pages without those lines, such as emails sent to OneNote, lost real content: a Tags table, a Subject line, the first command of a page.
  - The new `Remove-PageHeader` removes the header only when there really is a date and time line (in any language, including `9:19 AM`).
  - Without that header, it only removes a first line that repeats the page title.
- **Duplicate page names overwrote each other.** The `-1`, `-2` postfix was added before the name was truncated to `mdFileNameAndFolderNameMaxLength`, so for long titles it was cut off again and the pages were written to the same file.
  - Duplicates are now detected on the truncated name, which also catches long names that only differ after the cut-off.
  - The postfix survives truncation (`Truncate-PathFileName -KeepSuffix`).
- **Attachment links were nested inside each other.** Inserted-file links were created with a plain text search. With attachments `11-36-00002.pdf` and `Datenblatt_11-36-00002.pdf` on one page, the first name also matched inside the second. The search now only matches a name that isn't part of a longer file name.
- **Image names broke Markdown links.** Image file names are built from the page path, which kept characters like `#` and `'` (e.g. `c't-expert-community--Newsletter-#10-image1.jpg`). A `#` in a link is read as a URL fragment. Image names now go through the same clean-up as inserted attachments (`Remove-InvalidFileNameCharsInsertedFiles`).
- **Section names contained OneNote escape codes.** OneNote reports characters that aren't allowed in section file names as `^` codes: `AI ^M  ML` for `AI / ML`, `DECT^J Bluetooth` for `DECT & Bluetooth`. They ended up literally in folder names. `ConvertFrom-OneNoteEscapedName` decodes `^M` to `/` and `^J` to `&` for section, section group and notebook names; `/` then becomes `-` as usual. These are the only two codes seen so far. Others would still show up as `^<letter>`.
- **Wrong time zone offset.** `Created:` / `Modified:` used the format `zz00`, which only printed the hours of the offset (wrong for e.g. +05:30). The output format stays the same (`+0200`).

### New option

- **`overwrite`**: `2` removes a notebook's previous output before converting it (`Remove-PreviousNotebookOutput`), so notes, folders and media renamed or deleted in OneNote since the last run don't linger.
  - The notebook's `docx` folder is always kept, so `usedocx = 2` can still reuse it.
  - Only the notebook's own folder under `notesdestpath` is touched.
  - Read-only files and paths over 260 characters are handled.
  - The default is `1` (keep previous output), which is the old behaviour.

### Not changed, for the record

- **Wrong-looking Created/Modified times:** on some pages "Modified" is 1 to 2 hours *before* "Created". That's OneNote's own data, not a conversion error: those pages (probably created by the web clipper) store local time labelled as UTC in `dateTime`. The script converts correctly.
- **`>` removed at the start of lines** (`keepspaces = 1`): deliberate. Pandoc renders OneNote's indented paragraphs as block quotes, and this un-indents them.

## Sent upstream

On 2026-09-24 all of the changes above were offered to the original project as five independent pull requests, #191 to #195. They come from the fork https://github.com/klangme1ster/ConvertOneNote2MarkDown, whose branches are `origin/<branch>` in this repository. Longer PR texts, with test details, are in `docs/pr-descriptions.md`.

| PR | Branch | Contains |
|---|---|---|
| [#191](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/191) | `fix/keepspaces-keep-non-breaking-spaces-between-words-as-spaces` | Words glued together |
| [#192](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/192) | `fix/convert-remove-page-header-only-when-present` | Page content deleted, wrong time zone offset |
| [#193](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/193) | `fix/file-names-keep-duplicate-postfix-and-avoid-broken-links` | Duplicate page names, nested attachment links, image names |
| [#194](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/194) | `fix/file-names-decode-onenote-escape-codes-in-section-names` | Section names with OneNote escape codes |
| [#195](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/195) | `enhancement/configuration-add-option-overwrite-to-remove-previous-output` | The `overwrite` option |

The PR branches differ from this copy in two small ways, so that each branch works and merges on its own:

- The new functions and tests are placed at different spots in the files.
- In the `overwrite` branch, `Remove-PreviousNotebookOutput` doesn't call `ConvertFrom-OneNoteEscapedName`.

Not in any PR: this file, the README pointer to it, and `config.ps1`.

If upstream merges the PRs, this copy can be replaced by upstream's version (keep `config.ps1`).

## Recommended configuration

In `config.ps1`:

- `$keepescape = 3`: keep all backslashes. The default `1` also deletes the ones in Windows paths (`C:\Program Files` → `C:Program Files`), and `2` turns escaped `\*` and `\<placeholder\>` back into formatting and hidden HTML.
- `$conversion = 'markdown-simple_tables-multiline_tables-grid_tables+pipe_tables-smart'`: `-smart` stops Pandoc from escaping every quote (`\"`). Don't add `-link_attributes` or `-bracketed_spans`: Pandoc then writes raw `<img style=...>` and `<span>` HTML instead.
- `$overwrite = 2` when re-exporting into an existing folder.

## Development notes

- **Keep both `.ps1` files pure ASCII.** Windows PowerShell 5.1 reads scripts without a BOM as ANSI, so a UTF-8 character (e.g. a literal non-breaking space in a regex) breaks parsing or silently changes the pattern. Write `\u00A0` in regexes and `[char]0x00A0` in strings. Check with `grep -c '[^ -~	]' *.ps1`, which should print `0`.
- **Running the tests without installing anything:** `test/test.ps1` installs Pester 4 into the user profile. Instead:

  ```powershell
  Save-PSResource -Name Pester -Version 4.10.1 -Path <somewhere> -TrustRepository   # PowerShell 7
  # then, in Windows PowerShell 5.1:
  Import-Module <somewhere>\Pester\4.10.1\Pester.psd1 -Force
  Invoke-Pester -Script . -Tag Unit
  ```

  Don't set `$ErrorActionPreference = 'Stop'` for the run: several tests expect non-terminating errors.
- **Test status:** 92 unit tests. The only failure, "Throw if -Length is greater than 255", depends on the system language: it expects the English .NET error message and fails on a German Windows, with or without these changes.
