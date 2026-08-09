# Strings

* MUST guarantee valid UTF-8 encoding for all `String` instances.
* MUST manage heap-allocated buffers via RAII (Owning Type).
* MUST move ownership upon assignment.
* MUST explicitly invoke `.Copy()` to duplicate the underlying memory buffer.

## `String`

* MUST back the `String` type structurally with an array of 8-bit characters (`Char[]`).
* MUST return a single 8-bit `Char` upon indexing.
* MUST yield 8-bit `Char` elements during standard iteration.
* MUST enforce valid UTF-8 encoding during mutation operations.
* MUST panic or throw an exception if a malformed byte sequence is injected.

## `RuneString`

* MUST back the `RuneString` type structurally with an array of 32-bit Unicode scalars (`Rune[]`).
* MUST return a single 32-bit `Rune` upon indexing.
* MAY explicitly transform between `String` and `RuneString` using the `to` operator.
* MUST perform memory reallocation and UTF-8 decoding/encoding during `to` transformations.

```C#
text: String = "Hello 🌍"; 

byteChar: Char = text[6]; 

runeText := text to RuneString;

scalar: Rune = runeText[6]; 

backToString := runeText to String;
```

## Equality and Comparisons

* MUST evaluate structural equality operators via a byte-for-byte comparison of the underlying buffer.
* MUST support lexical sorting based on UTF-8 byte order via implementation of the `~Comparable` interface.

```C#
strA := "Coggy";
strB := "Coggy";

// Evaluates to true
if strA == strB 
{
    Console.WriteLine("Exact match");
}
```

## Iteration

* MUST iterate over raw 8-bit bytes (`Char`) when utilizing standard loops on a `String`.
* MAY iterate dynamically over decoded Unicode scalars (`Rune`) without intermediate heap allocation via the `.Runes()` method.

```C#
text := "A🌍C";

for c in text 
{
    ProcessByte(c);
}

for r in text.Runes() 
{
    ProcessRune(r);
}
```
