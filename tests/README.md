# OireMark Conformance Tests

These tests are part of the OireMark specification. A conforming parser passes all of them.

The test files are themselves OireMark documents (`.om`), one per area of the spec, numbered after the spec's sections (`03-headings.om` covers Section 3). Files numbered `00` cover the Introduction: encoding, line endings and case sensitivity.

## Case Format

Each test case starts with a level-3 heading with a custom ID. The ID is the case's name and is unique across the whole suite.

```
h3. Heading with custom ID {#headings-custom-id}
Optional explanation in ordinary prose.
[code om]
h2. My Section {#custom-id}
[/code]
[code html]
<h2 id="custom-id">My Section</h2>
[/code]
```

A case contains, in this order:

1. **Options** (optional), as a short definition list. Each `; key` line is followed by one `: value` line.
   - `; profile` — the parser profile to use: `inline`, `basic`, `standard` or `full`. Default: `full`.
   - `; backlink` — the configured footnote back-link symbol. Default: `↩`.
2. **Input**, exactly one of:
   - `[code om]` — the OireMark input, as UTF-8 text.
   - `[code hex]` — the input as hexadecimal bytes separated by whitespace. Used when the test depends on bytes a text file can't hold reliably: a BOM, invalid UTF-8, CR or CRLF line endings, trailing spaces, tabs.
3. **Expected result**, exactly one of:
   - `[code html]` — the expected HTML output.
   - `[code error]` — a single line: the error ID, a space, and the 1-based line number, for example `unclosed-block 3`. The parser must fail with that first error.
4. **Expected warnings** (optional): `[code warnings]`, one `id line` pair per line. Each listed warning must be emitted. Extra warnings are allowed, because the spec lets parsers emit optional ones. A case without a warnings block asserts nothing about warnings.

A block's content is its lines joined with LF; the newline before the closing line is not part of it. (The spec ignores newlines at the end of a document anyway.)

When an input contains a `[/code]` line, the case uses the fenced form instead (```` ```om ```` … ```` ``` ````, with a longer fence if the input itself contains backtick fences).

Everything else in a file (the `[meta]` block, `h1.` and `h2.` headings, prose) is for readers and is ignored by test runners.

## Extracting Cases

A runner doesn't need an OireMark parser to read the suite. It only needs to recognize:

- `h3. ... {#id}` lines (start of a case),
- `; key` and `: value` lines (options),
- `[code X]` … `[/code]` blocks, and fenced blocks whose closing fence has at least as many backticks as the opening one. Their content is taken verbatim.

## Comparing HTML

Expected output is a fragment (the content of `<body>`), pretty-printed for reading. Compare as follows:

1. Parse both the expected and the actual output as HTML5 fragments.
2. Outside `<pre>`, in every text node, delete each run of whitespace that contains a newline. Then drop text nodes that became empty. (Pretty-printing whitespace always contains a newline; meaningful spaces in the expected output never do.)
3. Inside `<pre>`, compare text exactly.
4. Compare the trees: element names, attribute sets (order ignored, values exact), and text. Character references are compared after decoding, so `&nbsp;` and a literal U+00A0 are equal.

`[meta]` only affects the document wrapper (`<html lang>`, `<title>`), which fragment output doesn't have. A document containing only `[meta]` produces empty output.
