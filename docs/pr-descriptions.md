# Pull request descriptions

Paste each title and description into GitHub's "Open a pull request" form for the matching branch.

All five PRs are independent and merge cleanly in any order and combination. On a German Windows, one existing test ("Throw if -Length is greater than 255") fails with or without these changes: it expects the English .NET error message.

---

## 1. Branch `fix/keepspaces-keep-non-breaking-spaces-between-words-as-spaces`

**Title:** Fix (keepspaces): Keep non-breaking spaces between words as spaces

**Description:**

With `keepspaces = 1` (the default), every non-breaking space is removed. Content clipped from web pages often has a non-breaking space as the *only* space between two words, so words get glued together. For example, a page containing `press the<nbsp> Windows<nbsp>key at the same time<nbsp>to open` came out as `press the Windowskey at the same timeto open`. In my notebooks, 193 of 197 pages contained non-breaking spaces (4306 in total).

This PR changes the rule:

- **Start of a line:** removed, keeping the line's indentation. Turning them into spaces there could create an indented code block.
- **End of a line, and lines that contain only non-breaking spaces:** removed. Turning them into spaces there could create a trailing-double-space hard line break.
- **Anywhere else:** a run of them, together with adjacent spaces, becomes a single space.

The option's description and the README are updated to match. The `keepspaces` test now also covers words separated by non-breaking spaces, a line that starts with them, and an indented line.

---

## 2. Branch `fix/convert-remove-page-header-only-when-present`

**Title:** Fix (convert): Remove the page's title, date and time lines only when present

**Description:**

`Convert-OneNotePage` always discards the first 6 lines of pandoc's output, assuming they are the page's title, date and time. Some pages don't have these, for example emails sent to OneNote from Outlook. On those pages, real content was lost. In my notebooks this happened to two pages:

- one lost a `Tags` table and its `Subject:` line
- the other lost its first line, a `curl` command

This PR adds `Remove-PageHeader`, which only removes the header when it's really there:

- **Title, date, time:** removed when there is a date line (any text with a 4-digit year, so it works in any language, e.g. `Donnerstag, 17. Oktober 2024` or `Monday, September 8, 2014`) followed by a time line (`09:50`, `9:19 AM`). The title may be empty; an untitled page yields a line with only a non-breaking space.
- **No date/time:** only a first line that repeats the page's name is removed. The comparison uses letters and digits only, because pandoc escapes and autolinks the title.
- **Anything else:** kept as is.

I checked this against pandoc's output for all 197 pages of one of my notebooks. The header was recognized on all pages that have one, and the two pages without one kept their content.

**Also included**, since it's about the same header: the `Created:` / `Modified:` lines used the format `zz00`, which only prints the hours of the UTC offset. That is wrong for time zones like +05:30. The output format doesn't change (`+0200`).

New unit tests cover `Remove-PageHeader`: German and English headers, a title containing a year, an untitled page, a page without a header, a title-only page, a header-only page, and empty input.

---

## 3. Branch `fix/file-names-keep-duplicate-postfix-and-avoid-broken-links`

**Title:** Fix (file names): Keep duplicate page postfixes and avoid broken attachment and image links

**Description:**

Three related problems with generated file names and links.

**Duplicate page names overwrite each other.** The `-1`, `-2` postfix for identically named pages in a section is appended *before* the name is truncated to `mdFileNameAndFolderNameMaxLength`. For long page names, truncation cuts the postfix off again, and the pages are written to the same file. Now:

- duplicates are detected on the truncated name, which also catches long names that only differ after the cut-off
- the postfix survives truncation, via a new `Truncate-PathFileName -KeepSuffix` parameter

**Nested attachment links.** Inserted-file references are replaced with a plain text search. A page with the attachments `11-36-00002.pdf` and `Datenblatt_11-36-00002.pdf` ended up with a broken, nested link: `[Datenblatt_[11-36-00002.pdf](media/11-36-00002.pdf)](media/Datenblatt_[11-36-00002.pdf](media/11-36-00002.pdf))`. The search now only matches a name that isn't part of a longer file name.

**Broken image links.** Image file names are prefixed with the page path, which can contain characters like `#` and `'`. For example, a page named `c't expert community: Newsletter #10` produces `c't-expert-community--Newsletter-#10-image1.jpg`. In a Markdown link, `#` starts a URL fragment, so the image doesn't load. Image names now go through `Remove-InvalidFileNameCharsInsertedFiles`, like inserted attachments already do.

New unit tests: `-KeepSuffix` in both cases, long duplicate page names (including two names that only differ after the cut-off), and an attachment name contained in another attachment's name.

---

## 4. Branch `fix/file-names-decode-onenote-escape-codes-in-section-names`

**Title:** Fix (file names): Decode OneNote's escape codes in section names

**Description:**

OneNote reports characters that aren't allowed in section file names as `^` codes in the section's name, for example:

- `AI ^M  ML` for a section named `AI / ML`
- `DECT^J Bluetooth` for `DECT & Bluetooth`

These codes ended up literally in folder names (`AI-^M--ML`, `DECT^J-Bluetooth`).

This PR adds `ConvertFrom-OneNoteEscapedName`, which decodes `^M` to `/` and `^J` to `&`, the two codes I've seen. It runs before `Remove-InvalidFileNameChars` for notebook, section group and section names, so `/` then becomes `-` as usual. Other codes, if they exist, are left as they are.

If the `overwrite` PR (#5 in this series) is merged too, `Remove-PreviousNotebookOutput` should use the same decoding for the notebook's directory name, so both agree for notebook names that contain these codes.

New unit tests cover the decoding and the resulting directory name.

---

## 5. Branch `enhancement/configuration-add-option-overwrite-to-remove-previous-output`

**Title:** Enhancement (configuration): Add option `overwrite` to remove a notebook's previous output

**Description:**

When converting into an existing notes folder, files of the same name are overwritten. But notes, folders and media that were renamed or deleted in OneNote since the last run are left behind, and they're easy to mistake for current notes.

This PR adds the option `overwrite`:

- **`1` (default):** current behavior.
- **`2`:** before converting a notebook, its previous output is removed (`Remove-PreviousNotebookOutput`).
  - The notebook's `docx` folder is kept, so `usedocx = 2` can still reuse it.
  - Only the notebook's own folder under `notesdestpath` is touched, and nothing happens on a dry run.
  - Read-only files (e.g. inserted attachments, which OneNote marks read-only) are removed too.
  - On Windows PowerShell 5, paths longer than 260 characters are handled via the `\\?\` prefix.

This overlaps in purpose with #190 (`skipIfExists`), which is about re-running conversions too. The two options don't conflict.

New unit tests run against real files in `TestDrive`: removal while keeping `docx`, including a read-only file; a dry run; and a notebook folder that doesn't exist. I also used it on two notebooks (393 pages, with long paths): a planted stale note was removed, and both `docx` folders were kept.

---

## 6. Branch `enhancement/configuration-add-keepescape-mode-4-to-repair-pandoc-escapes`

Based on upstream `master`, independent of PRs 1 to 5. Opened upstream as [#217](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/217) on 2026-11-01.

**Title:** Enhancement (configuration): Add keepescape mode 4 to repair Pandoc escapes

**Description:**

OneNote pages often start paragraphs with a literal `- ` or `7.` instead of using real lists. Pandoc escapes such text so it does not turn into a list (`\- item`, `7\. item`). In an export of two notebooks (393 pages) this left the notes with plain paragraphs instead of lists, a blank line between every item, a literal `\` at the end of many lines (Pandoc's hard line break) and many needless escapes (`--\>`, `snake\_case`).

The existing `keepescape` modes cannot fix this: `1` deletes every backslash (also in Windows paths such as `C:\Program Files`), `2` and `3` keep the list escapes.

This PR adds `keepescape = 4`, implemented as the function `Repair-MarkdownEscapes`, run on Pandoc's output before the existing replacements:

- Rebuilds lists from `\- item` and `7\. item` (also inside block quotes and when a non-breaking space follows the marker), removes the blank lines between list items, and drops lone `-` lines (empty bullets, or an accidental heading underline).
- Turns a `\` at the end of a line into two trailing spaces, or drops it at the end of a paragraph.
- Removes escapes that cannot change the rendering (`\>` in `--\>`, `\_` in `snake\_case`, `\.` and `\-` in the middle of a line).
- Keeps escapes that matter: at the start of a line, `\[`, `\]`, `\*` pairs, `\`, Windows and UNC paths.
- Never touches code fences and inline code.

The default (`1`) and modes 1 to 3 are unchanged. A literal `- ` at the start of a line is treated as a list in mode 4; use `3` if that is not wanted.

**Tested**

- 22 new unit tests: each rule, cases that must stay untouched (Windows and UNC paths, `\*`, code), block quotes, non-breaking spaces, running it twice gives the same result, and that mode 4 adds no replacement that touches backslashes.
- Real run against OneNote with two notebooks (393 pages, same `.docx` files, mode 3 against mode 4): escaped bullets 601 to 0, blank lines between list items 67 to 0, line-end backslashes 1804 to 0, lone `-` lines 29 to 0, escaped numbers 133 to 0. The letters and digits of every page are identical in both modes, so no text is lost.
- Unit tests: 96 passed in Windows PowerShell 5.1; the one failure ("Throw if -Length is greater than 255") expects the English .NET error message and also fails on `master` on a German Windows.
- Not tested here: PowerShell 6.x and 7.0 (see the CI PR below).

---

## 7. Branch `fix/ci-remove-retired-windows-2019-and-unusable-container-jobs`

Based on upstream `master`. Opened upstream as [#216](https://github.com/theohbrothers/ConvertOneNote2MarkDown/pull/216) on 2026-11-01. Best merged before any other open PR, so their CI can finish.

**Title:** CI: Remove the retired windows-2019 job and the unusable PowerShell 6.x/7.0 container jobs

**Description:**

Running the workflow on this repository's pull requests (in a fork) shows three problems that have nothing to do with the code:

- `test-powershell-5-1-windows-2019` stays queued forever because GitHub retired the `windows-2019` runner image (June 2025). The whole workflow run never completes.
- `test-powershell-6-0`, `6-1`, `6-2` and `7-0` run in Ubuntu 16.04/18.04 containers. `actions/checkout` fails there before any test runs: `/__e/node24/bin/node: ... GLIBC_2.28 not found`, because GitHub now forces a newer Node runtime on JavaScript actions.
- `actions/checkout@v3` is outdated.

This PR removes those five jobs (and their entries in `needs:`) and updates `actions/checkout` to `v4`. Windows PowerShell 5.1 on `windows-2022` and PowerShell 7.1 to 7.4 remain and pass. The tests are not changed.

The trade-off: PowerShell 6.x and 7.0 are no longer exercised by CI. They could be added back with a newer container image or a different way to install them, if wanted.
