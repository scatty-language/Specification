# Type Aliases

A Type Alias defines a transparent, compile-time synonym for an existing type.

```EBNF
TypeAliasDecl ::= Identifier ["<" TypeParams ">"] "=" Type ";"
```

* **Transparent Equivalence**: A Type Alias DOES NOT create a new, distinct type. The alias and its underlying type are identical and 100% interchangeable during type checking and assignment.
* **Generic Support**: Aliases MAY declare generic type parameters. These parameters MUST be passed to the underlying type.
* **Functions Forbidden**: Aliasing is restricted to Types. Aliasing methods, properties, or variables is FORBIDDEN.
* **No Circularity**: An alias MUST NOT reference itself, either directly or indirectly through another alias.

```C#
// Simplifies a dynamic list of SoA static arrays of standard heap arrays
Chunk<T> = [~][]T[][64];

// Simplifies a heavily constrained interface or refinement
ValidId = Int^Positive;

ProcessChunk<T>(chunk: Chunk<T>, id: ValidId) 
{
    // 'Chunk<T>' is transparently treated as '[~][]T[][64]'
    // 'ValidId' is transparently treated as 'Int^Positive'
}

// ERROR: Cannot alias a method
Print = Console.WriteLine; 
```
