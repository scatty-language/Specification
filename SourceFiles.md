# Source Files

* **File Extension**: All Coggy source code files MUST use the `.cog` file extension. The compiler MUST ignore files with any other extension during standard package traversal.
* **Encoding**: Source files MUST be strictly encoded in UTF-8.
* **Byte Order Mark (BOM)**: Source files MUST NOT contain a Byte Order Mark (BOM). If a UTF-8 BOM (`U+FEFF`) is present at the absolute beginning of the file, the compiler MUST strictly reject the file with a compilation error.
* **Malformed Sequences**: The compiler MUST reject any file containing invalid or malformed UTF-8 byte sequences.

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

* **Line Endings**: The compiler MUST seamlessly accept both Line Feed (`LF`, `\n`) and Carriage Return + Line Feed (`CRLF`, `\r\n`) as valid line termination sequences.
* **Whitespace**: Spaces (`U+0020`), horizontal tabs (`U+0009`), and line terminators are treated as whitespace. Whitespace is strictly used to separate tokens and has no semantic meaning outside of string literals.
