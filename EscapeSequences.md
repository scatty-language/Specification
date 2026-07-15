# Escape Sequences

Escape sequences allow the representation of non-printable and special characters inside string and character literals. An escape sequence begins with a backslash (``) and is interpreted by the lexer.

```EBNF
EscapeSequence ::= "\" (SimpleEscape | NumericEscape)
SimpleEscape   ::= "n" | "r" | "t" | "\" | "\"" | "'" | "0"
NumericEscape  ::= "u" HexDigit4 | "U" HexDigit8 | "x" HexDigit2
HexDigit4      ::= HexDigit HexDigit HexDigit HexDigit
HexDigit8      ::= HexDigit HexDigit HexDigit HexDigit HexDigit HexDigit HexDigit HexDigit
HexDigit2      ::= HexDigit HexDigit
HexDigit       ::= [0-9A-Fa-f]
```

## Simple Escapes

| Sequence | Value | Description |
| :--- | :--- | :--- |
| **`\n`** | U+000A | Line feed |
| **`\r`** | U+000D | Carriage return |
| **`\t`** | U+0009 | Horizontal tab |
| **`\`** | U+005C | Backslash |
| **`\"`** | U+0022 | Double quotation mark |
| **`\'`** | U+0027 | Single quotation mark |
| **`\0`** | U+0000 | Null character |

## Numeric Escapes

| Sequence | Format | Description |
| :--- | :--- | :--- |
| **`\xHH`** | 2 hex digits | Single byte value (U+0000–U+00FF) |
| **`\uHHHH`** | 4 hex digits | Unicode code point (U+0000–U+FFFF) |
| **`\UHHHHHHHH`** | 8 hex digits | Unicode code point (U+0000–U+10FFFF) |

* **Validation**: A `\U` escape MUST represent a valid Unicode scalar value (U+0000–U+10FFFF, excluding surrogates U+D800–U+DFFF). An out-of-range or surrogate value is a compilation error.
* **Case Insensitivity**: Hex digits in numeric escapes are case-insensitive.

## Invalid Escapes

* **Unknown Sequence**: A backslash followed by a character not listed above is a compilation error.
* **Truncated Escape**: A numeric escape with fewer than the required hex digits is a compilation error.

```C#
s := "Line1\nLine2";        // Valid: line feed
c := '\t';                  // Valid: tab
u := "\u0041";              // Valid: "A"
bad := "\q";                // Error: unknown escape sequence
trunc := "\u41";            // Error: \u requires exactly 4 hex digits
surrogate := "\UD800DC00";  // Error: surrogate code point
```
