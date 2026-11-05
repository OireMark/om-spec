# OireMark Parser Design

**Status:** Guide for implementers. Not normative: the [specification](../specs/v1.0.0.md) and the [conformance tests](../tests/README.md) are. If this document disagrees with them, this document is wrong.

This is the common design for every OireMark implementation (PHP, .NET, JavaScript, Python, Kotlin, C++). When all of them share one shape, porting is mostly mechanical, a bug found in one implementation can be looked for in the others, and every port can be judged against the same test suite.

## Goals

1. **Conformance.** An implementation is done when it passes every case in `tests/`.
2. **One architecture.** The same stages, node types and names in every language, adapted only to local naming conventions.
3. **Strictness without guessing.** The first error stops parsing and reports a stable ID and a line number.
4. **Linear time.** Parsing time grows with the input size and never worse. OireMark is meant for user input on forums and chats, so a crafted document must not be able to stall a server.
5. **One place for escaping.** Only the HTML renderer writes HTML, and it escapes everything it writes.

## The Key Insight: Everything Inline Is Line-Local

Every inline construct in OireMark lives within one line: formatting markers, code spans, links, images, footnote references and inline `[lang]` spans all pair or close on the same line, or are literal.

This allows a single top-to-bottom pass. Each line is classified as block syntax, and its inline content is parsed right away. Errors are therefore found in exactly the order the spec defines ("first found when reading from top to bottom", spec Section 22). A design that parsed all blocks first and all inlines afterwards would report a block error on line 50 before an inline error on line 2, and fail the tests.

## Pipeline

```
bytes ─▶ 1 Decode ─▶ 2 Lines ─▶ 3 Scan (blocks + inlines, one pass) ─▶ 4 Finish ─▶ 5 IDs ─▶ 6 Resolve ─▶ Document
                                                                                                   │
                                                                                    7 Render ◀─────┘ ─▶ HTML
```

### 1. Decode

- If the input is bytes, validate UTF-8. On failure, raise `invalid-utf8` with the line of the first invalid byte (count LF, CR and CRLF the same way stage 2 does).
- Strip a BOM at the very start. U+FEFF anywhere else is ordinary text.
- Languages whose strings are UTF-16 (C#, JavaScript, Kotlin): a lone surrogate is the equivalent of invalid UTF-8 and raises `invalid-utf8` on its line.

### 2. Lines

- Split on LF, CRLF and CR. Number lines from 1.
- Drop trailing blank lines at the end of the document.
- Do **not** trim trailing whitespace or remove comments here. Both depend on whether a line is inside a code block, which only the scanner knows.

### 3. Scan

One pass over the lines with a stack of open containers (`[more]`, `[quote]`, `[note]`, `[def]`, block `[lang]`, `[fn]`) and at most one open leaf (paragraph, list, definition list, table, short quote).

For each line:

1. **Inside a code block or comment?** Check only for its closing line (`[/code]` or `[/comment]` alone on the line, leading whitespace allowed, or a fence at least as long as the opening one). Code content is kept byte for byte; comment content is dropped.
2. **Otherwise**, trim trailing whitespace, then classify the line (see the table below).
3. **Check the profile** for the construct found. A disallowed feature raises `feature-not-in-profile` here.
4. **Check nesting** against the innermost container's allow-list (spec Section 21).
5. **Apply it:** continue the open leaf, close it, open a new block, or close a container.
6. **Parse the inline content** of the line immediately (see Inline Parsing).

#### Line Classification

Leading whitespace is ignored for all of these. Test in this order; the first match wins.

| Order | Line looks like | Classified as |
|-------|-----------------|---------------|
| 1 | Empty | Blank line |
| 2 | `[comment]` | Comment start |
| 3 | `[/tag]` alone | Closing tag (`mismatched-tag` if it closes nothing or the wrong block) |
| 4 | `[code lang]` or a fence of 3+ backticks | Code block start |
| 5 | Other block tag alone: `[meta]`, `[quote ...]`, `[note ...]`, `[more ...]`, `[def ...]`, `[fn]`, `[toc ...]`, `[lang xx]` | Container or block start |
| 6 | Reserved tag (`[alert]`, `[verse]`, ...) | `reserved-tag` error |
| 7 | `h1.` to `h6.` followed by a space | Heading |
| 8 | Only 3 or more hyphens | Horizontal rule |
| 9 | List marker path followed by a space and content | List item |
| 10 | `; ` or `: ` | Definition term or definition |
| 11 | Starts with `|` | Table row |
| 12 | `> ` or `>` alone | Short quote line |
| 13 | Anything else | Text |

Then, for text lines and inline content, check whether a block tag appears *after* other content on the line: that is `block-tag-mid-line`. (`[/lang]` is the exception; see spec Section 17.)

A text line continues the open paragraph, list item, definition or short quote. Any other classification closes the open leaf first. A blank line closes it as well.

#### When Errors Become Certain

The scanner raises an error on the line where it becomes certain, which is not always the line it reports:

| Error | Becomes certain | Reports |
|-------|-----------------|---------|
| `unclosed-block` | At the end of the enclosing container, or of the document | The opening line |
| `term-without-definition` | At the line after the term that is not a term or definition | The term's line |
| `footnote-block-misplaced` | At the first content after `[/fn]` | That content |
| `invalid-span` (row span past the end) | When the table ends | The row with the span |
| `duplicate-id` | At the second heading with that custom ID | That heading |

All other errors are certain on the line they report. If two errors become certain on the same line, report the one listed first in the spec's error table.

#### Meta and Profiles

`[meta]` must be the first block (only blank lines and comments may come before it). Read it before applying any profile. The effective profile is then the **intersection** of the parser's configured feature set and the declared profile. If the declared profile adds features the parser doesn't allow, emit `profile-mismatch` (optional) on the `profile:` line. With the four standard profiles, intersection gives exactly the precedence rules in spec Section 23. With custom feature sets, it's the natural generalization.

In the `inline` profile, any newline is `feature-not-in-profile` on the line after the break.

### 4. Finish

At the end of the document:

- Raise `unclosed-block` for the innermost container still open.
- Close the open leaf.

### 5. IDs

All IDs share one namespace (spec Section 3).

1. **Reserve** every custom heading ID and every footnote ID: `fnN` for each definition, and `fnrefN`, `fnrefN-2`, ... for each reference. Footnote reference IDs are numbered in document order.
2. **Generate**, in document order, the IDs for headings without a custom ID (slugs), titled notes (`note-N`) and titled tables of contents (`toc-N`). If a candidate is taken, try `-2`, `-3`, ... until one is free.

Slugs are made from the heading's inline tree, not its source text: take the text of every node, use link text but not URLs, and keep the text of code spans.

Two platform traps when making slugs:

- **Lowercasing must ignore the locale.** Use `mb_strtolower` (PHP), `ToLowerInvariant` (C#), `lowercase()` with `Locale.ROOT` (Kotlin), `toLowerCase()` (JS) and `str.lower()` (Python), never a call that depends on the machine's language setting.
- **Normalize to NFC first**, with `Normalizer::normalize` (PHP intl), `string.Normalize()` (C#), `String.prototype.normalize('NFC')` (JS), `unicodedata.normalize` (Python) or `java.text.Normalizer` (Kotlin).

Unicode character tables differ slightly between platform versions. For letters added in recent Unicode versions, implementations can disagree until their platforms catch up. The tests avoid such characters.

### 6. Resolve

- **Footnotes:** match references to definitions. A reference with no definition becomes literal text again and emits `footnote-undefined`. A definition with no reference emits `footnote-unused` and gets no back link. Sort definitions by label; give `value` to any label that doesn't follow the previous one.
- **Tables of contents:** collect every heading in the document (including those inside `[more]`), filter by range, and nest each heading under the nearest preceding heading of a higher level. Reduce links in heading text to their text.
- **Warnings** collected during the scan are kept in document order.

### 7. Render

A separate HTML renderer walks the finished tree. It is the only code that produces HTML, and it escapes all text and attribute values (`&`, `<`, `>`, `"`). Because of the tree, other renderers (plain text, EPUB, terminal) can be added later without touching the parser.

## Inline Parsing

Inline parsing works on one line's content at a time (a paragraph line, a list item line, a heading's text, a table cell, a title). Follow the spec's Pairing Algorithm (Section 5) step by step:

1. **Code spans.** Scan for backtick runs. A run opens a span that closes at the next run of the same length on the same line. A backslash-escaped backtick outside a span can't open one. The rest of the line is split into code tokens and text segments.
2. **Escapes.** In the text segments, turn `\` + ASCII punctuation into literal-character tokens that can never act as markers or brackets. Turn `\ ` into a non-breaking space. Any other backslash is literal.
3. **Bracketed constructs.** Find links, images, footnote references and inline `[lang]` spans, matching brackets while respecting escapes. Apply the URL termination rules (spec Section 7). The text inside a link or `[lang]` span is a separate scope: run step 4 on it on its own.
4. **Markers.** One left-to-right pass with a stack of open spans:
   - Recognize a marker only as a run of exactly its defined length (`__`, `==`, or a single character). Longer runs are literal.
   - Check the boundary conditions using the neighboring characters. A code span, link, image or footnote reference next to a marker counts as punctuation.
   - Try closing first, then opening, exactly as the spec describes, including the overlap rule and the no-empty-content rule.
   - Any openers left at the end of the scope are literal.

Each step is a single pass, so inline parsing is linear in the length of the line.

## Tables

Keep a grid while reading rows. For each row, keep an array of "remaining row span" counters per column.

```
for each written cell in the row:
    skip columns whose counter is > 0 (filled from above)
    place the cell at the next free column, occupying colspan columns
    set those columns' counters to rowspan
after the row:
    width = columns filled from above + sum of colspans written
    if width != first row's width: uneven-table-row
    decrement every counter > 0
at the end of the table:
    if any counter is still > 0: invalid-span, reported on the row with that span
```

The first row sets the width. A header row is the first row if all of its cells are `||`. Alignment comes from header cells and applies to every column a header cell covers. Body cells take the alignment of the column they start in.

## Data Model

Every node records the line it starts on.

**Document:** effective profile, meta values, block list, footnote definitions, warnings.

**Block nodes:**

| Node | Holds |
|------|-------|
| `Heading` | level, custom ID or none, generated ID, inline content |
| `Paragraph` | inline content (lines separated by `LineBreak`) |
| `List` | ordered or not, start number, type hint, items |
| `ListItem` | explicit value or none, inline content, nested lists |
| `DefinitionList` | entries: terms, short definitions, long `[def]` definitions |
| `Table` | header row or none, rows of cells (header or data, scope, colspan, rowspan, alignment, inline content) |
| `CodeBlock` | language or none, literal text |
| `Quote` | author (inline content) or none, source URL or none, blocks |
| `ShortQuote` | paragraphs |
| `More` | title (inline content), blocks |
| `Note` | title (inline content) or none, ID, blocks |
| `LangBlock` | language code, blocks |
| `TableOfContents` | level range, title or none, ID |
| `HorizontalRule` | — |

Footnote definitions are kept on the document, not in the block list.

**Inline nodes:** `Text`, `LineBreak`, `Strong`, `Emphasis`, `Underline`, `Deleted`, `Inserted`, `Subscript`, `Superscript`, `Keyboard`, `Highlight`, `Code`, `Link` (URL, mention flag, content), `Image` (source, plain alt text, width, height), `FootnoteReference` (label, plus its source text in case it turns out to be undefined), `LangSpan` (code, content).

## Errors and Warnings

- **Error:** an exception type (`OireMarkException` or the local equivalent) with `id` (the spec's string ID, for example `unclosed-block`), `line`, and a human-readable `message`. Expose the ID as a string or as an enum whose values are those exact strings.
- **Warning:** a value with the same three fields, collected in a list on the result.
- Messages are for people. Never compare them in tests: tests compare only the ID and the line.

## Public API

Use the same names everywhere, adapted to each language's conventions:

| Concept | PHP | C# | JavaScript | Python | Kotlin |
|---------|-----|----|------------|--------|--------|
| Parser | `OireMarkParser` | `OireMarkParser` | `OireMarkParser` | `OireMarkParser` | `OireMarkParser` |
| Profile | `OireMarkProfile::Basic` | `OireMarkProfile.Basic` | `OireMarkProfile.Basic` | `OireMarkProfile.BASIC` | `OireMarkProfile.BASIC` |
| Feature | `OireMarkFeature::Tables` | `OireMarkFeature.Tables` | `OireMarkFeature.Tables` | `OireMarkFeature.TABLES` | `OireMarkFeature.TABLES` |
| Error | `OireMarkException` | `OireMarkException` | `OireMarkError` | `OireMarkError` | `OireMarkException` |

Core operations:

- **Create** a parser with a profile or a custom feature set (default: `full`), plus options: the footnote back-link symbol (default `↩`), and any off-by-default extras the spec allows (typographic replacement, opening web links in a new window).
- **`parse(input)`** returns the `Document` tree, or raises the first error.
- **`toHtml(input)`** is a shortcut that parses and renders, returning the HTML and the warnings.
- **`HtmlRenderer.render(document)`** renders a tree.

The parser keeps no state between calls, so one instance can be reused and shared across threads.

## Test Runner

Every implementation ships a runner for the shared suite:

1. Read the `.om` files in `tests/` and extract the cases as `tests/README.md` describes. This needs only line matching, not an OireMark parser.
2. For each case, configure the parser from the options (`profile`, `backlink`), parse the input (decoded from hex if needed), and compare: the HTML per the README's comparison rules, or the error ID and line exactly. Check that every listed warning was emitted.
3. Report each failure with the case ID, so it can be found in the suite.

Include the spec repository as a pinned dependency (a git submodule, or a copy at a recorded commit). An implementation then states exactly which version of the spec and tests it passes.

## Security and Limits

- **No backtracking regular expressions** over user input. Simple anchored patterns on a single short token are fine. Patterns with nested quantifiers over a whole line or document are not.
- **Nesting depth:** containers and list levels together may nest at most 16 deep; the 17th level is a `nesting-too-deep` error (spec Section 21). Check the depth when opening a container or list level. This also bounds recursion in the renderer.
- **Input size** is left to the calling application. Document this in each implementation's README.
- **Escaping:** only the renderer writes HTML, and it escapes every text node and attribute value.
- **Unsafe URLs:** check every URL (links, self-links, images, quote sources) for the `javascript:`, `vbscript:` and `data:` schemes in any letter case, and for ASCII control characters, and raise `unsafe-url` (spec Section 7). Do this check in the parser, not the renderer, so the error is found in document order.

## Porting Checklist

Build in this order. Each step unlocks a group of test files.

1. Decode, lines, and the test runner (`00`)
2. Paragraphs, escapes, inline markers and code spans (`04`, `05`, `06`)
3. Headings and IDs (`03`)
4. Block scanner and containers: notes, quotes, `[more]`, code blocks, comments, horizontal rules (`02`, `10`, `12` to `14`, `19` to `21`)
5. Lists and definition lists (`09`, `15`)
6. Links and images (`07`, `08`)
7. Tables (`11`)
8. Meta, `[lang]`, footnotes, table of contents (`01`, `16` to `18`)
9. Profiles and the remaining parsing rules (`22`, `23`)

## Open Points for the Spec

None at the moment. When writing an implementation turns up a gap in the spec, record it here until the spec settles it, then add a test case for it.
