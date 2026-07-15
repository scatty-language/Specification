# Escape Sequences

Escape sequences allow the representation of non-printable and special characters inside string and character literals. An escape sequence strictly begins with a backslash (`\`) and is interpreted by the lexer.

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
| **`\\`** | U+005C | Backslash |
| **`\"`** | U+0022 | Double quotation mark |
| **`\'`** | U+0027 | Single quotation mark |
| **`\0`** | U+0000 | Null character |

## Numeric Escapes

| Sequence | Format | Description |
| :--- | :--- | :--- |
| **`\xHH`** | 2 hex digits | Single byte value (U+0000–U+00FF) |
| **`\uHHHH`** | 4 hex digits | Unicode code point (U+0000–U+FFFF) |
| **`\UHHHHHHHH`** | 8 hex digits | Unicode code point (U+0000–U+10FFFF) |

* **Unicode Validation**: A `\u` or `\U` escape MUST represent a valid Unicode scalar value. The maximum valid value is `U+10FFFF`. Surrogate code points (`U+D800` to `U+DFFF`) are strictly FORBIDDEN and result in a compilation error.
* **UTF-8 Strictness**: String literals in Coggy MUST remain valid UTF-8 sequences. If `\xHH` is used to inject raw bytes that result in a malformed UTF-8 string literal, the compiler MUST reject it.
* **Case Insensitivity**: Hex digits (`A-F`) in numeric escapes are case-insensitive.

## Invalid Escapes

* **Unknown Sequence**: A backslash followed by a character not listed above is a strict compilation error.
* **Truncated Escape**: A numeric escape with fewer than the required hex digits is a compilation error.

```C#
s := "Line1\nLine2";
c := '\t';
u := "\u0041";

bad := "\q";                // ERROR: unknown escape sequence
trunc := "\u41";            // ERROR: '\u' requires exactly 4 hex digits
outOfRange := "\U00110000"; // ERROR: exceeds maximum Unicode scalar U+10FFFF
surrogate := "\uD800";      // ERROR: surrogate code point is invalid
badUtf8 := "\xFF";          // ERROR: '\xFF' alone is not a valid UTF-8 sequence
```
