# Clear-Text Footnotes

Clear-Text Footnotes is a small text transformation for writing simple footnotes directly inside plain Unicode text.

Write an inline source annotation like this:

```text
Wikipedia ƒ(https://wikipedia.org)
```

Render it into this, with the space before the marker collapsed:

```text
Wikipedia¹

---
1) https://wikipedia.org
```

The same system can also reverse rendered footnotes back to source annotations, and it can normalize mixed text that contains both existing rendered footnotes and newly added source annotations.

## Render example

Input:

```text
When I was a childƒ(born in 1968), I loved sci-fi and the first tv-show, that I remember, is Space: 1999ƒ(https://en.wikipedia.org/wiki/Space:_1999), that aired on the Danish National Broadcast Serviceƒ(DR, https://dr.dk, and https://lex.dk/DR).
```

Expected rendered text:

```text
When I was a child¹, I loved sci-fi and the first tv-show, that I remember, is Space: 1999², that aired on the Danish National Broadcast Service³.

---
1) born in 1968
2) https://en.wikipedia.org/wiki/Space:_1999
3) DR, https://dr.dk, and https://lex.dk/DR
```

## Reverse example

Input:

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

## Mixed input example

If rendered text is edited and a new source annotation is added, rendering should not append a second footnote block. Instead, the existing rendered footnotes are reversed first, then all source annotations are rendered again.

Mixed input:

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

## Core rules

A valid source footnote annotation:

```text
<text>ƒ(<footnote text>)<text>
```

renders as:

```text
<text><superscript number><text>
```

and the footnote text is collected at the end:

```text
---
<number>) <footnote text>
```

A rendered inline reference and matching block entry:

```text
<text>¹<text>

---
1) <footnote text>
```

reverses as:

```text
<text>ƒ(<footnote text>)<text>
```

Numbers are assigned in encounter order, starting at `1`. Inline numbers use Unicode superscript digits, for example `¹`, `²`, `³`, and `¹⁰`. Horizontal whitespace before the `ƒ(` marker is allowed in source text and collapsed in rendered output.

## Design intent

This project defines the behaviour of the text transformations, not a specific implementation. The same rules can later be implemented in any suitable host, for example:

- a command-line tool,
- an editor extension,
- a web application,
- a reusable library.

See `SPEC.md` for the normative behaviour and portable test scenarios.
