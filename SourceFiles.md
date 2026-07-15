# Source Files

A Coggy source file is a UTF-8 encoded text file with the `.cog` extension. It acts as a physical container contributing declarations to its enclosing package namespace.

## Encoding

* **Character Set**: Source files MUST be encoded in UTF-8.
* **BOM**: A UTF-8 Byte Order Mark (U+FEFF) MUST NOT appear anywhere in a source file. Its presence is a compilation error.

## Whitespace

Whitespace separates tokens and has no semantic significance beyond delimitation.

```EBNF
Whitespace ::= Space | Tab | NewLine
Space      ::= U+0020
Tab        ::= U+0009
NewLine    ::= LineFeed | CarriageReturn | CRLF
LineFeed   ::= U+000A
CRLF       ::= U+000D U+000A
```

* **Normalization**: A CarriageReturn immediately followed by a LineFeed (`\r\n`) is normalized to a single LineFeed (`\n`). A lone CarriageReturn is also normalized to a LineFeed.
* **Significance**: Whitespace between two adjacent tokens is required only when their concatenation would otherwise form a single valid token.

## Line Terminators

* **Line Length**: A single physical line MUST NOT exceed 4096 characters. Exceeding this limit is a compilation error.
* **Line Continuation**: A backslash (`\`) immediately before a line terminator does NOT continue a line. It is a syntax error outside of a string literal.
