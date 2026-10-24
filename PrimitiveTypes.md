# Primitive Data

* MUST map primitive data types directly to machine-native types during compilation.

## Numeric Data

* MUST size numeric primitive types according to the following definitions:

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

* MUST size logical and character primitive types according to the following definitions:

| Type | Size (bits) | Description |
| :--- | :--- | :--- |
| **`Bool`** | 1 | Logical value (`true` or `false`) |
| **`Char`** | 8 | Single UTF-8 encoded character |
| **`Rune`** | 32 | Unicode code point |

## Literals

### Integer Literals

* MAY use underscores (`_`) anywhere between digits as visual separators.
* MUST infer an unsuffixed integer literal as `Int` if its value fits within 32 signed bits.
* MUST NOT permit an unsuffixed integer literal that exceeds 32 signed bits.
* MUST explicitly type suffixed integer literals as follows: `u` for `Uint`, `L` for `Long`, `uL` for `Ulong`, and `b` for `Byte`.

```C#
a := 1_000_000;    // Int
b := 0xFF_AA;      // Int
c := 0b1010_0011b; // Byte
d := 42L;          // Long
```

### Floating-Point Literals

* MUST evaluate an unsuffixed floating-point literal as `Double` (64-bit).
* MUST explicitly type suffixed floating-point literals as follows: `f` for `Float` (32-bit), `d` for `Double` (64-bit).

```C#
pi := 3.14159;     // Double
mass := 5.972e24;  // Double
gravity := 9.81f;  // Float
```

### Logical Literals

* MUST evaluate logical literals as the `Bool` type.

### Character and Rune Literals

* MUST contain exactly one valid character or one valid escape sequence.
* MUST infer a literal evaluating to a 1-byte ASCII character (`U+0000` to `U+007F`) as `Char`.
* MUST infer a literal evaluating to a multi-byte Unicode code point as `Rune`.
* MAY explicitly cast a 1-byte ASCII literal to a `Rune` type.

```C#
ascii := 'A';       // Char
newline := '\n';    // Char
cyrillic := 'Я';    // Rune (U+042F)
emoji := '🔥';      // Rune (U+1F525)
```

### String Literals

* MUST evaluate to the `String` type.
* MUST encode string literal memory as UTF-8.
* MUST NOT span multiple physical lines.
* MAY embed interpolated expressions within a string literal using the `\(Expression)` syntax.
* MUST fold an interpolated string literal into a single constant `String` literal at compile time if all embedded expressions evaluate to compile-time constants.

```C#
text := "Hello, Scatty!";

MaxUsers :: 100;

// Evaluated at compile time into "Max capacity: 100"
Banner :: "Max capacity: \(MaxUsers)";
```

### Raw String Literals

* MUST ignore escape sequences and interpret backslashes (`\`) as literal text.
* MAY contain unescaped single (`'`) and double (`"`) quotes.
* MAY span multiple physical lines.
* MUST determine the base indentation column from the position of the closing `"""` sequence in multi-line raw strings.
* MUST strip leading whitespace up to the base indentation column from each line during compilation.
* MUST NOT permit content lines to contain less leading whitespace than the column position of the closing `"""`.
* MUST evaluate to the `String` type.

```C#
regex := """\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b""";

json := """
    {
        "name": "Scatty",
        "version": 1.0,
        "features": ["Exceptions", "Generics"]
    }
    """;

badString := """
        Text
    """; // ERROR: The closing quotes cannot be less indented than the text content
```
