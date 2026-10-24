# Escape Sequences

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

* MUST represent a valid Unicode scalar value for all `\u` and `\U` escapes.
* MUST NOT exceed a maximum value of `U+10FFFF` in numeric escapes.
* MUST NOT use surrogate code points in the range `U+D800` through `U+DFFF`.
* MUST reject any `\xHH` sequence that results in a malformed UTF-8 string literal.
* MAY use either uppercase or lowercase hex digits (`A-F` or `a-f`).

## String Interpolation Delimiter

* MUST initiate an interpolated expression segment upon encountering the `\(` sequence within a standard double-quoted `String` literal (`"..."`).
* MUST parse a full expression with balanced parentheses inside `\(` and `)`, including nested function invocations and nested string literals.
* MUST reject an empty interpolation sequence (`\()`) at compile time.
* MUST use `\\(` to represent the literal character sequence `\(` within a standard `String` literal.
* MUST NOT evaluate `\(` as an interpolation delimiter within `Char` literals (`'...'`) or raw string literals (`"""..."""`).

## Invalid Escapes

* MUST reject any backslash sequence containing characters not specified in the simple or numeric escape tables.
* MUST reject numeric escapes containing fewer than the designated count of hexadecimal digits.
