# Implementation-Neutral Design Notes: Clear-Text Footnotes

This document describes one portable design for implementing the Clear-Text Footnotes transformations. It is intentionally independent of programming language, runtime, packaging, invocation style, user interface, and storage mechanism.

Use these notes to implement the same core behaviour in any environment, such as a command-line program, editor extension, web application, library, script, or service.

## Core boundary

Keep the core transformations separate from host-environment concerns.

Recommended core operations:

```text
render(input text) -> transformed text | transformation error
reverse(input text) -> transformed text | transformation error
```

The core should:

- accept one complete Unicode text value,
- scan and transform that text according to `SPEC.md`,
- return one complete transformed text value on success,
- return a structured error on malformed input,
- avoid reading files, writing files, printing, mutating editor buffers, touching the DOM, or exiting a process.

The host environment should handle concerns such as:

- where input text comes from,
- where transformed text goes,
- how the user or caller selects render versus reverse,
- how diagnostics are presented,
- whether a user can preview, replace, save, copy, or cancel the result,
- installation, configuration, commands, menus, shortcuts, routes, or process exit codes.

## Suggested internal model

A useful render result shape is:

```text
Result
  outputText: text
  footnotes: list of text
```

A useful rendered footnote block shape is:

```text
RenderedBlock
  bodyText: text
  entries: ordered map from positive integer to text
  separatorRange: position range
  blockRange: position range
```

A useful error shape is:

```text
TransformationError
  kind: unclosed source marker | newline in source footnote | empty source footnote | malformed rendered block | unmatched rendered reference
  line: optional positive integer
  column: optional positive integer
  offset: optional zero-based position
  message: human-readable text
```

The exact data structures and names may vary by implementation language. The important behaviour is that malformed input can be reported clearly and does not produce partial transformed text.

## Render algorithm

The render operation should support plain source input and mixed input.

Algorithm:

1. Detect whether the input ends with a valid rendered footnote block.
2. If a rendered footnote block is detected, apply the reverse operation first. This turns existing rendered footnotes into source footnote annotations while leaving any existing source annotations unchanged.
3. Apply the source-to-rendered algorithm to the resulting text.

This means mixed input is normalized through this conceptual pipeline:

```text
mixed text -> reverse rendered footnotes -> source text -> render source footnotes -> rendered text
```

## Source-to-rendered algorithm

Use an explicit Unicode-character scanner rather than a simple regular expression. Balanced parentheses inside footnote text are allowed, and single-line validation is easier with a scanner.

Algorithm:

1. Initialize:
   - transformed body buffer,
   - footnote list,
   - current text position,
   - optional line/column counters for diagnostics.
2. Scan the input left to right.
3. If the current position is not the exact marker `ƒ(`:
   - append the current character to the transformed body buffer,
   - advance one character,
   - update line/column counters if tracked.
4. If the current position is the exact marker `ƒ(`:
   - if the marker is preceded on the same line by body text, remove any horizontal whitespace between that body text and the marker from the transformed body buffer,
   - do not remove line breaks while performing this collapse,
   - record marker line/column/offset if tracked,
   - parse until the matching unmatched `)` on the same line,
   - maintain parenthesis depth, starting at `1` for the marker's opening `(`,
   - increment depth for `(`,
   - decrement depth for `)`,
   - when depth reaches `0`, the annotation is complete.
5. During annotation parsing:
   - if a line break is encountered before depth reaches `0`, return a malformed-source-footnote error,
   - if the end of input is reached before depth reaches `0`, return an unclosed-source-footnote error.
6. Extract the raw footnote text between the marker and matching close.
7. Trim leading and trailing whitespace from the extracted text.
8. If trimmed text is empty, return an empty-source-footnote error.
9. Append trimmed text to the footnote list.
10. Append the superscript reference for the new footnote number to the transformed body buffer.
11. Continue scanning after the closing `)`.
12. After scanning:
    - if no footnotes were found, return the original input unchanged,
    - otherwise trim trailing whitespace from the transformed body,
    - append `\n\n---\n`,
    - append numbered footnote lines.

## Reverse algorithm

The reverse operation turns a valid rendered footnote block and its inline references back into source footnote annotations.

Algorithm:

1. Detect whether the input ends with a valid rendered footnote block.
2. If no valid rendered footnote block is present, return the input unchanged.
3. Split the input into:
   - body text before the block separator,
   - ordered block entries.
4. Scan the body text left to right for rendered inline references.
5. A rendered inline reference is one or more consecutive superscript digits that decode to a positive integer present in the block entries.
6. Replace each rendered inline reference with `ƒ(<footnote text>)` using the matching block entry.
7. Preserve all body text surrounding each replaced reference.
8. Ignore source footnote annotations while reversing. The reverse scanner does not validate or change `ƒ(...)` instances, and it should not replace superscript digits inside them.
9. Remove the rendered footnote block from the output.
10. Return the reversed body text.

For generated text, inline references should appear once each and in increasing order. Detection should validate that property before reverse transformation to avoid accidental conversion of unrelated superscript characters.

## Rendered footnote block detection algorithm

Detection should be conservative. It is better to reject an ambiguous candidate than to silently remove ordinary text.

Suggested algorithm:

1. Split the input into logical lines while retaining enough position information to reconstruct text ranges.
2. Ignore trailing whitespace-only lines for detection.
3. Find the last line before trailing whitespace whose trimmed content is exactly `---`.
4. Treat the following lines as candidate block entries.
5. Each candidate block entry must match `{number}) {footnote text}`:
   - `{number}` is a positive decimal integer,
   - `)` follows immediately after the number,
   - at least one spacing character separates `)` from the footnote text,
   - footnote text is non-empty after trimming.
6. Entry numbers must be contiguous and start at `1`.
7. Decode rendered inline references in the candidate body, excluding text inside existing source footnote annotations:
   - `⁰` to `0`,
   - `¹` to `1`,
   - `²` to `2`,
   - `³` to `3`,
   - `⁴` to `4`,
   - `⁵` to `5`,
   - `⁶` to `6`,
   - `⁷` to `7`,
   - `⁸` to `8`,
   - `⁹` to `9`.
8. Treat consecutive superscript digits as one candidate number.
9. The body must contain exactly one rendered inline reference outside existing source annotations for each block entry number.
10. Those rendered inline references must appear in increasing encounter order.
11. If all checks pass, the rendered footnote block is valid.
12. If a final candidate block is present but fails validation, report a malformed-rendered-block error when reverse or mixed-input rendering requires interpreting it.

## Whitespace and line break handling

Horizontal whitespace between preceding body text and a source footnote marker is allowed and collapsed during rendering. For example, `Wikipedia ƒ(note)` renders as `Wikipedia¹`, not `Wikipedia ¹`. The collapse applies only to spaces or similar horizontal spacing before the marker; it must not cross a line break.

A line break inside a source footnote annotation makes the annotation malformed. Implementations should recognise the line break conventions that are natural for their text environment. At minimum, `\n` should be treated as a line break. If the environment can receive `\r\n` or `\r`, handle those consistently as line breaks for validation and diagnostic line/column counting.

Rendered footnote block assembly uses `\n` as the canonical line separator in transformed output.

## Superscript rendering

Render the decimal representation of the footnote number digit by digit:

```text
0 -> ⁰
1 -> ¹
2 -> ²
3 -> ³
4 -> ⁴
5 -> ⁵
6 -> ⁶
7 -> ⁷
8 -> ⁸
9 -> ⁹
```

Examples:

```text
1  -> ¹
10 -> ¹⁰
23 -> ²³
```

Reverse transformation decodes the same mapping in the opposite direction.

## Footnote block assembly

When footnotes exist after source-to-rendered scanning:

1. Trim trailing whitespace from the transformed body.
2. Append exactly two newline characters.
3. Append `---`.
4. Append exactly one newline character.
5. Append each footnote as `{number}) {footnote text}`.
6. Separate footnote lines with one newline character.

When no source footnotes exist and no rendered footnotes were normalized, return the original input exactly as received.

## Host integration examples

These examples are illustrative only. They do not change the core specification.

- A command-line host could let a caller choose render or reverse, then read text from a file or standard input and write either transformed text or diagnostics.
- An editor host could render or reverse the current selection or buffer and replace it only after a successful transformation.
- A web host could render or reverse text from a text area and show the result in another view.
- A library host could expose the core transformations as functions to other code.

In every host, the same input text should produce the same transformed text or transformation error for the selected action.

## Portable verification checklist

An implementation is conformant when it satisfies the scenarios in `SPEC.md`, including:

1. Render with no footnotes returns the input exactly unchanged.
2. Render with one source footnote creates one superscript reference and one block entry.
3. Render with multiple source footnotes numbers them in encounter order.
4. Rendered superscript references above 9 render digit by digit.
5. Render collapses horizontal whitespace immediately before a source footnote marker.
6. Render preserves other surrounding whitespace.
7. Render trims footnote text in the block.
8. Render supports balanced parentheses inside source footnote text.
9. Render trims transformed body trailing whitespace before appending the block.
10. Reverse converts rendered inline references and block entries to source annotations.
11. Reverse ignores existing source annotations.
12. Render normalizes mixed input to one coherent block.
13. Reverse without a rendered footnote block returns the input unchanged.
14. Malformed source footnotes fail without partial transformed text.
15. Malformed rendered footnote blocks fail without partial transformed text when interpretation is required.
16. Errors include useful diagnostics, preferably line and column.

## Documentation map

- `PITCH.md` explains the idea with concise examples.
- `CONTEXT.md` defines the domain language and decisions.
- `SPEC.md` defines normative transformation behaviour and portable test scenarios.
- `IMPLEMENTATION.md` gives implementation-neutral design guidance.
