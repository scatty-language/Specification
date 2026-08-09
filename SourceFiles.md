# Source Files

* MUST use the `.cog` file extension for all source code files.
* MUST ignore files without the `.cog` extension during package traversal.
* MUST encode source files in UTF-8.
* MUST NOT contain a Byte Order Mark (BOM).
* MUST reject source files containing a UTF-8 BOM (`U+FEFF`) at the absolute beginning of the file.
* MUST reject source files containing invalid or malformed UTF-8 byte sequences.

## Whitespace

```EBNF
Whitespace     ::= Space | Tab | NewLine
Space          ::= U+0020
Tab            ::= U+0009
NewLine        ::= LineFeed | CarriageReturn | CRLF
LineFeed       ::= U+000A
CarriageReturn ::= U+000D
CRLF           ::= U+000D U+000A
```

* MUST accept both Line Feed (`LF`, `\n`) and Carriage Return + Line Feed (`CRLF`, `\r\n`) as valid line terminators.
* MAY use spaces (`U+0020`), horizontal tabs (`U+0009`), and line terminators to separate tokens.
* MUST NOT assign semantic meaning to whitespace outside of `String` literals.
