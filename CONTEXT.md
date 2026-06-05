# Clear-Text Footnotes

A language- and invocation-agnostic text transformation system for simple footnotes in clear text. The system receives a single Unicode text value and can render source footnote annotations, reverse rendered footnotes back to source annotations, or normalize mixed input that contains both forms.

The same behaviour can later be implemented in different environments, for example as a command-line tool, an editor extension, or a web application. This document describes the domain language and rules, not any particular programming language, framework, executable name, file layout, or user interface.

## Language

**Input text**:
A single Unicode clear-text document to transform.
_Avoid_: file, buffer, stdin, textarea

**Source footnote annotation**:
An inline footnote written as `ƒ(<footnote text>)`.
_Avoid_: raw note, author note

**Footnote marker**:
The literal opening sequence `ƒ(` that starts a source footnote annotation.
_Avoid_: trigger, sigil, command syntax

**Footnote text**:
The non-empty single-line trimmed content inside a source footnote annotation or inside a rendered footnote block entry.
_Avoid_: note body, annotation text

**Rendered inline reference**:
A superscript number in the body text that refers to an entry in a rendered footnote block.
_Avoid_: citation, exponent

**Rendered footnote block**:
The collected section at the end of rendered text, introduced by `---` and containing numbered footnote texts.
_Avoid_: appendix, references

**Rendered footnote**:
The pair formed by one rendered inline reference and its matching rendered footnote block entry.
_Avoid_: citation pair

**Source text**:
Text that contains source footnote annotations and no rendered footnote block to be interpreted.
_Avoid_: draft file, unprocessed buffer

**Rendered text**:
Text that contains rendered inline references and a rendered footnote block.
_Avoid_: final file, processed buffer

**Mixed text**:
Text that contains at least one source footnote annotation and at least one rendered footnote block.
_Avoid_: partially processed file

**Malformed source footnote**:
A source footnote annotation that starts with a footnote marker but lacks a valid matching closing parenthesis, contains a line break before the matching closing parenthesis, or contains only whitespace after trimming.
_Avoid_: bad note, invalid markup

**Malformed rendered footnote block**:
A candidate rendered footnote block that cannot be mapped safely back to source footnote annotations.
_Avoid_: bad appendix, invalid references

**Render transformation**:
The transformation that converts source footnote annotations into rendered inline references and a rendered footnote block. If rendered footnotes are detected first, they are reversed before rendering so mixed text is normalized.
_Avoid_: forward command, export mode

**Reverse transformation**:
The transformation that converts rendered footnotes back into source footnote annotations and removes the rendered footnote block. Existing source footnote annotations are ignored by the reverse pass and left unchanged.
_Avoid_: undo command, import mode

**Transformation result**:
Either transformed text or a transformation error. A failed transformation does not produce partial transformed text.
_Avoid_: stdout, response payload, buffer mutation

## Relationships

- A **Footnote marker** starts exactly one candidate **Footnote text**.
- A **Malformed source footnote** prevents rendering and produces a transformation error instead of partial output.
- Each valid **Source footnote annotation** produces exactly one **Rendered inline reference** during rendering.
- A **Rendered inline reference** replaces only the exact source annotation from **Footnote marker** through its matching closing parenthesis.
- A **Rendered inline reference** for `10` renders as `¹⁰`, not as a single symbol.
- The **Rendered footnote block** lists **Footnote text** entries in encounter order.
- A **Reverse transformation** replaces rendered inline references with source footnote annotations using the matching block entries.
- A **Reverse transformation** leaves existing source footnote annotations unchanged.
- A **Render transformation** on **Mixed text** first applies the reverse transformation to rendered footnotes, then renders all resulting source footnote annotations into one coherent rendered footnote block.
- If there are no source footnote annotations and no rendered footnotes to transform, the transformed text is exactly the original input text.
- If at least one footnote annotation is rendered, the transformed body and **Rendered footnote block** are separated by exactly two newlines followed by `---` and one newline.

## Example dialogue

> **Developer:** "If the input contains `Outlookƒ(https://outlook.com)`, where does the **Footnote text** end?"
> **Domain expert:** "At the next unmatched closing parenthesis on the same line; balanced parentheses inside the **Footnote text** are allowed."

> **Developer:** "If the input contains rendered footnotes and a new `ƒ(...)` annotation, should rendering append a second block?"
> **Domain expert:** "No. Reverse the rendered footnotes to source annotations first, leave the new source annotation as it is, then render all source annotations into one fresh block."

## Resolved decisions

- `ƒ(` starts a **Footnote marker**.
- The **Footnote text** ends at the next unmatched `)` rather than the first literal `)`.
- Balanced parentheses inside **Footnote text** are allowed.
- If there are no footnotes, output is unchanged.
- If there are footnotes, the **Rendered footnote block** is appended after trimming trailing whitespace from the transformed body.
- Whitespace around a **Footnote marker** is preserved exactly; only the annotation itself is replaced.
- A **Malformed source footnote** is a strict transformation error.
- Empty and whitespace-only **Footnote text** are invalid.
- Valid **Footnote text** is trimmed before being listed in the **Rendered footnote block** and before being inserted by reverse transformation.
- **Footnote text** must be single-line; a line break before the matching closing parenthesis makes the annotation a **Malformed source footnote**.
- The core system transforms one input text into either one output text or one error.
- The core system has no escape syntax; every literal `ƒ(` is treated as a **Footnote marker** during rendering.
- The reverse transformation ignores source footnote annotations; it neither validates nor changes them.
- A render transformation may interpret a rendered footnote block, reverse it to source annotations, and then render all source annotations to normalize mixed text.
- Packaging, invocation style, storage, display, and integration are outside the core system definition.
