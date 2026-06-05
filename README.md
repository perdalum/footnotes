# Clear-Text Footnotes

Clear-Text Footnotes is a language- and invocation-agnostic specification for handling simple footnotes in plain Unicode text.

It defines how to:

- render inline source footnote annotations into superscript references and a collected footnote block,
- reverse rendered footnotes back into source annotations,
- normalize mixed input that contains both already-rendered footnotes and newly added source annotations.

The project is intentionally not tied to any implementation language, runtime, user interface, or invocation style. The same behaviour can later be implemented as a CLI tool, an editor extension, a web application, a reusable library, or another host integration.

## Source annotations

Authors write footnotes inline with the marker `ƒ(` and a closing `)`:

```text
Wikipedia ƒ(https://wikipedia.org)
```

Rendering collapses the space before the marker, converts the source annotation into a superscript reference, and adds a collected block:

```text
Wikipedia¹

---
1) https://wikipedia.org
```

## Render transformation

The conceptual render operation is:

```text
render(input text) -> transformed text | transformation error
```

Render rules include:

- source annotations use `ƒ(<footnote text>)`,
- horizontal whitespace before the `ƒ(` marker is allowed and collapsed in rendered output,
- footnote text must be non-empty and single-line,
- balanced parentheses are allowed inside footnote text,
- footnotes are numbered in encounter order, starting at `1`,
- inline references use Unicode superscript digits, for example `¹`, `²`, `³`, and `¹⁰`,
- the rendered footnote block is appended after the transformed body,
- malformed source annotations fail the transformation without partial output.

## Reverse transformation

The conceptual reverse operation is:

```text
reverse(input text) -> transformed text | transformation error
```

Reverse converts rendered footnotes back to source annotations:

```text
When I was a child¹.

---
1) born in 1968
```

becomes:

```text
When I was a childƒ(born in 1968).
```

Reverse removes the rendered footnote block and replaces each matching superscript reference with `ƒ(<footnote text>)`. Existing source annotations are ignored and left unchanged by the reverse pass.

## Mixed input normalization

Rendered text may later be edited with new source annotations. For example:

```text
When I was a child¹, I loved sci-fiƒ(especially optimistic space opera).

---
1) born in 1968
```

Rendering mixed input first reverses the existing rendered footnotes, then renders all source annotations again. The result is one coherent rendered block:

```text
When I was a child¹, I loved sci-fi².

---
1) born in 1968
2) especially optimistic space opera
```

## Documents

- `PITCH.md` gives a concise explanation and examples.
- `CONTEXT.md` defines the domain language and resolved decisions.
- `SPEC.md` is the normative specification, including portable test scenarios.
- `IMPLEMENTATION.md` gives implementation-neutral design notes and algorithms.
- `CHANGELOG.md` records project changes.

## Versioning

This repository uses documented release versions for the specification. Version `1.0.0` is the initial baseline for the Clear-Text Footnotes render, reverse, and mixed-input behaviour.

## Implementation status

This repository currently contains specification and design documents only. It does not provide an implementation, package, executable, editor extension, or web application.
