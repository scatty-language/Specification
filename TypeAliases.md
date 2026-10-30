# Type Aliases

* MUST declare a type alias at the package or file top level.
* MUST evaluate a type alias as a transparent compile-time synonym for its underlying target type.
* MUST NOT create a distinct nominal type when declaring a type alias.
* MUST treat a type alias and its underlying target type as completely interchangeable during type checking, assignment, and dispatch.
* MUST NOT define aliases for methods, properties, functions, or variables.
* MUST NOT contain direct or indirect circular references within type alias declarations.

## Generic Type Aliases

* MAY declare generic type parameters on a type alias.
* MUST map and forward all declared generic type parameters to the underlying aliased type.

```C#
Chunk<T> = [[[T]; 64] soa];

ValidId = Int^Positive;

ProcessChunk<T>(chunk: Chunk<T>, id: ValidId) { }
```
