# Primitive Data

Primitive data represent the fundamental building blocks of program.

> While they are defined as data within the `System` library to describe their capabilities and contracts, the compiler maps them directly to machine-native types.

## Numeric Data

Numeric data support standard arithmetic, bitwise and comparison operations.

| Type | Size (bits) | Description |
| :--- | :--- | :--- |
| **`Int`** | 32 | Signed integer |
| **`Uint`** | 32 | Unsigned integer |
| **`Long`** | 64 | Signed integer |
| **`Ulong`** | 64 | Unsigned integer |
| **`Size`** | Arch-dependent | Pointer-sized signed integer (32 or 64 bits) |
| **`Usize`** | Arch-dependent | Pointer-sized unsigned integer (32 or 64 bits) |
| **`Byte`** | 8 | Unsigned 8-bit integer |
| **`Float`** | 32 | IEEE 754 single-precision floating point |
| **`Double`** | 64 | IEEE 754 Double-precision floating point |

## Logical and Character Data

| Type | Size (bits) | Description |
| :--- | :--- | :--- |
| **`Bool`** | 1 | Logical value (`true` or `false`) |
| **`Char`** | 8 | Single UTF-8 encoded character |
| **`Rune`** | 32 | Unicode code point |

## Literals

Literals are the lexical representation of primitive values in the source code.

```EBNF
Literal ::= IntLiteral | FloatLiteral | BoolLiteral | CharLiteral | StringLiteral
```

### Integer Literals

Integer literals can be written in decimal, hexadecimal, or binary formats. Underscores (`_`) MAY be used as visual separators anywhere between digits.

```EBNF
IntLiteral ::= (Decimal | Hexadecimal | Binary) [IntSuffix]
Decimal    ::= [0-9] ([0-9_]* [0-9])?
Hexadecimal::= "0x" [0-9A-Fa-f] ([0-9A-Fa-f_]* [0-9A-Fa-f])?
Binary     ::= "0b" [01] ([01_]* [01])?
IntSuffix  ::= "u" | "L" | "uL" | "b"
```

* **Default Type**: An integer literal without a suffix is inferred as `Int` if it fits within 32 signed bits. If it exceeds 32 bits, it is a compilation error unless explicitly suffixed.
* **Suffixes**:
  * `u` = `Uint`
  * `L` = `Long`
  * `uL` = `Ulong`
  * `b` = `Byte`

```C#
a := 1_000_000;    // Int
b := 0xFF_AA;      // Int
c := 0b1010_0011b; // Byte
d := 42L;          // Long
```

### Floating-Point Literals

Floating-point literals MUST contain a decimal point or an exponent.

```EBNF
FloatLiteral ::= Decimal "." Decimal [Exponent] [FloatSuffix]
Exponent     ::= ("e" | "E") ["+" | "-"] Decimal
FloatSuffix  ::= "f" | "d"
```

* **Default Type**: A floating-point literal without a suffix is inferred as `Double` (64-bit).
* **Suffixes**:
  * `f` = `Float`  (32-bit)
  * `d` = `Double` (64-bit)

```C#
pi := 3.14159;     // Double
mass := 5.972e24;  // Double
gravity := 9.81f;  // Float
```

### Logical Literals

```EBNF
BoolLiteral ::= "true" | "false"
```

* Evaluates to the `Bool` type.

### Character and Rune Literals

Character literals are enclosed in single quotes (`' '`) and contain exactly one character or one valid escape sequence.

```EBNF
CharLiteral ::= "'" (SourceChar | EscapeSequence) "'"
```

* **Type Inference**:
  * If the literal evaluates to a valid 1-byte ASCII character (U+0000 to U+007F), its default type is `Char`.
  * If the literal evaluates to a multi-byte Unicode code point (e.g., `'🦀'` or `'\u042F'`), its default type is `Rune`.
* **Explicit Cast**: A 1-byte literal CAN be explicitly typed as a `Rune` at declaration (e.g., `val: Rune = 'A';`).

```C#
ascii := 'A';       // Char
newline := '\n';    // Char
cyrillic := 'Я';    // Rune (U+042F)
emoji := '🔥';      // Rune (U+1F525)
```

### String Literals

String literals are enclosed in Double quotes (`" "`).

```EBNF
StringLiteral ::= '"' (SourceChar | EscapeSequence)* '"'
```

* **Type**: Evaluates to the `String` type.
* **Encoding**: Strings are UTF-8 encoded in memory.
* **Multi-line**: Standard String literals MUST NOT span multiple lines.

```C#
text := "Hello, Coggy!";
```

### Raw String Literals

Raw String literals are enclosed in at least three Double quotes (`"""`).

```EBNF
RawStringLiteral ::= '"""' RawChar* '"""'
RawChar          ::= SourceChar | NewLine
```

* **No Escape Sequences**: Backslashes (`\`) have absolutely no special meaning inside a raw String. Escape sequences MUST be ignored and interpreted as literal text.
* **Embedded Quotes**: Single (`'`) and Double (`"`) quotes can be used freely inside a raw String without escaping.
* **Multi-line and Indentation**: Raw strings MAY span multiple physical lines. If the literal is multi-line, the compiler MUST use the column position of the closing `"""` to determine the base indentation. Any whitespace before this base column on each line is stripped during compilation.
* **Type**: Evaluates to the `String` type.

```C#
regex := """\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b""";

json := """
    {
        "name": "Coggy",
        "version": 1.0,
        "features": ["ARC", "Generics"]
    }
    """;

badString := """
        Text
    """; // ERROR: The closing quotes cannot be less indented than the text content
```
