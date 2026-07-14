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
