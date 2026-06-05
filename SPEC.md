# SPEC: Clear-Text Footnotes

## Purpose

Clear-Text Footnotes is a text transformation system. It accepts one Unicode clear-text document and supports two core transformations:

- **Render**: convert source footnote annotations `ƒ(...)` into superscript references and append a collected rendered footnote block.
- **Reverse**: convert rendered footnotes back into source footnote annotations and remove the rendered footnote block.

The render transformation also supports mixed input. If the input contains both rendered footnotes and newly added source annotations, the system first reverses the rendered footnotes to source annotations and then renders all source annotations into one coherent rendered result.

The specification is independent of implementation language and invocation style. The same rules can be implemented in a CLI tool, editor extension, library function, web application, or another host environment.

## Transformation interface

The core transformations are conceptual operations:

```text
render(input text) -> transformed text | transformation error
reverse(input text) -> transformed text | transformation error
```

Requirements:

- The input is treated as one complete text value.
- A successful transformation returns one complete transformed text value.
- A failed transformation returns a transformation error and no partial transformed text.
- Host environments decide how input is collected, how the desired action is selected, how output is displayed or stored, and how errors are reported.

## Source footnote syntax

A source footnote annotation starts with the literal marker:

```text
ƒ(
```

and ends at the next unmatched closing parenthesis on the same line.

Examples:

```text
Outlookƒ(https://outlook.com)
```

```text
See thisƒ(RFC 3986 (URLs))
```

The second example is valid because the inner parentheses are balanced.

## Source footnote text

Source footnote text is the content inside `ƒ(` and its matching `)`.

Rules:

- The text must be single-line.
- The text is trimmed before validation and before being listed in the rendered footnote block.
- Empty or whitespace-only footnote text is invalid.
- Balanced parentheses are allowed inside footnote text.
- A line break before the matching closing parenthesis makes the annotation malformed.

## Render transformation

The render transformation converts source footnote annotations into rendered inline references and a rendered footnote block.

Each valid source annotation is replaced by an inline reference.

- Replace only the exact annotation from `ƒ(` through its matching `)`.
- Preserve all surrounding whitespace exactly.
- Number footnotes in encounter order, starting at 1.
- Render inline references using Unicode superscript digits.
- Multi-digit references are rendered digit by digit.

Superscript digit mapping:

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
wordƒ(note)  -> word¹
word ƒ(note) -> word ¹
wordƒ(note). -> word¹.
10           -> ¹⁰
23           -> ²³
```

## Rendered footnote block

If at least one source footnote annotation is rendered, append a rendered footnote block to the end of the transformed body.

Before appending:

- Trim trailing whitespace/newlines from the transformed body.
- Insert exactly two newlines.
- Insert `---`.
- Insert one newline.
- List one footnote per line as `{number}) {footnote text}`.

Example input:

```text
Jeg skrev en mail i Outlookƒ(https://outlook.com) og sendte til en privatpersonƒ(private address).
```

Expected rendered text:

```text
Jeg skrev en mail i Outlook¹ og sendte til en privatperson².

---
1) https://outlook.com
2) private address
```

If no source footnote annotations exist and no rendered footnotes are detected, the rendered text is exactly the original input text.

## Reverse transformation

The reverse transformation converts rendered footnotes back into source footnote annotations.

A rendered footnote consists of:

- one rendered inline reference in the body, such as `¹`, `²`, or `¹⁰`, and
- one matching entry in the rendered footnote block, such as `1) text`, `2) text`, or `10) text`.

Reverse transformation rules:

- Detect a rendered footnote block at the end of the input.
- Remove that rendered footnote block from the output.
- Replace each rendered inline reference with `ƒ(<footnote text>)`, using the text from the matching block entry.
- Trim footnote text from block entries before inserting it into source annotations.
- Preserve body text surrounding each replaced inline reference.
- Leave existing source footnote annotations unchanged. The reverse pass ignores all `ƒ(...)` instances, including any superscript digits inside them.
- If no rendered footnote block is detected, reverse returns the input unchanged.

Example rendered input:

```text
When I was a child¹, I loved sci-fi and the first tv-show, that I remember, is Space: 1999², that aired on the Danish National Broadcast Service³.

---
1) born in 1968
2) https://en.wikipedia.org/wiki/Space:_1999
3) DR, https://dr.dk, and https://lex.dk/DR
```

Expected reversed text:

```text
When I was a childƒ(born in 1968), I loved sci-fi and the first tv-show, that I remember, is Space: 1999ƒ(https://en.wikipedia.org/wiki/Space:_1999), that aired on the Danish National Broadcast Serviceƒ(DR, https://dr.dk, and https://lex.dk/DR).
```

## Rendered footnote detection

Rendered footnote detection should be conservative to avoid treating ordinary superscript characters or horizontal rules as footnotes.

A rendered footnote block is detected when all of these are true:

1. The input has a final block introduced by a line containing exactly `---`, ignoring surrounding whitespace on that separator line.
2. The separator is followed only by numbered footnote lines and optional trailing whitespace.
3. Numbered footnote lines have the form `{number}) {footnote text}`.
4. Numbers are positive decimal integers, contiguous, and start at `1`.
5. Each footnote text is non-empty after trimming.
6. The body before the separator contains each corresponding rendered inline reference exactly once outside existing source footnote annotations.
7. The rendered inline references appear in the body in increasing encounter order.

A candidate block that does not meet these requirements is not a valid rendered footnote block for reverse transformation. If an action depends on reversing that block, the transformation should fail with a clear error rather than silently losing content.

Inline references are parsed by converting consecutive superscript digits to decimal digits:

```text
¹   -> 1
²   -> 2
¹⁰  -> 10
²³  -> 23
```

## Mixed input normalization

Mixed input contains both source footnote annotations and rendered footnotes.

The render transformation handles mixed input by normalizing it:

1. Detect whether a rendered footnote block is present.
2. If a rendered footnote block is present, apply the reverse transformation to the rendered footnotes. Existing source footnote annotations are ignored by reverse and remain unchanged.
3. Render the resulting source text normally.

This produces one coherent rendered footnote block and renumbers all footnotes in encounter order.

Example mixed input:

```text
When I was a child¹, I loved sci-fiƒ(especially optimistic space opera).

---
1) born in 1968
```

Expected rendered text:

```text
When I was a child¹, I loved sci-fi².

---
1) born in 1968
2) especially optimistic space opera
```

## Escaping

The core transformation has no escape syntax. Every literal `ƒ(` is treated as a footnote marker during rendering.

To mention the marker literally, write it in a way that avoids the exact sequence, such as `ƒ (...)`.

## Malformed input

Malformed source footnotes include annotations that start with `ƒ(` but:

- have no valid matching closing parenthesis,
- contain a line break before the matching closing parenthesis, or
- contain only whitespace after trimming.

Malformed rendered footnotes include candidate rendered footnote blocks that cannot be mapped safely back to body references.

On malformed input, the transformation must fail and produce no partial transformed text.

A transformation error should include enough information for a host environment to present a clear diagnostic. Line and column are recommended when practical.

Example diagnostic text:

```text
error: unclosed footnote marker at line 1, column 15
```

## Non-goals for the core system

These concerns are intentionally outside the core specification:

- Specific programming language or runtime.
- Specific executable name, command syntax, editor command, web route, or API shape.
- File reading, writing, in-place editing, clipboard handling, or network transport.
- Packaging, installation, version printing, or help text.
- Escape syntax.
- Rich footnote formatting, nested footnote blocks, or multiple independent rendered footnote blocks.

## Portable test scenarios

These scenarios define expected core behaviour and can be implemented in any test framework.

### Scenario 1: render no footnotes

Input:

```text
Plain text without markers.
```

Expected render result:

```text
Plain text without markers.
```

### Scenario 2: render one source footnote

Input:

```text
Outlookƒ(https://outlook.com) is mentioned.
```

Expected render result:

```text
Outlook¹ is mentioned.

---
1) https://outlook.com
```

### Scenario 3: render multiple source footnotes in encounter order

Input:

```text
Aƒ(first) Bƒ(second) Cƒ(third)
```

Expected render result:

```text
A¹ B² C³

---
1) first
2) second
3) third
```

### Scenario 4: render preserves surrounding whitespace

Input:

```text
word ƒ(note) end
```

Expected render result:

```text
word ¹ end

---
1) note
```

### Scenario 5: render trims footnote text in the block

Input:

```text
wordƒ(  note with spaces  )
```

Expected render result:

```text
word¹

---
1) note with spaces
```

### Scenario 6: render supports balanced parentheses inside footnote text

Input:

```text
See thisƒ(RFC 3986 (URLs)).
```

Expected render result:

```text
See this¹.

---
1) RFC 3986 (URLs)
```

### Scenario 7: render superscript references above 9

Input:

```text
ƒ(n1) ƒ(n2) ƒ(n3) ƒ(n4) ƒ(n5) ƒ(n6) ƒ(n7) ƒ(n8) ƒ(n9) ƒ(n10)
```

Expected render result:

```text
¹ ² ³ ⁴ ⁵ ⁶ ⁷ ⁸ ⁹ ¹⁰

---
1) n1
2) n2
3) n3
4) n4
5) n5
6) n6
7) n7
8) n8
9) n9
10) n10
```

### Scenario 8: render trims transformed body trailing whitespace before block

Input has trailing spaces after the final transformed body character:

```text
wordƒ(note)   
```

Expected render result:

```text
word¹

---
1) note
```

### Scenario 9: reverse rendered footnotes

Input:

```text
When I was a child¹, I loved sci-fi and the first tv-show, that I remember, is Space: 1999², that aired on the Danish National Broadcast Service³.

---
1) born in 1968
2) https://en.wikipedia.org/wiki/Space:_1999
3) DR, https://dr.dk, and https://lex.dk/DR
```

Expected reverse result:

```text
When I was a childƒ(born in 1968), I loved sci-fi and the first tv-show, that I remember, is Space: 1999ƒ(https://en.wikipedia.org/wiki/Space:_1999), that aired on the Danish National Broadcast Serviceƒ(DR, https://dr.dk, and https://lex.dk/DR).
```

### Scenario 10: reverse ignores existing source annotations

Input:

```text
A¹ and Bƒ(new source note).

---
1) old rendered note
```

Expected reverse result:

```text
Aƒ(old rendered note) and Bƒ(new source note).
```

### Scenario 11: render mixed input normalizes to one block

Input:

```text
A¹ and Bƒ(new source note).

---
1) old rendered note
```

Expected render result:

```text
A¹ and B².

---
1) old rendered note
2) new source note
```

### Scenario 12: reverse without rendered footnote block is unchanged

Input:

```text
Aƒ(source note) and plain text.
```

Expected reverse result:

```text
Aƒ(source note) and plain text.
```

### Scenario 13: render unclosed source marker fails

Input:

```text
wordƒ(note
```

Expected render result: transformation error, no transformed text.

### Scenario 14: render source marker interrupted by newline fails

Input:

```text
wordƒ(note
still note)
```

Expected render result: transformation error, no transformed text.

### Scenario 15: render empty source footnote fails

Input:

```text
wordƒ()
```

Expected render result: transformation error, no transformed text.

### Scenario 16: render whitespace-only source footnote fails

Input:

```text
wordƒ(   )
```

Expected render result: transformation error, no transformed text.

### Scenario 17: reverse malformed rendered block fails

Input:

```text
A¹.

---
2) starts at two
```

Expected reverse result: transformation error, no transformed text.
