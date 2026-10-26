# Strings

* MUST guarantee valid UTF-8 encoding for all `String` instances.
* MUST manage heap-allocated string buffers via ARC and Copy-on-Write as Owning Types.
* MUST retain the underlying buffer upon assignment or pass-by-value without duplicating memory.
* MUST automatically clone the underlying buffer via Copy-on-Write prior to any mutating operation if the buffer's strong reference count exceeds `1`.
* MAY explicitly invoke `.Copy()` to force an eager physical duplication of the underlying memory buffer.

## `String`

* MUST back the `String` type structurally with an array of 8-bit characters (`Char[]`).
* MUST return a single 8-bit `Char` upon indexing.
* MUST yield 8-bit `Char` elements during standard iteration.
* MUST enforce valid UTF-8 encoding during mutation operations.
* MUST panic or throw an exception if a malformed byte sequence is injected.

## `RuneString`

* MUST back the `RuneString` type structurally with an array of 32-bit Unicode scalars (`Rune[]`).
* MUST return a single 32-bit `Rune` upon indexing.
* MAY explicitly transform between `String` and `RuneString` using the `as` operator.
* MUST perform memory reallocation and UTF-8 decoding/encoding during `as` transformations between `String` and `RuneString`.

```C#
text: String = "Hello 🌍"; 

byteChar: Char = text[6]; 

runeText := text as RuneString;

scalar: Rune = runeText[6]; 

backToString := runeText as String;
```

## Equality and Comparisons

* MUST evaluate structural equality operators via a byte-for-byte comparison of the underlying buffer.
* MUST support lexical sorting based on UTF-8 byte order via implementation of the `~Comparable` interface.

```C#
strA := "Scatty";
strB := "Scatty";

// Evaluates to true
where strA == strB 
{
    Console.WriteLine("Exact match");
}
```

## Iteration

* MUST iterate over raw 8-bit bytes (`Char`) when utilizing standard loops on a `String`.
* MAY iterate dynamically over decoded Unicode scalars (`Rune`) without intermediate heap allocation via the `.Runes()` method.

```C#
text := "A🌍C";

c in text 
{
    ProcessByte(c);
}

r in text.Runes() 
{
    ProcessRune(r);
}
```

## String Interpolation

* **MAY** embed arbitrary expressions inside a standard double-quoted string literal using the `\(Expression)` syntax.
* **MUST** permit the full expression grammar within `\(...)`, including arithmetic operations, method invocations, and nested string literals.
* **MUST NOT** evaluate `\(...)` interpolation within raw string literals (`"""..."""`).
* **MUST** evaluate an interpolated string literal at compile time if every embedded expression is a compile-time constant (`::` or literal), producing a static constant `String` without runtime allocation.

```C#
Version :: 2;
// Compile-time constant folding
AppTitle :: "Scatty App v\(Version)";

name := "Alice";
score := 42;

// Runtime interpolation with nested expressions and string literals
greeting := "Player \(name.Trim(" ")) scored \(score + 10) points!";
```

### Formatting

* MUST require any expression interpolated via `\(...)` at runtime to satisfy the standard library `~Formattable` interface.
* MUST provide intrinsic implementations of `~Formattable` for all primitive types, tuples, and collections.
* MUST automatically synthesize a structural `~Formattable` implementation at compile time for any `Data` type that does not explicitly implement `~Formattable`, formatting its public fields or active variant name and payload.
* MAY explicitly implement `~Formattable` on a `Data` type to override the default structural formatting.
* MUST format `T weak` fields non-recursively without traversing the target instance's fields during compiler-synthesized `~Formattable` execution.
* MUST NOT automatically synthesize a structural `~Formattable` implementation for an untagged union (`::: none`); an untagged union MUST explicitly implement `~Formattable` to participate in runtime string interpolation.

```C#
~Formattable 
{
    Format(@, ref builder: StringBuilder);

    ToString(@) -> String 
    {
        sb := StringBuilder.New();
        @.Format(ref sb);

        return sb.Build();
    }
}
```

* MUST desugar runtime string interpolation into a single `StringBuilder` sequence:
  1. Pre-calculate the initial byte capacity from the static string segments and estimated formatted lengths of the interpolated expressions.
  2. Append static literal segments directly to the `StringBuilder` buffer.
  3. Invoke `.Format(ref builder)` on each interpolated expression, writing UTF-8 bytes directly into the `StringBuilder` buffer without allocating intermediate heap `String` instances.
  4. Transfer ownership of the `StringBuilder` buffer directly to the resulting `String` via `.Build()` without copying the buffer.

```C#
Vector(X: Int, Y: Int);

vec := Vector { X: 10, Y: 20 };

// Uses compiler-synthesized structural '~Formattable' for 'Vector'
// Desugars into a single StringBuilder allocation:
//   _sb := StringBuilder.New(24);
//   _sb.Append("Pos: ");
//   vec.Format(ref _sb);
//   log := _sb.Build();
log := "Pos: \(vec)";
```
