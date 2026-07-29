# Strings

Coggy represents text primarily through the `String` standard library `Data` type.

* **UTF-8 Encoding**: All `String` instances are guaranteed to contain valid UTF-8 encoded sequences.
* **Value Semantics**: `String` is managed by Automated Reference Counting (ARC) with Copy-on-Write (CoW). Assigning a string creates a logical copy without duplicating the underlying buffer until a mutation occurs.

## Memory Representation

Coggy separates byte-oriented text from scalar-oriented text.

### `String`

The standard `String` type is structurally backed by an array of 8-bit characters (`Char[]`).

* **Indexing**: Indexing into a `String` (e.g., `text[i]`) is a O(1) operation that returns a single 8-bit `Char`. It DOES NOT return a Unicode code point.
* **Iteration**: Standard iteration (`for c in text`) yields 8-bit `Char` elements.
* **Validation**: Operations that mutate a `String` MUST maintain valid UTF-8 encoding. Attempting to inject malformed byte sequences results in an exception or panic.

### `RuneString`

For complex text manipulation requiring direct access to Unicode code points, the standard library provides the `RuneString` type.

* **Memory**: It is structurally backed by an array of 32-bit Unicode scalars (`Rune[]`).
* **Indexing**: Indexing into a `RuneString` (e.g., `text[i]`) is an O(1) operation that returns a 32-bit `Rune`.
* **Transformation**: A standard `String` CAN be explicitly transformed into a `RuneString` (and vice versa) using the `to` operator. This performs a physical memory reallocation and decoding/encoding pass.

```C#
// 'text' is internally a 'Char[]'
text: String = "Hello 🌍"; 

// Evaluates to the first byte of the Earth emoji, not the emoji itself
byteChar: Char = text[6]; 

// Transformation decoding
runeText := text to RuneString;

//Evaluates to the actual 🌍 (U+1F30D) code point
scalar: Rune = runeText[6]; 

// Transformation encoding back to UTF-8
backToString := runeText to String;
```

## Immutability and Mutation

Because `String` adheres to standard Value Semantics, variables declared with `:=` or `=` are fully mutable.

* **In-Place Mutation**: Methods that modify the string (e.g., `.Append()`, `.Replace()`) mutate the local buffer in-place if the ARC count is `1`.
* **Copy-on-Write**: If the string buffer is shared across multiple variables, mutating one forces a physical copy of the buffer before the mutation is applied.

```C#
greeting := "Hello";
shared := greeting; // ARC increment, no memory duplication

// Mutates 'shared' triggers a buffer duplication
shared.Append(" World"); 

Console.WriteLine(greeting); // Output: "Hello"
Console.WriteLine(shared);   // Output: "Hello World"
```

## Equality and Comparisons

* **Structural Equality (`==`, `!=`)**: The compiler-generated equality operators for `String` intrinsically perform a byte-for-byte comparison.
* **Relational Operators**: `String` explicitly implements the `~Comparable` interface, allowing lexical sorting based on UTF-8 byte order.

```C#
strA := "Coggy";
strB := "Coggy";

// Evaluates to true
// Directly compares the underlying 'Char[]' buffers
if strA == strB 
{
    Console.WriteLine("Exact match");
}
```

## Iteration

* **Byte Iteration**: Using standard loops on a `String` iterates over raw bytes.
* **Code Point Iteration**: To iterate over decoded Unicode scalars without physically allocating a `RuneString`, the `String` type provides an iterator explicitly via the `.Runes()` method.

```C#
text := "A🦀C";

// Iterates over 1 + 4 + 1 = 6 bytes
for c in text 
{
    ProcessByte(c);
}

// Iterates over 3 Unicode scalars dynamically without extra heap allocation
for r in text.Runes() 
{
    ProcessRune(r);
}
```
