# Type Transformations

Coggy separates physical memory transformation from abstraction casting.

While the `as` operators change the compiler's interpretation of existing memory, the `to` operators physically read the source data, allocate new memory, and construct an entirely new value of the target type.

```EBNF
TransformExpr      ::= Expression "to" Type
TryTransformExpr   ::= Expression "to?" Type
ForceTransformExpr ::= Expression "to!" Type
```

## Transformation Interfaces

User-defined types MUST explicitly declare their supported transformations by implementing one of two standard library interfaces. A `Data` type MUST NOT implement both interfaces for the exact same target type `T`.

### Infallible Transformation

Used for guaranteed, zero-fail conversions (e.g., expanding a 32-bit integer to a 64-bit integer, or mapping one structural type to another).

```C#
~Transformable<T> 
{
    Transform(@) -> T;
}
```

* **Execution**: Applying `A to B` structurally invokes the `Transform` method.
* **Prohibition**: Using `to?` or `to!` on a type implementing `~Transformable<T>` is FORBIDDEN.

### Fallible Transformation

Used for conversions that may fail at runtime.

```C#
~FallibleTransformable<T> 
{
    TryTransform(@) throws -> T;
}
```

* **`to` (Throwing)**: Invokes `TryTransform`. It inherently acts as a `throws` expression. If the conversion fails, it throws an exception.
* **`to?` (Option)**: Safely attempts the conversion. Evaluates to an Option type (`T?`). If the conversion fails, it catches the exception and yields `None`.
* **`to!` (Forced)**: Asserts success. Evaluates to `T`. If the conversion fails, the runtime MUST immediately trigger the Abort/Fatal Panic mechanism.

## Transformation of Primitive Types

The compiler inherently provides implementations of `~Transformable<T>` and `~FallibleTransformable<T>` for native primitive types, allowing them to use the `to` operators natively.

* **Safe Reinterpretation**: `Int to Float`, `Int to Long` are infallible.
* **Narrowing**: `Long to Int` is fallible and supports `to?` and `to!`.

```C#
Vector : ~Transformable<Point> 
{
    X: Int;
    Y: Int;

    Transform(@) -> Point 
    {
        return Point { X: @.X, Y: @.Y };
    }
}

vec := Vector { X: 10, Y: 20 };

// Allocates a new 'Point' instance
point := vec to Point; 

// ERROR: Infallible conversions do not support 'to?'
bad := vec to? Point; 

Record : ~FallibleTransformable<User> 
{
    RawId: String;

    TryTransform(@) throws -> User 
    {
        if @.RawId == "" => throw Exception.New("Empty ID");
        
        // Assumes 'String' implements '~FallibleTransformable<Int>'
        parsedId := try! @.RawId to Int; 
        
        return User { Id: parsedId };
    }
}

record := Record { RawId: "123" };

// Fallible 'to' requires a throws context or try construct
ProcessRecord() throws 
{
    user := record to User; 
}

// Safe inline unpacking using 'to?'
if user := record to? User 
{
    Console.WriteLine(user.Id);
}

// Forced transformation
validUser := record to! User; // ABORTS immediately if 'RawId' is invalid
```
