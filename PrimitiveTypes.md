# Primitive Data

Primitive data represent the fundamental building blocks of program.

> While they are defined as data within the `System` library to describe their capabilities and contracts, the compiler maps them directly to machine-native types.

## Numeric Data

Numeric data support standard arithmetic, bitwise and comparison operations.

| Type | Size (bits) | Description |
| :--- | :--- | :--- |
| **`int`** | 32 | Signed integer |
| **`uint`** | 32 | Unsigned integer |
| **`long`** | 64 | Signed integer |
| **`ulong`** | 64 | Unsigned integer |
| **`byte`** | 8 | Unsigned 8-bit integer |
| **`float`** | 32 | IEEE 754 single-precision floating point |
| **`double`** | 64 | IEEE 754 double-precision floating point |

## Logical and Character Data

| Type | Size (bits) | Description |
| :--- | :--- | :--- |
| **`bool`** | 1 | Logical value (`true` or `false`) |
| **`char`** | 8 | Single UTF-8 encoded character |
| **`rune`** | 32 | Unicode code point |

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

* **Default Type**: An integer literal without a suffix is strictly inferred as `int` if it fits within 32 signed bits. If it exceeds 32 bits, it is a compilation error unless explicitly suffixed.
* **Suffixes**:
  * `u` = `uint`
  * `L` = `long`
  * `uL` = `ulong`
  * `b` = `byte`

```C#
a := 1_000_000;    // int
b := 0xFF_AA;      // int
c := 0b1010_0011b; // byte
d := 42L;          // long
```

### Floating-Point Literals

Floating-point literals MUST contain a decimal point or an exponent.

```EBNF
FloatLiteral ::= Decimal "." Decimal [Exponent] [FloatSuffix]
Exponent     ::= ("e" | "E") ["+" | "-"] Decimal
FloatSuffix  ::= "f" | "d"
```

* **Default Type**: A floating-point literal without a suffix is strictly inferred as `double` (64-bit).
* **Suffixes**:
  * `f` = `float`  (32-bit)
  * `d` = `double` (64-bit)

```C#
pi := 3.14159;     // double
mass := 5.972e24;  // double
gravity := 9.81f;  // float
```

### Logical Literals

```EBNF
BoolLiteral ::= "true" | "false"
```

* Evaluates strictly to the `bool` type.

### Character and Rune Literals

Character literals are enclosed in single quotes (`' '`) and contain exactly one character or one valid escape sequence.

```EBNF
CharLiteral ::= "'" (SourceChar | EscapeSequence) "'"
```

* **Type Inference**:
  * If the literal evaluates to a valid 1-byte ASCII character (U+0000 to U+007F), its default type is `char`.
  * If the literal evaluates to a multi-byte Unicode code point (e.g., `'🦀'` or `'\u042F'`), its default type is strictly `rune`.
* **Explicit Cast**: A 1-byte literal CAN be explicitly typed as a `rune` at declaration (e.g., `val: rune = 'A';`).

```C#
ascii := 'A';       // char
newline := '\n';    // char
cyrillic := 'Я';    // rune (U+042F)
emoji := '🔥';      // rune (U+1F525)
```

### String Literals

String literals are enclosed in double quotes (`" "`).

```EBNF
StringLiteral ::= '"' (SourceChar | EscapeSequence)* '"'
```

* **Type**: Evaluates to the `string` type.
* **Encoding**: Strings are strictly UTF-8 encoded in memory.
* **Multi-line**: Standard string literals MUST NOT span multiple lines.

```C#
text := "Hello, Coggy!";
```
